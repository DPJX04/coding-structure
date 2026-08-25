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
│       ├── features/
│       │   ├── __init__.py
│       │   └── booking/
│       │       ├── __init__.py     # the public door, re-exports only
│       │       ├── orchestrator.py # entry point for the feature
│       │       ├── _validator.py   # private, leading underscore
│       │       ├── _processor.py
│       │       └── _formatter.py
│       ├── services/
│       │   ├── __init__.py
│       │   ├── db_service.py
│       │   └── email_service.py
│       ├── utils/
│       │   ├── __init__.py
│       │   ├── error_handler.py
│       │   └── date_utils.py
│       ├── types/                  # shared dataclasses, TypedDicts, or pydantic models, and Result
│       │   ├── __init__.py
│       │   ├── booking.py
│       │   └── result.py
│       └── config/
│           ├── __init__.py
│           ├── settings.py         # the only file that reads raw env
│           └── db.py               # connection pool, built once from settings
├── tests/                          # mirrors src/myapp/
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

---

## The Dependency Law, Mapped

Law 1 translated into this layout's folder names. The diagram in `SKILL.md` stays the authoritative copy.

```
features/  can use →  services, utils, types
services/  can use →  utils, types, config
utils/     can use →  types only
types/     can use →  nothing
```

There is no UI logic layer in a backend, so the feature's entry point composes its private helpers and its one service call directly, per Law 4: it may call them, it defines none of them.

### Why there is no store layer here

A server handles one request at a time and forgets it. There is usually no state that outlives a request, so the store layer from the Laws simply does not exist in a Python backend and you should not create a folder for it.

Two things get mistaken for one:

- **Request scoped state.** A user, a tenant, a correlation id. Pass it as an argument, or use a framework dependency such as FastAPI's `Depends`. Never a module level global, which is shared across every request in the process and is a data leak between users waiting to happen.
- **Process wide caches and connection pools.** These belong in `config/` or `services/`, created once at startup and injected. They are infrastructure, not application state.

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
from myapp.services.db_service import insert_booking
from myapp.types.booking import Confirmation
from myapp.types.result import Err, Ok, Result
from ._validator import validate_booking
from ._formatter import to_confirmation

def create_booking(user_id: str, slot_id: str) -> Result[Confirmation]:
    error = validate_booking(user_id, slot_id)
    if error:
        return Err(error)

    result = insert_booking(user_id, slot_id)
    if isinstance(result, Err):
        return result

    return Ok(to_confirmation(result.data))
```

### Sealing
Python cannot enforce privacy, so it relies on two conventions and one tool:

1. Leading underscore on private modules and functions. `_validator.py` signals internal.
2. `__all__` in the door, which declares the public surface.
3. `import-linter`, which fails the build on an illegal import. See below.

---

## Naming

| Type | Convention | Example |
|---|---|---|
| Module | snake_case | `slot_validator.py` |
| Private module | leading underscore | `_processor.py` |
| Package or folder | snake_case | `booking_slots/` |
| Class | PascalCase | `BookingRequest` |
| Function and variable | snake_case | `create_booking` |
| Constant | UPPER_SNAKE | `MAX_SLOTS` |
| Test file | `test_` prefix | `test_db_service.py` |

---

## The Result Shape and the Service Pattern

Law 12: one error shape, declared once in `types/`, imported by every service. Python's sum type is a union of two frozen dataclasses, narrowed with `isinstance` or `match`.

```python
# src/myapp/types/result.py
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

Services are named functions, each handling its own failure and returning a `Result`. Type hints are not optional in a large codebase.

```python
# src/myapp/services/db_service.py
from datetime import date as iso_date
from myapp.config.db import pool
from myapp.utils.error_handler import parse_error
from myapp.types.booking import Slot
from myapp.types.result import Err, Ok, Result

def _to_slot(row: dict) -> Slot:
    if not isinstance(row.get("id"), str):
        raise ValueError("slot row is missing an id")
    return Slot(**row)

def fetch_slots(date: str) -> Result[list[Slot]]:
    try:
        iso_date.fromisoformat(date)
        with pool.connection() as conn:
            rows = conn.execute(
                "SELECT * FROM slots WHERE date = %s", (date,)
            ).fetchall()
        return Ok([_to_slot(row) for row in rows])
    except Exception as err:
        return Err(parse_error(err))
```

`fromisoformat` and `_to_slot` are the Law 10 gate: input is rejected before it reaches SQL, and rows are checked before they become models. In a real project a pydantic model does `_to_slot` better; the point is that something checks. Law 8 in this stack: identity comes from the framework's verified session, `request.user` in Django or a FastAPI `Depends` that resolves the session, never from a field in the request body. An endpoint with no login gets a rate limiter such as `django-ratelimit` or `slowapi`.

---

## Config Pattern

One typed settings module, validated on import, so a missing value fails at boot rather than inside a request.

```python
# src/myapp/config/settings.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    database_url: str
    api_key: str
    is_prod: bool = False

settings = Settings()  # raises immediately if a required value is missing
```

No other module reads `os.environ`. The connection pool in `config/db.py` is built once from `settings.database_url` and imported by services.

---

## Types

Shared shapes and `Result` live in `types/` and depend on nothing. Use dataclasses, `TypedDict`, or pydantic models. Type hint every public function, and treat a bare `dict` passed between layers as the escape hatch it is. Run `mypy` or `pyright` in strict mode on `services/`, `utils/`, and `types/` at minimum, since those are the layers everything else leans on.

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
    myapp.types

[importlinter:contract:features]
name = Features are independent
type = independence
modules =
    myapp.features.booking
    myapp.features.auth
    myapp.features.notifications

[importlinter:contract:config]
name = Only services read config
type = forbidden
source_modules =
    myapp.features
    myapp.utils
    myapp.types
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
    myapp.types
```

The `layers` contract enforces the one way flow. The `independence` contract enforces that features never touch each other. The two `forbidden` contracts pin `config` down: only `services` reads it, and it reads nothing back. `allow_indirect_imports` is required on the first, because by default a forbidden contract also fails on the chain `features → services → config`, which is exactly the flow this layout prescribes. Run `lint-imports` in CI.

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
