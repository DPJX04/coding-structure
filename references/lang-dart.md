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
│   ├── repositories/                   # domain data and app wide state, the service layer
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
- **Repository** owns a domain concept. It calls one or more services, converts raw responses into your models, handles caching, holds app wide state such as the signed in session, and is what the rest of the app talks to. It never imports another repository.

Features call repositories. Repositories call services. Widgets call neither directly.

---

## The Dependency Law, Mapped

Law 1 translated into this layout's folder names. The diagram in `SKILL.md` stays the authoritative copy.

```
features/      can use →  repositories, widgets, utils, models
widgets/       can use →  utils, models
repositories/  can use →  services, utils, models, config (never another repository)
services/      can use →  utils, models, config
utils/         can use →  models only
models/        can use →  nothing
```

`routing/`, `app.dart`, and `main.dart` sit above `features/`: they import public doors and wire them up, and nothing imports them. There is no shared UI-logic folder: a controller is never shared.

### State, and where it lives

Flutter has no store layer, so there is no `state/` folder. Law 1 in `SKILL.md` hands this choice to the framework's own guide, and Flutter's guide keeps app wide state in the repository that owns the data. Two features that read or change that state are both using the service layer, not each other, so Law 5 holds. The Flutter mechanics:

- A controller belongs to one screen and stays in its feature, as `booking_controller.dart` shows above. Two screens that show the same data each keep their own controller.
- App wide state, such as the signed in session or an in-memory cache, lives in the repository that owns that data. The repository exposes it as a `Stream` or a `Listenable`, and each controller that needs it listens.
- Repositories never import each other, which is narrower than Law 1 allows inside a layer. Logic that needs two of them takes both in the controller and combines them there. A value every request needs, such as the session token, reaches the client as a callback that the owning repository sets on it.
- Whatever the mechanism, a provider, a notifier, or a stream, the layer rule is the same. The mechanism is a detail. The direction of the arrow is not.

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
| File | snake_case, required by the `file_names` lint | `booking_screen.dart` |
| Folder | snake_case, by convention, not linted | `features/booking/` |
| Class, enum, typedef | PascalCase | `BookingScreen`, `SlotStatus` |
| Function, variable, parameter | camelCase | `fetchSlots`, `slotId` |
| Constant | camelCase, `k` prefix is legacy, avoid it | `maxSlots` |
| Private member | leading underscore | `_SlotHeader`, `_validate` |
| Feature barrel | matches folder name | `features/booking/booking.dart` |
| Test file | mirrors source, `_test` suffix | `test/utils/date_utils_test.dart` |
| Function comment | one line `///` above every function and method | `/// Fetches the open slots for one date.` |

---

## Widget Rules, Flutter Specific

These matter more here than in any other stack, because a Flutter widget tree nests deeply and rebuild cost is real.

### Import Material from its own package
Since Flutter 3.47, Material lives in `package:material_ui/material_ui.dart` and Cupertino in `package:cupertino_ui/cupertino_ui.dart`. The old `package:flutter/material.dart` import is deprecated. A file that needs only the base widgets imports `package:flutter/widgets.dart`. `dart fix --apply` migrates old imports.

### Extract widget classes, never helper methods
A method that returns a widget looks tidy and costs you. It has no separate element in the tree, so it cannot be `const` and it rebuilds every time the parent rebuilds. A separate widget class gets its own build context and lets Flutter skip rebuilding it. Write `const` wherever a widget allows it. The `prefer_const_constructors` lint finds the ones you missed. It is no longer in `flutter_lints` because it is noisy, so turn it on only if you want it.

```dart
// Wrong, rebuilds with the whole parent every time
Widget _buildHeader() => Row(children: [...]);

// Right, its own element, can be const, rebuilds independently
class _SlotHeader extends StatelessWidget {
  const _SlotHeader();

  @override
  Widget build(BuildContext context) => const Row(children: [...]);
}
```

### Keep the orchestrator screen thin
A screen widget composes and reads state. It does not call a repository, hold business rules, or format data. Past roughly 120 lines, extract a child widget (Law 13).

### State management is the controller layer
Whatever you use, Riverpod, Bloc, Provider, or a plain `ChangeNotifier`, that object plays the role the hook plays in React. It calls the repository, holds loading and error state, and hands clean values to the widget. Pick one approach for the whole project and never mix two.

Build each service and repository once, in `main.dart` or `app.dart`, and pass it in through the constructor of whatever uses it, as `BookingRepository(this._api)` does below. Nothing constructs its own dependencies. The Flutter team's default is a `ChangeNotifier` controller with `package:provider` doing that injection. Provider there is the wiring, not a second state approach.

---

## The Result Shape and the Repository Pattern

Law 12: one error shape, declared once in the lowest layer, imported by every repository. A sealed class, so the compiler forces the caller to handle both branches in a switch.

```dart
// lib/models/result.dart
sealed class Result<T> {
  const Result();
}
final class Ok<T> extends Result<T> {
  final T data;
  const Ok(this.data);
}
final class Err<T> extends Result<T> {
  final Exception error;
  const Err(this.error);
}
```

```dart
// lib/repositories/booking_repository.dart
import 'package:myapp/models/booking.dart';
import 'package:myapp/models/result.dart';
import 'package:myapp/services/api_client.dart';

class BookingRepository {
  BookingRepository(this._api);
  final ApiClient _api;

  /// Fetches every slot on one date. Returns [Err] instead of throwing.
  Future<Result<List<Slot>>> fetchSlots(String date) async {
    try {
      final raw = await _api.get('/slots', query: {'date': date}); // Object?
      if (raw is! List<Object?> || raw.any((row) => row is! Map<String, dynamic>)) {
        throw const FormatException('Expected a list of slot objects.');
      }
      return Ok(raw.cast<Map<String, dynamic>>().map(Slot.fromJson).toList());
    } on Exception catch (err) {
      return Err(err);
    }
  }
}
```

The shape check and `Slot.fromJson` are the Law 10 gate. With `checked: true` (see Models), a missing or mistyped field throws `CheckedFromJsonException`, so a malformed row becomes an `Err` here rather than a model the rest of the app trusts. Catch `on Exception`, never everything: a bug in your own code throws an `Error`, which must still crash loudly. Everything arriving from the network or local storage passes through a `fromJson` before it becomes a model.

`Err` carries the exception itself, not a message. The service turns a raw failure into a typed exception declared in `models/`, such as offline or session expired, and the repository returns it inside `Err`. The controller may branch on that type to choose what to show, which Law 12 leaves to the UI side. Turning it into text is parsing, and only `parseError` in `utils/error_handler.dart` does that.

---

## Models

Shared domain types live in `lib/models/` and depend on nothing. Generate serialization rather than hand writing it. `json_serializable` or `freezed` both work, and `freezed` also gives you immutability and copy methods for free. Generated files are allowed to be long, since they do exactly one thing.

Set `checked: true` for every model, in `build.yaml` under the `json_serializable` builder's options. Without it, a missing or mistyped field throws a `TypeError`, which `on Exception` rightly does not catch. Generated `fromJson` checks types, not values: `DateTime.parse` turns `2026-02-31` into 3 March, and an empty string passes. Put format, range, and emptiness rules in a `JsonKey(fromJson: ...)` converter that throws `FormatException`. A freezed `@Assert` is a debug assert and is gone in release builds.

Prefer `Object?` to `dynamic`. The one accepted use is JSON: generated code takes `Map<String, dynamic>`, and that type stays at the boundary. The strict modes and `avoid_dynamic_calls` below catch what `dynamic` would hide.

---

## Config and Secrets

Law 9 applies in full, and Flutter has two traps of its own. Values passed with `--dart-define` are compiled into the binary and recoverable. A `.env` file loaded with an env plugin is shipped as a plain asset and readable with a file browser. Neither hides a secret; they only carry public configuration.

Route all compile time values through one file.

```dart
// lib/config/env.dart
class Env {
  static const apiUrl = String.fromEnvironment('API_URL');
  static const isProd = bool.fromEnvironment('dart.vm.product');

  /// Stops the app at launch if a required value was not passed in.
  static void assertConfigured() {
    if (apiUrl.isEmpty) {
      throw StateError('Missing API_URL. Pass it with --dart-define.');
    }
  }
}
```

Call `Env.assertConfigured()` at the top of `main()` so a misconfiguration fails at launch, not inside a screen. For many values, keep them in a JSON file and pass `--dart-define-from-file=dev.json`. `Env` reads them the same way, and they are compiled in the same way, so that file holds public configuration only.

---

## Analyzer and Import Rules

Start with the analyzer, which catches the cheap things:

```yaml
# analysis_options.yaml, merged into the generated one (keep its exclude: list)
include: package:flutter_lints/flutter.yaml

analyzer:
  language:
    strict-casts: true
    strict-inference: true
    strict-raw-types: true

linter:
  rules:
    - always_declare_return_types
    - avoid_dynamic_calls
    - avoid_print
    - file_names
```

The analyzer cannot express layer rules on its own. For that, add an analyzer plugin that bans imports per folder, under the top-level `plugins:` key (Dart 3.10 and later), then declare the forbidden directions from the map above. Check pub.dev for one that resolves against your SDK, since this corner moves fast. Older `custom_lint` rules clash with current `build_runner`. One that works is `dart_code_linter` with `avoid-banned-imports`, one entry per arrow:

```yaml
# analysis_options.yaml
plugins:
  dart_code_linter:
    version: ^4.4.5
    diagnostics:
      avoid-banned-imports: error

dart_code_linter:
  rules:
    - avoid-banned-imports:
        entries:
          - paths: ['.*/lib/services/.*\.dart']
            deny: ['(^|/)(features|widgets|repositories)/']
            message: 'services/ may use only utils, models, and config.'
```

The `deny` pattern matches the import as written, so it catches `package:myapp/...` and relative paths alike. A plugin that fails to resolve reports nothing at all, no error and no warning. So the proof is required: plant one illegal import as a `package:` path and one as a relative path, and watch both fail.

If no plugin fits the project, a short CI script that greps `lib/` for illegal `package:myapp/...` import paths and fails the job is the fallback. Make it fail on any relative import too, and add `always_use_package_imports` to the linter rules, so no import slips past the grep. It still counts as the machine checking Law 1.

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
