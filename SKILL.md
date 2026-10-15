---
name: coding-structure
description: >
  Use whenever a request writes, changes, or moves code files: scaffolding a project; adding a
  feature, component, screen, endpoint, or service; fixing or extending existing code; or
  refactoring a messy codebase. Also when asked where code should live, how modules may depend on
  each other, how to test a large codebase, or how to keep keys secure. Simple readable functions,
  every file in its layer folder, layered architecture Laws, and folder layouts for
  TypeScript/JavaScript, Python, and Dart/Flutter.
---

# Coding Structure Skill

A codebase that stays simple to read, safe to change, hard to break, and secure by default, at any size.

Two priorities come before everything else:

- **Simple code.** A person reading a function for the first time understands it on one pass. Law 0.
- **Every file in its layer folder.** Files are sorted by layer, so a stranger finds each one where they expect it. Law 3.

Run Step 0, Step 1, and Step 2 in order; when the request only changes existing code, skip straight from Step 0 to Changing Existing Code. Step 2 sends you to a procedure. Below the procedures, Where Does This Code Go? and the Laws are reference: consult them while you work.

**Two moments stop the run and wait for the user's OK:** a leaked key found in the code (Law 15), and the refactor map before anything moves. Everywhere else, "tell the user" or "show the user" means report it and carry on.

---

## Step 0: Resolve the Layout Before Writing Any Path

Each language keeps its own folder names: `src/features/` belongs to TypeScript, Flutter builds under `lib/`, Python lives in a named package. Work down this list. The first match wins. Stop there.

1. **The framework or build tool.** If the project uses a framework or build tool that dictates a layout, that layout wins outright. Flutter requires `lib/`. Django expects apps. Next.js and Expo Router own the `app/` directory. Apply the Laws inside the shape the tool demands. The framework's own folder holds only what the framework requires, such as thin route files; everything else goes into the layer folders beside it.
2. **The existing project, if it is already layered.** If the repo already sorts files into layer or feature folders, match its folder names, even where they differ from the trees below. Consistency beats correctness in an established codebase. **A flat folder of mixed files is not a layout.** It is the problem this skill exists to fix, so put new files into proper layer folders, and tell the user the old files can be moved with the refactor procedure.
3. **The language row below.** Find the language in the table, use its four answers and its folder tree.
4. **No row for the language.** Establish those same four answers and a folder tree from that language's own community convention before writing any path, state them to the user, then proceed on the Laws alone. A language with no row gets its own convention, not a borrowed `src/features/` tree.

Only four decisions and the folder names are language specific. Everything else in this skill is language neutral.

| Language | Sealing mechanism | Import lint tool | File and folder naming | Test location |
|---|---|---|---|---|
| TypeScript, JavaScript | one barrel `index.ts` (`index.js` in plain JavaScript) per feature, no barrels elsewhere | `eslint-plugin-boundaries`, or `import/no-restricted-paths` | PascalCase components, camelCase for everything else | beside the file as `*.test.ts`, end to end in a top level `e2e/` |
| Python | thin `__init__.py` re-export with `__all__`, underscore prefix on private modules | `import-linter` | snake_case modules and packages, PascalCase classes | top level `tests/` mirroring the package |
| Dart, Flutter | one barrel `.dart` file per feature, underscore prefix on private members | a `custom_lint` rule or analyzer plugin for banned imports, CI grep as fallback | snake_case files, PascalCase classes | top level `test/` mirroring `lib/` |

### The folder tree per language

These are the layer folders. Every file you write goes into one of them. Create a folder the first time a file needs it, and skip layers the project does not use. When a file's folder does not exist yet, create the folder.

**TypeScript, JavaScript**
```
src/
├── features/<name>/     one folder per feature, index.ts is its door
├── components/          shared UI
├── hooks/               shared UI logic
├── store/               only when two features share state
├── services/            every DB, HTTP, and SDK call
├── utils/               pure helpers
├── constants/           design tokens and fixed values
├── types/               shared types and Result
├── config/              the one env module
└── navigation/          routes, unless the framework owns them
e2e/                     end to end flows
```

**Python**
```
src/myapp/
├── features/<name>/     __init__.py is its door
├── services/            every DB, HTTP, and SDK call
├── utils/               pure helpers
├── domain/              shared types and Result, never named types/
└── config/              the one settings module
tests/                   mirrors src/myapp/
```

**Dart, Flutter**
```
lib/
├── main.dart, app.dart  entry files
├── features/<name>/     <name>.dart is its door
├── widgets/             shared UI
├── state/               only when two features share state
├── repositories/        domain data access
├── services/            raw external clients
├── utils/               pure helpers
├── models/              shared types and Result
├── config/              the one env file
└── routing/             the router
test/                    mirrors lib/
```

The **source root** is the folder that holds the code: `src/` in TypeScript, the package `src/myapp/` in Python, `lib/` in Dart. It holds entry files only: `main`, `app`, `index`, and Python's `__init__.py` and `__main__.py`. The **project root** above it holds the manifest, config files, data files, and the command that runs the app. A command-line app's run file, such as an `index.js` that users start with `node index.js`, may stay at the project root as long as it only hands over to the source root.

The language references expand these trees into framework overrides, lint config, and worked patterns. Read the matching one before scaffolding a project or adding the first feature to a stack. This table is the only list of them.

| Language | Reference file |
|---|---|
| TypeScript, JavaScript, React, React Native | `references/lang-typescript.md` |
| Python | `references/lang-python.md` |
| Dart, Flutter | `references/lang-dart.md` |

---

## Step 1: Size the Project

Skip this step in Changing Existing Code; that project's size is already settled. A refactor still sizes the project, because the size decides its boundary check.

Structure costs reading time too. Build only as much of it as the code needs today.

A **feature** is one domain noun the user works with, such as invoices, bookings, or reminders, together with every screen, command, or endpoint that acts on it. Listing, paying, and totalling invoices is one feature. Count nouns, not commands. A command belongs to the noun it acts on: reminders about overdue invoices are part of invoices until reminders get screens or commands of their own. A noun the user only sees through another noun, such as the client named on an invoice, is part of that feature even when it has its own data.

| Size | What it looks like | What you build |
|---|---|---|
| Script | one file, one job, run by hand: a scraper, a data fix, a one off migration | One file. It is its own entry point and reads its env values once, at the top. Law 0 and Laws 8 to 11 apply. |
| Small app | one or two features | Only the layer folders the code uses, usually features, services, domain types, and config. The config module and the one error shape from day one. The boundary check once there are two features. |
| Full app | three or more features, or more than one developer | Every step of Starting a New Project, every Law. |

Move up a size the day the code does: a script that takes on a second job becomes a small app, and a small app's third feature brings the full kit. Laws 8, 9, and 10 hold at every size. Whenever you run this step, tell the user the size you picked in one line.

---

## Step 2: Pick the Mode

| The request | Mode |
|---|---|
| Fix, tweak, or extend code that already exists | Changing Existing Code |
| Add a feature, screen, endpoint, or flow | Adding a New Feature |
| Create a new project | Starting a New Project, then Adding a New Feature once per feature |
| Clean up, reorganise, or restructure | Refactoring a Project You Own |
| Review a codebase without changing it | `references/architecture-maturity.md` has the forbidden-arrow table, the secrets table, and a checklist |

---

## Changing Existing Code

The most common mode. The goal is the requested change, done simply, in the right place. Restructuring is a separate request the user makes.

1. Resolve the layout with Step 0, then find the file that owns the job the change touches. Its one-line comment, where it has one, is the contract the code should meet.
2. For a bug fix, first write the test that would have caught it, in the matching test file, and watch it fail.
3. Make the change in the owning file when it fits the file's one-sentence job. When it is a new job, create a new file in its layer folder, per Law 3.
4. Write every function you add or edit to Law 0. An edited function gets its one-line comment if it lacks one. Functions you did not touch stay as they are.
5. Keep the diff to the request and its test. Other problems you notice, including messy or flat structure, are left unfixed for the user to choose: name the three riskiest in your closing message, one line each, security and data loss before correctness before style, then say how many more you saw.

**Done when** the change works and the tests pass, every function you added or edited passes Law 0, every new file sits in its layer folder, and the diff holds only the request and its test.

---

## Adding a New Feature

Bottom up, the same order the refactor section uses: each file imports only files that already exist, so the project compiles at every step.

1. Resolve the layout using Step 0.
2. Put shared shapes in the type layer.
3. Put shared pure helpers in the shared logic layer.
4. Create service functions for any data work, with validation at the edge.
5. Create the feature folder in the resolved location.
6. Create the UI logic layer that bridges UI and services, where the feature has loading, error, or other stateful logic to hold.
7. Create the feature's own UI and the orchestrator, thin. Promote a piece to the shared UI layer only when a second feature uses it.
8. Declare the public door using the language's sealing mechanism.
9. Mount it: the route or entry point imports the public door and renders it.
10. **Check the Laws that catch most mistakes.** Each file has a one-sentence job (Law 2). The orchestrator only composes (Law 4). No import reaches a sibling feature (Law 5). Every DB, HTTP, and SDK call sits in a service (Law 7), which validates outside input (Law 10) and returns the one error shape (Law 12). Env values come only from the config module (Law 11). Run the boundary check if the project has one; a small app with one feature checks these imports by reading them.
11. **Check placement and simplicity.** List the tree of every file you created or changed, and confirm each sits in its layer folder, with nothing new in the source root or in a framework folder. Then check every function against Law 0. Show the user the final tree.

Commit after each layer, so every commit compiles. **Done when** the feature runs from its route, its tests pass, the boundary check (where one exists) passes with no new violations, steps 10 and 11 hold for every new file, and the user has seen the tree.

`references/worked-example.md` runs one feature through these steps in this order, plus the config module a new project builds once, showing the actual files as you would write them.

---

## Starting a New Project

This is the full app procedure. A small app runs steps 1 to 4 and step 7, and creates layer folders as files need them. A script needs step 2 only.

The order matters more than the speed. Installing the boundary check on day one takes ten minutes while the violation count is zero. Retrofitting it at forty files takes a weekend.

1. **Run the official scaffolder for the stack.** `flutter create`, `create-next-app`, `django-admin startproject`, `npm create vite`, `npx create-expo-app`. Accept the shape it produces. Step 0 rule one already says the framework wins, and on a new project that is the normal case. A stack with no scaffolder, such as a plain Python package or a Node CLI, gets its manifest file and the source root from the Step 0 tree, written by hand.
2. **`git init`, write `.gitignore`, then commit the untouched scaffold.** Before writing a line of your own code. That first commit is the clean floor you can always return to, and it keeps the scaffold separate from your work in every later diff. The ignore file comes before the commit, for the reason in Law 15.
3. **Build the typed config module.** One file reads raw environment values and checks the required ones at startup. A missing key should stop the app at boot with a clear message, not surface as a strange bug inside a feature three weeks later.
4. **Define the one error shape and the single parser that produces it.** Decide now what a service returns when it fails. Retrofitting this means touching every service you ever wrote.
5. **Install the boundary check and set it to fail the build.** The Step 0 table names the tool; Law 1 has the fallback when it cannot be installed. Set it to error. There are no violations yet, so there is no cost, and the check now guards every file that follows.
6. **Create the empty folder skeleton.** The layer folders from the Step 0 tree that this project will use, each holding a `.gitkeep` so the commit can carry them. Empty folders make the layer model visible, so the first file lands in the right place.
7. **Set up the test runner with one passing test.** A trivial test that asserts something obvious. The point is that the runner works and the command is known, so writing the second test is free.

Commit after each step. **Done when** each step you ran has its own commit, and the test runner, the boundary check if installed, and the app itself all run clean. Then run Adding a New Feature for feature one.

---

## Refactoring a Project You Own

This is the mode with the most risk. A restructure that half lands leaves a project worse than it was before it started.

**Read the language reference first.** The map below needs its folder names and its lint setup.

**Preconditions.** The app actually runs right now. If it is already broken, fix it before restructuring, otherwise you will never know which break came from where.

**Record a baseline before touching anything.** Run the existing tests. If there are none, run each command or screen once and save what it prints or shows. Run commands that write data against a copy of the data, and stub network calls so nothing is sent. After every later commit, compare against this baseline; a refactor changes structure, never output.

**Sweep for secrets before the first commit.** A project messy enough to need restructuring is likely to have a key pasted into a source file. Search for `key`, `secret`, `token`, `password`, and `Bearer` before running `git add`. When you find one, **stop and tell the user** to revoke it at the provider (Law 15), and wait for their reply. Then move the value into an ignored `.env` file, read it through the config module at its target path, and put a placeholder in `.env.example`. That config file is the one piece of the new structure allowed before the map.

**A clean commit to return to.** With no git yet, run `git init`, write `.gitignore` (Law 15), and commit the project exactly as it stands, minus any secret you just moved out. That commit is the floor every later step can return to.

**Map before moving.** List every source file. Beside each, write its one job, its layer from Where Does This Code Go?, and its target path from the Step 0 tree. A file whose job needs an `and` gets one row per job, because it will be split. **Stop and show the map to the user**, and wait for their OK before moving anything. That table is the plan, and it turns "clean this up" into a list you can tick off.

**Install the boundary check before the first move, at warning level,** when Step 1 calls for one. It then counts violations as they fall, and you flip it to error when the count reaches zero. Setting it to error on day one of a refactor turns the build red for a week, and a build that is always red gets disabled and never re-enabled. A tool with no warning level, such as `import-linter`, runs as a non-blocking CI step until the count is zero, then becomes blocking.

**Move bottom up.** Types first, then pure helpers, then services, then whatever sits between services and features, then features.

```
1. type layer                     nothing depends downward, safest to move
2. shared logic layer             depends only on types
3. service layer                  depends on shared logic, types, config
4. store, shared UI, UI logic     if present, depend on services and below
5. feature layer                  depends on everything above, moves last
```

Each layer you move is only depended on by things above it, so the project keeps running the whole way through. Moving features first breaks everything at once, and you lose the ability to tell a real error from a leftover import.

**Split by extraction.** A file that spans several layers is split by pulling one layer out at a time, in the same bottom up order, one commit per extraction. The original file shrinks with each commit and is deleted when empty.

**One feature per pass.** Move it, run the app, compare against the baseline, commit. Then the next. A commit per feature means a bad move costs you one feature, not the day.

**Keep content out of move commits.** A move commit changes paths: a file goes straight to its target path and name, and only its import lines change. Git then shows it as a rename, and a stray edit stands out in the diff. Changing a file's content in the same commit as its move turns the diff into a delete plus an unrelated new file, and a genuine mistake hides inside it. In a small app, one move commit per layer is enough.

**Know the stop signal.** Every file moves to its layer folder, because placement is what lets the next person find it. Rewriting a file's insides is different: a file that is ugly but stable, and untouched for months, keeps its insides. Simplify what is about to change anyway, and everything when the user asked for readability.

**Simplify after moving.** A move commit changes paths and imports only. Splitting a long function, renaming a vague one, adding the missing one-line comments per Law 0, and bringing services up to Laws 10 and 12 (validated input, the one error shape) come after: one file per commit in a full app, one layer per commit in a small app, each plus the callers a changed signature forces.

**Bugs found on the way** are fixed after the moves, each in its own commit unless a signature change in the simplify phase already fixed it, and listed to the user at the end as behaviour changes, including any error message that now reads differently or any failure the services now report instead of crashing. Behaviour that might be intended, such as which records count as overdue, is a question for the user, not a fix.

**Done when** the boundary check, where the size calls for one, runs at error with zero violations, the app runs and matches the baseline apart from the listed bug fixes, the source root holds entry files only, every file sits in its layer folder, and no commit mixes a move with a content change.

---

## Where Does This Code Go?

Language neutral. The answer is a layer; the Step 0 tree gives its folder.

```
Is it a DB call, HTTP call, or third party SDK call?
  └─ YES → service layer

Is it an env value, key name, or base URL?
  └─ YES → config layer, read once, never read raw env elsewhere

Is it a shared domain type or data shape?
  └─ YES → type layer

Is it state that outlives one screen?
  ├─ Used by one feature only        → stays inside that feature
  ├─ Used by two or more features    → store layer
  └─ A cache of data from a service  → thin wrapper over the service layer, no rules in it

Is it a pure helper that knows nothing about any feature: money, dates, strings, error parsing?
  └─ YES → shared logic layer, even with one caller

Is it used by two or more features?
  ├─ Pure logic that knows a domain rule → shared logic layer
  ├─ Stateful UI logic → UI logic layer, the shared one
  └─ UI element → shared UI layer

Is it used by exactly one feature?
  └─ Stays inside that feature, in a private file named for its job

Is it the entry point of a feature?
  └─ It is the orchestrator, keep it thin
```

Write the file into that layer's folder, creating the folder if it does not exist yet. "Inside that feature" means inside `features/<name>/`.

---

## The Laws

These hold everywhere. No folder names appear here on purpose. This section is the single source for every Law; the reference files cite them by number and show the per-language mechanics.

### 0. Simple code first
Code is read far more often than it is written. Write for the person who opens the file next month. Law 0 decides how code is written; the other Laws decide where it lives. Laws 8, 9, and 10 still hold in full.

- **One function, one job.** Its job fits in one sentence without the word `and`, the same test Law 2 applies to a file. A list of things one verb covers is fine: "formats the row, header, and total" is one job. Two verbs joined by `and` are two jobs.
- **A name that says exactly what it does.** Verb plus object for functions: `fetchInvoices`, `formatMoney`, `validate_booking`. A yes or no question for booleans: `isPaid`, `has_access`. If the honest name needs a generic word to cover several jobs, split the function.
- **A one-line doc comment on every top-level function, method, and class.** Say what the name alone cannot: the unit, an example, the rule it enforces, or what comes back on failure. Use the language's own form: `/** */` above the line in TypeScript and JavaScript, `///` above it in Dart, a docstring as the first line of the body in Python. Test cases, inline callbacks, nested helpers, and door files need none; a test's name is its comment. Inside a body, comment only the reason behind something surprising.
- **Short and flat.** Roughly 30 lines, two levels of nesting at most. Return early so the main path reads straight down.
- **Plain constructs.** An `if` and a loop read faster than a nested ternary, a chained `reduce`, or a one-line trick. Reach for metaprogramming or reflection only where the framework requires it.
- **Direct code until the second use.** Write it inline the first time. Extract a helper, base class, or generic when a second caller actually appears.
- **Three parameters at most.** Past that, pass one named object so every call site reads clearly.
- **Only live code.** Delete unused functions and commented out blocks; git keeps them.

```ts
/** Formats an amount in cents as a currency string, such as 125000 → "RM 1,250.00". */
export function formatMoney(amountCents: number, currency: string, locale: string): string {
  return new Intl.NumberFormat(locale, { style: 'currency', currency }).format(amountCents / 100);
}
```

### 1. Dependencies point one way
Code may depend only on the layer below it. Never sideways, never upward. Files inside one layer may use each other, such as a service calling the shared database client in `services/`, with one exception: a feature never imports a sibling feature (Law 5).

```
feature layer      can use →  UI logic, store, service, shared UI, shared logic, type layers
UI logic layer     can use →  store, service, shared logic, type layers
shared UI layer    can use →  store, shared logic, type layers
store layer        can use →  service, shared logic, type layers
service layer      can use →  shared logic, type, config layers
shared logic layer can use →  type layer only
config layer       can use →  nothing
type layer         can use →  nothing
```

The UI logic layer is the bridge: a hook, a controller, a view model. It holds loading and error state and calls services, so the orchestrator above it can stay a composition of components. A shared UI element does not appear on that line, because a reusable component takes data as a prop rather than fetching its own.

When dependencies flow one way you can rewrite or delete anything in an upper layer without breaking what sits below. The lower a layer sits, the more stable it must be, because more code leans on it.

If you need an upward dependency, the layering is wrong. Move the shared thing down to a layer both sides can reach.

Automate this with a **boundary check**. TypeScript and Python have a tool that fails the build on a bad import; Dart's options are younger and `references/lang-dart.md` says what to do. The Step 0 table names them. When the tool cannot be installed, because the project is offline, standard library only, or has no package manager, a short script in `scripts/` that reads each file's imports and exits non-zero on a forbidden arrow is the boundary check. Prove any check is live by adding one illegal import and watching it fail. A law nobody checks is a suggestion. `references/architecture-maturity.md` lists the arrows that must not exist, for auditing.

**Where a store sits.** A store is any holder of state that outlives a single screen. It is its own layer, above service and below the things that read it. Five rules:

- **App wide state only.** State used by exactly one feature stays inside that feature. Promote it to the store layer the day a second feature needs it. A store is the most expensive place to put state, because everything can reach it.
- **A store may call services. It may never import a feature or a shared UI element.** That is the upward dependency the whole law exists to prevent.
- **A store holds state and the transitions between states.** Business rules belong in the shared logic layer, data access in the service layer. If a store is doing arithmetic or validation, that work belongs one layer down where it can be tested without the store.
- **A server state cache is not a store.** A cache of remote data is a thin wrapper over the service layer. It may hold keys, staleness, and retry policy. It holds no business rules, and it never becomes the place features talk to each other through.
- **One store per domain, not one store for the app.** Law 2 applies to a store like any other file.

The folder name is yours. `store/`, `contexts/`, `providers/`, `state/` are all fine, and so is a library or the language's own built in mechanism. What matters is one place for it, with the arrows pointing the same way. Pick a name on day one and keep it, because two names for one layer is worse than either name.

If your project has no store, the layer simply does not exist and everything above collapses one step. Create one when two features share state.

### 2. One responsibility per file
Describe each file's job in one sentence without the word `and`; when you cannot, split it. Name each file for its one job. A `helpers` or `utils` file inside a feature is the usual way this law gets broken.

### 3. A new job gets a new file, in its layer folder
A new capability gets its own file when it is a separate job: a new component, a new service domain, a new validator. A function that shares an existing file's one-sentence job joins that file, such as `cancelBooking` beside `createBooking` in the booking service. A private helper used by one function stays in the same file, just below the function that calls it. The test after any addition: the file still fits its one sentence without an `and`.

A new file goes into the folder for its layer, found with Where Does This Code Go? and named with the Step 0 tree. Many small files read well only when folders sort them.

A folder whose files do different jobs, such as a service beside a screen beside a helper, is a dumping ground. Split it by layer, then by feature.

### 4. The entry point orchestrates, never implements
The screen, widget, route handler, or module entry composes things. It does not define them. No business rules, no raw client calls, no formatting. What it pulls out goes into the feature's private files grouped by job, not one file per function: all of a feature's display formatting can share one formatter file. It calls the UI logic layer where one exists; where the stack has no UI logic layer between entry point and service, such as a backend route handler, it calls one service function and returns. In a command-line app, the entry file parses the arguments and calls one door function per command; parsing arguments is composition, not implementation.

### 5. Features never touch each other
A feature must not import from a sibling feature. When one needs something from another, there are three paths: move the shared work down into a service, move the shared shape into the type layer, or use an event so the sender does not know who listens. A shared store is not a fourth path. Writing to a store in feature A so feature B can read it is a sibling import with extra steps: the coupling is still there, it has just left the import line where a lint rule could catch it.

### 6. Every feature has one public door
The outside world enters a feature through one declared surface. Everything else is private and may change freely. The door holds exports and no logic. Use the strongest sealing mechanism the language offers, not a barrel file by habit. The Step 0 table names the mechanism per language.

### 7. All external interaction goes through a service layer
No UI, widget, screen, or route handler ever calls a database, HTTP client, or third party SDK directly. One layer owns that, and it is also the security gate.

### 8. The client is never the authority
Treat every request from an app, browser, or kiosk as if a hostile person wrote it by hand. Identity, role, and ownership come from the verified server session, never from a field the client sent.

### 9. Secrets stay server side
A public client key may ship. Any admin key, service key, or paid third party key must live only on a server. A mobile app bundle is not private. An APK can be unzipped, an IPA can be dumped, a JS bundle can be read in dev tools, a compile time constant is recoverable from all three. If a client needs a paid API, it calls your server and your server holds the key. `references/architecture-maturity.md` has the table sorting each kind of key to where it may live.

### 10. Validate at the boundary
Every value entering from outside is checked at the service edge before use or storage. Past the gate, code trusts its inputs. Before the gate, it trusts nothing. Public paths with no login also need rate limiting.

### 11. One config source
Every environment value, key name, base URL, and flag flows through one typed module. No other file reads a raw environment value. A value every path needs is checked at startup, so a misconfiguration fails loudly at boot, not silently inside a feature. A value only one command or route needs, such as one mailer key, is checked by a config function that the one service calls, so the rest of the app still runs without it.

### 12. One error shape
Services return a consistent result rather than throwing to the caller. The result type is declared once, in the type layer, and every service imports it. One place parses raw errors into clean messages. One place logs them. The UI decides how to show an error, never how to parse one. Where the language has a sum type, use it: a Dart `sealed` class, a TypeScript discriminated union, or a Python union of two dataclasses forces the caller to handle the failure branch, which a pair of optional fields does not.

### 13. Length is a signal, not a law
Roughly 400 lines for a file, roughly 120 for an orchestrator, roughly 30 for a function, is a prompt to check whether it took on a second responsibility. A long file that still does exactly one thing may stay long. `references/testing-and-exceptions.md` lists the cases where that is so.

### 14. Test by risk and by layer
Spend most effort on the shared logic and service layers, which are pure or near pure and hold the logic that matters. Spend least on screens and widgets, which change often and produce brittle tests. Cover those with a few end to end flows. `references/testing-and-exceptions.md` has the order to write them in.

### 15. Version control before the first line of code
Run `git init` before writing your own code (a scaffolder's output is the first commit), commit a working state before restructuring, and keep one logical change per commit. This is local git: nothing leaves the machine, and a remote is a separate choice.

**Ignore secrets before the first commit.** A commit is permanent. Deleting a key in a later commit leaves it in the history that every clone receives.

```
# .gitignore, written before git add
.env
.env.*
!.env.example
*.pem
*.keystore
google-services.json
GoogleService-Info.plist
*.db
*.sqlite
*.sqlite3
```

Add the stack's dependency and build folders, and any local data file that holds real records.

**If a key already sits in a tracked file**, do these in order before committing anything:

1. **Revoke and reissue the key at the provider.** Rewriting history does not call a leaked key back.
2. **Move the value into an ignored env file** read through the config module, per Law 11. If that file is already tracked, run `git rm --cached path/to/file` first, or git keeps committing it.
3. **Then commit.**

`references/git-and-secrets.md` explains why each step sits in that order, why a folder copy is no substitute for commits, and lists the six commands a newcomer to git needs. Read it when the user is new to git or asks why.

### A note on the examples in this skill

An example may name a domain. A rule may not. Where a reference file illustrates a point with a booking, a payment, or any other concrete product, that is there to show the shape of the answer, never to supply it. Derive the answer from the project in front of you.
