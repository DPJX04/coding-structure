# Layout: Python

Python has real packaging rules, and ignoring them produces a tree that imports inconsistently depending on where you run it from. Get the packaging right first, then apply the Laws inside it.

---

## Folder Layout

Use the src layout. The source root is `src/`, but your code lives in a **named package** inside it, not directly in `src/`.

```
project_root/
├── pyproject.toml
├── README.md
├── src/
│   └── myapp/                      # the package, this name is what you import
│       ├── __init__.py             # thin, exports the app's public surface
│       ├── __main__.py             # entry: python -m myapp, checks settings at startup
│       ├── features/
│       │   ├── __init__.py
│       │   └── booking/
│       │       ├── __init__.py     # the public door, re-exports only
│       │       ├── orchestrator.py # entry point for the feature
│       │       └── _rules.py       # private, leading underscore: the feature's checks and formatting
│       ├── services/
│       │   ├── __init__.py
│       │   ├── db.py               # get_pool(): builds the pool on first call, then reuses it
│       │   ├── db_service.py
│       │   └── email_service.py
│       ├── utils/
│       │   ├── __init__.py
│       │   ├── error_handler.py
│       │   └── date_utils.py
│       ├── domain/                 # shared dataclasses, TypedDicts, or pydantic models, and Result
│       │   ├── __init__.py
│       │   ├── booking.py
│       │   └── result.py
│       └── config/
│           ├── __init__.py
│           └── settings.py         # the only file that reads raw env
├── tests/                          # mirrors src/myapp/
│   ├── __init__.py
│   ├── services/test_db_service.py
│   └── utils/test_date_utils.py
└── scripts/                        # seeds, migrations, one off tooling
```

### Why a named package, not bare `src/`
`import src.features.booking` is broken. It only works when the interpreter happens to start in the project root, it collides with any other project that also uses `src`, and it cannot be installed as a package. With `src/myapp/`, you install the project in editable mode and `from myapp.services import db_service` works from anywhere, including tests and scripts.

```toml
# pyproject.toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "myapp"
version = "0.1.0"
requires-python = ">=3.11"

[tool.hatch.build.targets.wheel]
packages = ["src/myapp"]
```

Then `pip install -e .` once, and imports resolve consistently everywhere.

### Why `domain/`, not `types/`
The type layer is named `domain/` in Python. A package folder called `types` shadows the standard library's `types` module the moment that folder lands on `sys.path`, such as when a file inside the package is run directly or a mirrored `tests/types/` is discovered, and `dataclasses` itself imports `types`, so everything breaks at once. The same goes for any folder named after a standard library module: `logging`, `email`, `json`, `secrets`.

### Without pip
When the project must stay standard library only, skip the build backend and the editable install. Run the app with `python -m myapp` from `src/`, or with `PYTHONPATH=src`, and run tests with `PYTHONPATH=src python -m unittest discover -s tests -t .`.

---

## The Dependency Law, Mapped

Law 1 translated into this layout's folder names. The diagram in `SKILL.md` stays the authoritative copy.

```
features/  can use →  services, utils, domain
services/  can use →  utils, domain, config
utils/     can use →  domain only
domain/    can use →  nothing
```

There is no UI logic layer in a backend, so the feature's entry point composes its private helpers and its service calls directly, per Law 4: it may call them, it defines none of them.

### Why there is no store layer here

A server keeps no state between requests. There is usually no state that outlives a request, so the store layer from the Laws simply does not exist in a Python backend and you should not create a folder for it.

Two things get mistaken for one:

- **Request scoped state.** A user, a tenant, a correlation id. Pass it as an argument, or use a framework dependency such as FastAPI's `Depends`. Never a module level global, which is shared across every request in the process and is a data leak between users waiting to happen.
- **Process wide caches and connection pools.** These belong in `services/`, built once on first use by a cached function such as `db.get_pool()`, because the service layer owns every external client (Law 7) and `config/` imports nothing. They are infrastructure, not application state.

If a long running worker or a desktop app genuinely does hold state across time, apply the store rules from Law 1 and put it in its own module above `services/`.

---

## `__init__.py` Is a Door, Not an Orchestrator

Put no logic in `__init__.py`. It re-exports and declares the public surface, nothing else. Two reasons:

- Every import of the package runs it, so logic there slows startup and makes import side effects hard to trace.
- Logic in `__init__.py` is invisible to a reader scanning file names for behavior.

```python
# src/myapp/features/booking/__init__.py
"""Public surface of the booking feature."""

from .orchestrator import create_booking, cancel_booking

__all__ = ["create_booking", "cancel_booking"]
```

```python
# src/myapp/features/booking/orchestrator.py
from myapp.domain.booking import Confirmation
from myapp.domain.result import Err, Ok, Result
from myapp.services import db_service
from . import _rules

def create_booking(user_id: str, slot_id: str) -> Result[Confirmation]:
    """Validate and save one booking, returning its confirmation or an error."""
    error = _rules.validate_booking(user_id, slot_id)
    if error:
        return Err(error)

    result = db_service.insert_booking(user_id, slot_id)
    if isinstance(result, Err):
        return result

    return Ok(_rules.to_confirmation(result.data))
```

Import the module and call `module.function`: `db_service.insert_booking` shows the reader where the function lives, and the import block stays one line per module. Types are imported by name, which PEP 8 allows. Group imports as PEP 8 does: standard library, then third party, then this project, with a blank line between groups.

`_rules.py` holds every private function of this feature, per Law 3. It splits when a second topic has grown several functions of its own, each new file named for its topic.

### Sealing
Python cannot enforce privacy, so it relies on two conventions and one tool:

1. Leading underscore on private modules and functions. `_rules.py` signals internal.
2. `__all__` in the door, which declares the public surface.
3. `import-linter`, which fails the build on an illegal import. See below.

---

## Naming

| Type | Convention | Example |
|---|---|---|
| Module | snake_case | `slot_validator.py` |
| Private module | leading underscore | `_rules.py` |
| Package or folder | snake_case | `booking_slots/` |
| Class | PascalCase | `BookingRequest` |
| Function and variable | snake_case | `create_booking` |
| Constant | UPPER_SNAKE | `MAX_SLOTS` |
| Test file | `test_` prefix | `test_db_service.py` |
| Function | snake_case, verb first | `fetch_slots`, `create_booking` |
| Function comment | one line docstring in every top-level function, method, and class | `"""Fetch the open slots for one date."""` |

---

## The Result Shape and the Service Pattern

Law 12: one error shape, declared once in `domain/`, imported by every service. Python's sum type is a union of two frozen dataclasses, narrowed with `isinstance` or `match`.

```python
# src/myapp/domain/result.py
from dataclasses import dataclass
from typing import Generic, TypeAlias, TypeVar

T = TypeVar("T")

@dataclass(frozen=True)
class Ok(Generic[T]):
    data: T

@dataclass(frozen=True)
class Err:
    error: str

Result: TypeAlias = Ok[T] | Err
```

Services are named functions, each handling its own failure and returning a `Result`. Type hints are not optional in a large codebase. Every outside call carries a timeout: `timeout=` on an HTTP call, and a connect and statement timeout where the pool is built (Law 7). `parse_error` logs the raw error with `logging` before it returns the message (Law 12).

```python
# src/myapp/services/db_service.py
from datetime import date as iso_date

from myapp.domain.booking import Slot
from myapp.domain.result import Err, Ok, Result
from myapp.services import db
from myapp.utils import error_handler

def _to_slot(row: dict) -> Slot:
    """Check one raw row and return it as a Slot, or raise if the id is missing."""
    if not isinstance(row.get("id"), str):
        raise ValueError("slot row is missing an id")
    return Slot(**row)

def fetch_slots(date: str) -> Result[list[Slot]]:
    """Fetch every slot on one date. Returns Err instead of raising."""
    try:
        iso_date.fromisoformat(date)
        with db.get_pool().connection() as conn:
            rows = conn.execute(
                "SELECT * FROM slots WHERE date = %s", (date,)
            ).fetchall()
        return Ok([_to_slot(row) for row in rows])
    except Exception as err:
        return Err(error_handler.parse_error(err))
```

`fromisoformat` and `_to_slot` are the Law 10 gate: input is rejected before it reaches SQL, and rows are checked before they become models. In a real project a pydantic model does `_to_slot` better; the point is that something checks. Law 8 in this stack: identity comes from the framework's verified session, `request.user` in Django or a FastAPI `Depends` that resolves the session, never from a field in the request body. An endpoint with no login gets a rate limiter such as `django-ratelimit` or `slowapi`.

---

## Config Pattern

One typed settings module, read through a cached `get_settings()`. Nothing reads the environment at import, so any module imports cleanly in a test. The entry file calls `get_settings()` once at startup, so a missing value fails at boot rather than inside a request.

```python
# src/myapp/config/settings.py
from functools import cache

from pydantic import SecretStr, ValidationError
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    database_url: str
    is_prod: bool = False
    mail_api_key: SecretStr | None = None  # only email_service needs it

@cache
def get_settings() -> Settings:
    """Read the env once. Raises RuntimeError naming each missing or bad value."""
    try:
        return Settings()
    except ValidationError as err:
        names = ", ".join(str(e["loc"][0]).upper() for e in err.errors())
        raise RuntimeError(f"Missing or invalid env var: {names}") from None

def get_mail_api_key() -> str:
    """Return the mail key, or raise if it is unset. Only email_service calls this."""
    key = get_settings().mail_api_key
    if key is None:
        raise RuntimeError("Missing required env var: MAIL_API_KEY")
    return key.get_secret_value()
```

`mail_api_key` is Law 11's second half: a value only one service needs is optional, so the app runs without it, and that service checks it through `get_mail_api_key()`. `SecretStr` keeps the key out of logs and tracebacks.

Standard library only, the same module is a frozen dataclass built by the same cached function:

```python
# src/myapp/config/settings.py
import os
from dataclasses import dataclass
from functools import cache

@dataclass(frozen=True)
class Settings:
    """Every environment value the app reads."""
    database_url: str
    is_prod: bool

def _required(name: str) -> str:
    """Return the env value, or raise if it is missing."""
    value = os.environ.get(name)
    if not value:
        raise RuntimeError(f"Missing required env var: {name}")
    return value

@cache
def get_settings() -> Settings:
    """Read the env once. Later calls return the same object."""
    return Settings(
        database_url=_required("DATABASE_URL"),
        is_prod=os.environ.get("APP_ENV") == "production",
    )
```

The entry file turns a missing value into one clean line and a non-zero exit:

```python
# src/myapp/__main__.py
import sys

from myapp.config import settings

try:
    settings.get_settings()
except RuntimeError as err:
    sys.exit(str(err))  # one line on stderr, exit code 1
# start the server or command here
```

No other module reads `os.environ`. `services/db.py` exposes `get_pool()`, cached the same way, which builds the pool from `get_settings().database_url` on its first call. Services call it when they run, never at import.

---

## Types

Shared shapes and `Result` live in `domain/` and depend on nothing. Use dataclasses, `TypedDict`, or pydantic models. Type hint every public function, and treat a bare `dict` passed between layers as the escape hatch it is. Run `mypy` or `pyright` in strict mode on `services/`, `utils/`, and `domain/` at minimum, since those are the layers everything else leans on.

---

## Enforce the Dependency Law Automatically

Use `import-linter`. It reads contracts and fails CI on a violation.

```ini
# .importlinter
[importlinter]
root_package = myapp

[importlinter:contract:layers]
name = Layered architecture
type = layers
layers =
    myapp.features
    myapp.services
    myapp.utils
    myapp.domain

[importlinter:contract:features]
name = Features are independent
type = independence
modules =
    myapp.features.*

[importlinter:contract:config]
name = Only services read config
type = forbidden
source_modules =
    myapp.features
    myapp.utils
    myapp.domain
forbidden_modules =
    myapp.config
allow_indirect_imports = True

[importlinter:contract:config-leaf]
name = Config imports nothing
type = forbidden
source_modules =
    myapp.config
forbidden_modules =
    myapp.features
    myapp.services
    myapp.utils
    myapp.domain
```

The `layers` contract enforces the one way flow. The `independence` contract enforces that features never touch each other, and `myapp.features.*` picks up each new feature by itself. The two `forbidden` contracts pin `config` down: only `services` reads it, and it reads nothing back. `allow_indirect_imports` is required on the first, because by default a forbidden contract also fails on the chain `features → services → config`, which is exactly the flow this layout prescribes. Every module a contract names must exist, or `lint-imports` stops with an error, so add a layer to the contracts when you create its folder. Run `lint-imports` in CI.

---

## Testing

Tests live in a top level `tests/` folder mirroring the package. This keeps test code out of the shipped wheel, which the src layout is designed for.

```
tests/
├── services/test_db_service.py
├── utils/test_date_utils.py
└── features/booking/test_orchestrator.py
```

Spend effort on `utils/` and `services/`; `testing-and-exceptions.md` has the order to write them in.

Settings are read when `get_settings()` is first called, never at import, so a test imports any service with no environment set. To point services at a throwaway database, patch `myapp.config.settings.get_settings` to return a test `Settings`, or set the env value and call `get_settings.cache_clear()`. Clear `db.get_pool` the same way, and every service follows.

With `unittest` discovery and `-t .`, `tests/` and every folder under it need an `__init__.py`. Without one in `tests/`, discovery stops with an import error; without one in a subfolder, its tests are skipped without a word. With pytest, set `--import-mode=importlib` in its config, or two test files with the same name in different folders fail to collect.

---

## Framework Overrides

The framework wins.

### Django
Django dictates the layout, and it is not the src layout: `django-admin startproject` puts `manage.py` at the repo root with the project package beside it. Its apps are already the feature layer, so there is no parallel `features/` tree.

```
project_root/
├── manage.py
├── config/                  # settings, urls, wsgi. Django calls this the project package.
└── apps/
    └── booking/             # a Django app IS the feature
        ├── models.py
        ├── views.py         # thin, the orchestrator
        ├── services.py      # data logic lives here, not in views
        ├── serializers.py
        └── urls.py
```

Map the Laws onto it: keep views thin, move all query and business logic into a `services.py` inside the app, keep cross app helpers in a shared `utils` app or package, and never import another app's internals directly.

### FastAPI and Flask
Neither dictates a layout, so the base layout applies fully. Routers are the orchestrator (Law 4). Keep the router file free of business logic.

```
src/myapp/
├── features/booking/
│   ├── __init__.py
│   ├── router.py        # FastAPI APIRouter, thin
│   └── _rules.py
├── services/
└── main.py              # composes routers only
```

### Scripts and data pipelines
A single file script does not need this structure. Once it grows past one responsibility, promote it into the package layout and leave a thin entry point in `scripts/`.
