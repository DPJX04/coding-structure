# Layout: Dart and Flutter

Flutter dictates the top of the tree. There is no `src/` in a Flutter app. The tooling requires `lib/` as the source root and `test/` at the project root, and the linter enforces snake_case file names. Apply the Laws inside that shape.

---

## Folder Layout

```
project_root/
├── pubspec.yaml
├── analysis_options.yaml
├── lib/
│   ├── main.dart                       # entry point, boots the app, nothing else
│   ├── app.dart                        # root widget, theme, router wiring
│   ├── features/
│   │   └── booking/
│   │       ├── booking.dart            # the public door, export statements only
│   │       ├── booking_screen.dart     # orchestrator widget
│   │       ├── booking_controller.dart # state, bridges UI and repository
│   │       ├── booking_validator.dart  # rules
│   │       └── widgets/
│   │           └── slot_list.dart      # feature scoped widget
│   ├── widgets/                        # shared UI, used by two or more features
│   │   ├── app_button.dart
│   │   └── app_modal.dart
│   ├── repositories/                   # domain data access, the service layer
│   │   ├── booking_repository.dart
│   │   └── auth_repository.dart
│   ├── services/                       # raw external clients wrapped thin
│   │   ├── api_client.dart
│   │   └── storage_service.dart
│   ├── utils/                          # pure helpers
│   │   ├── date_utils.dart
│   │   └── error_handler.dart
│   ├── models/                         # shared domain types and Result, lowest layer
│   │   ├── booking.dart
│   │   └── result.dart
│   ├── config/
│   │   └── env.dart                    # the only file reading compile time values
│   └── routing/
│       └── app_router.dart
├── test/                               # mirrors lib/
└── integration_test/                   # end to end flows
```

### Repository versus service
Flutter convention splits what other stacks call one service layer into two:

- **Service** wraps a raw external client. An HTTP client, a local database, a plugin. It knows nothing about your domain.
- **Repository** owns a domain concept. It calls one or more services, converts raw responses into your models, handles caching, and is what the rest of the app talks to.

Features call repositories. Repositories call services. Widgets call neither directly.

---

## The Dependency Law, Mapped

Law 1 translated into this layout's folder names. The diagram in `SKILL.md` stays the authoritative copy.

```
features/      can use →  state, repositories, widgets, utils, models
widgets/       can use →  state, utils, models
state/         can use →  repositories, utils, models
repositories/  can use →  services, utils, models, config
services/      can use →  utils, models, config
utils/         can use →  models only
models/        can use →  nothing
```

`routing/`, `app.dart`, and `main.dart` sit above `features/`: they import public doors and wire them up, and nothing imports them. There is no shared UI-logic folder by convention: a controller stays in its feature, and a controller two features share is shared state and moves to `state/`.

### State, and where it lives

The store rules are Law 1 in `SKILL.md`. The Flutter mechanics:

A controller that belongs to one feature stays in that feature, as `booking_controller.dart` shows above. Add a shared state folder only when two or more features need the same state.

```
lib/state/                  # or providers/, or store/, pick one and keep it
├── session_state.dart      # one holder per domain, not one for the app
└── cart_state.dart
```

- Whatever the mechanism, a provider, a notifier, an inherited widget, or a plain singleton, the layer rule is the same. The mechanism is a detail. The direction of the arrow is not.
- A cache of remote data belongs in the repository that owns that domain, not in shared state. The repository already exists for exactly this.

---

## Feature Anatomy and the Public Door

Dart privacy is per library, and a file is its own library unless it uses `part`. A leading underscore makes a member private to its file. That is a real language mechanism, unlike a naming convention.

Combine it with a barrel:

```dart
// lib/features/booking/booking.dart
export 'booking_screen.dart' show BookingScreen;
export 'booking_controller.dart' show BookingController;
```

Outside code imports `package:myapp/features/booking/booking.dart`, never a file inside the feature. The `show` clause narrows the surface further, which is stronger than a bare re-export.

Inside the feature, mark anything not in the door as private with a leading underscore where the language allows it. A widget used only by `booking_screen.dart` should be `_SlotHeader`, declared in that file.

---

## Naming

The Dart linter enforces some of this. Do not fight it.

| Type | Convention | Example |
|---|---|---|
| File and folder | snake_case, required by lint | `booking_screen.dart` |
| Class, enum, typedef | PascalCase | `BookingScreen`, `SlotStatus` |
| Function, variable, parameter | camelCase | `fetchSlots`, `slotId` |
| Constant | camelCase, `k` prefix is legacy, avoid it | `maxSlots` |
| Private member | leading underscore | `_SlotHeader`, `_validate` |
| Feature barrel | matches folder name | `features/booking/booking.dart` |
| Test file | mirrors source, `_test` suffix | `test/utils/date_utils_test.dart` |

---

## Widget Rules, Flutter Specific

These matter more here than in any other stack, because a Flutter widget tree nests deeply and rebuild cost is real.

### Extract widget classes, never helper methods
A method that returns a widget looks tidy and costs you. It has no separate element in the tree, so it cannot be `const` and it rebuilds every time the parent rebuilds. A separate widget class gets its own build context and lets Flutter skip rebuilding it. Turn on `prefer_const_constructors` so the analyzer finds the widgets that could be `const` and are not.

```dart
// Wrong, rebuilds with the whole parent every time
Widget _buildHeader() => Row(children: [...]);

// Right, its own element, can be const, rebuilds independently
class _SlotHeader extends StatelessWidget {
  const _SlotHeader();

  @override
  Widget build(BuildContext context) => Row(children: const [...]);
}
```

### Keep the orchestrator screen thin
A screen widget composes and reads state. It does not call a repository, hold business rules, or format data. Past roughly 120 lines, extract a child widget (Law 13).

### State management is the controller layer
Whatever you use, Riverpod, Bloc, Provider, or a plain `ChangeNotifier`, that object plays the role the hook plays in React. It calls the repository, holds loading and error state, and hands clean values to the widget. Pick one approach for the whole project and never mix two.

---

## The Result Shape and the Repository Pattern

Law 12: one error shape, declared once in the lowest layer, imported by every repository. A sealed class, so the compiler forces the caller to handle both branches in a switch.

```dart
// lib/models/result.dart
sealed class Result<T> {
  const Result();
}
class Ok<T> extends Result<T> {
  final T data;
  const Ok(this.data);
}
class Err<T> extends Result<T> {
  final String error;
  const Err(this.error);
}
```

```dart
// lib/repositories/booking_repository.dart
import 'package:myapp/models/booking.dart';
import 'package:myapp/models/result.dart';
import 'package:myapp/services/api_client.dart';
import 'package:myapp/utils/error_handler.dart';

class BookingRepository {
  BookingRepository(this._api);
  final ApiClient _api;

  Future<Result<List<Slot>>> fetchSlots(String date) async {
    try {
      final raw = await _api.get('/slots', query: {'date': date});
      return Ok(raw.map(Slot.fromJson).toList());
    } catch (err) {
      return Err(parseError(err));
    }
  }
}
```

`Slot.fromJson` is the Law 10 gate: it throws on a missing or mistyped field, so a malformed row becomes an `Err` here rather than a model the rest of the app trusts. Everything arriving from the network or local storage passes through a `fromJson` before it becomes a model.

---

## Models

Shared domain types live in `lib/models/` and depend on nothing. Generate serialization rather than hand writing it. `json_serializable` or `freezed` both work, and `freezed` also gives you immutability and copy methods for free. Generated files are allowed to be long, since they do exactly one thing. Ban `dynamic` the way TypeScript bans `any`, and turn on `strict-casts` and `strict-raw-types` in the analyzer, shown below.

---

## Config and Secrets

Law 9 applies in full, and Flutter has two traps of its own. Values passed with `--dart-define` are compiled into the binary and recoverable. A `.env` file loaded with an env plugin is shipped as a plain asset and readable with a file browser. Neither hides a secret; they only carry public configuration.

Route all compile time values through one file.

```dart
// lib/config/env.dart
class Env {
  static const apiUrl = String.fromEnvironment('API_URL');
  static const isProd = bool.fromEnvironment('dart.vm.product');

  static void assertConfigured() {
    if (apiUrl.isEmpty) {
      throw StateError('Missing API_URL. Pass it with --dart-define.');
    }
  }
}
```

Call `Env.assertConfigured()` at the top of `main()` so a misconfiguration fails at launch, not inside a screen.

---

## Analyzer and Import Rules

Start with the analyzer, which catches the cheap things:

```yaml
# analysis_options.yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  language:
    strict-casts: true
    strict-raw-types: true
  errors:
    prefer_const_constructors: warning

linter:
  rules:
    - always_declare_return_types
    - prefer_const_constructors
    - avoid_print
    - file_names
```

The analyzer cannot express layer rules on its own. For that, add a `custom_lint` rule or an analyzer plugin that supports banned imports per folder, then declare the forbidden directions from the map above. Check what is current on pub.dev before adding one, since this corner of the ecosystem moves faster than the TypeScript and Python tools. If no package fits the project, a short CI script that greps `lib/` for illegal `package:myapp/...` import paths and fails the job is the fallback, and it still counts as the machine checking Law 1.

---

## Testing

Mirror `lib/` inside `test/`. Flutter requires this split, so tests never sit beside source here.

```
test/
├── utils/date_utils_test.dart
├── repositories/booking_repository_test.dart
└── features/booking/booking_controller_test.dart
integration_test/
└── booking_flow_test.dart
```

Spend effort per Law 14; `testing-and-exceptions.md` has the order to write them in. One Flutter-specific note: repository tests run against a fake service, because an app has no throwaway database of its own to test against.

Keep the controller free of Flutter imports where you can. A controller you can test without pumping a widget is far faster to test.

---

## Framework Notes

There is no framework override to worry about, because Flutter is the framework and its layout is already the top of this document. Two variations worth knowing:

- **Melos or a monorepo.** Shared code moves into sibling packages under `packages/`, each with its own `lib/`. The Laws apply per package, and package boundaries become a stronger version of the layer boundary, since a package cannot import from an app that depends on it.
- **Pure Dart, no Flutter.** Server or CLI Dart uses the same `lib/` and `test/` layout, with `bin/` for executables. Drop the widget rules, keep everything else.
