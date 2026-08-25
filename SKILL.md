---
name: coding-structure
description: >
  Use whenever a request produces or moves code files (writing, scaffolding, or adding a feature,
  component, screen, endpoint, service, or hook; refactoring or reorganising a codebase), or asks
  where code should live, how modules may depend on each other, how to test a large codebase, or
  how to keep keys and data secure. Layered architecture Laws plus per-language folder layouts for
  TypeScript/JavaScript, Python, and Dart/Flutter.
---

# Coding Structure Skill

A strict architecture with one goal: a large, complex codebase that stays simple to read, safe to change, hard to break, and secure by default.

This skill has two halves.

- **The Laws.** Language neutral. True in every codebase, every language, every framework.
- **The Layout.** Language specific. Folder names, file names, and sealing mechanisms differ per language and per framework.

Each language keeps its own folder names: `src/features/` belongs to TypeScript, Flutter builds under `lib/`, Python lives in a named package. Read the layout rule first, then write paths.

---

## Step 0: Resolve the Layout Before Writing Any Path

Work down this list. The first match wins. Stop there.

1. **The framework or build tool.** If the project uses a framework or build tool that dictates a layout, that layout wins outright. Flutter requires `lib/`. Django expects apps. Next.js and Expo Router own the `app/` directory. Apply the Laws inside the shape the tool demands.
2. **The existing project.** If the repo already has a consistent layout, match it. Consistency beats correctness in an established codebase. Note the mismatch to the user, then follow what is there.
3. **The language row below.** Find the language in the table and use its four answers.
4. **No row for the language.** Establish those same four answers from that language's own community convention before writing any path, state them to the user, then proceed on the Laws alone. A language with no row gets its convention, never a borrowed `src/features/` tree.

Only four decisions are truly language specific. Everything else in this skill is language neutral.

| Language | Sealing mechanism | Import lint tool | File and folder naming | Test location |
|---|---|---|---|---|
| TypeScript, JavaScript | one barrel `index.ts` per feature, no barrels elsewhere | `eslint-plugin-boundaries`, or `import/no-restricted-paths` | PascalCase components, camelCase for everything else | beside the file as `*.test.ts`, end to end in a top level `e2e/` |
| Python | thin `__init__.py` re-export with `__all__`, underscore prefix on private modules | `import-linter` | snake_case modules and packages, PascalCase classes | top level `tests/` mirroring the package |
| Dart, Flutter | one barrel `.dart` file per feature, underscore prefix on private members | a `custom_lint` rule or analyzer plugin for banned imports, CI grep as fallback | snake_case files, PascalCase classes | top level `test/` mirroring `lib/` |

For a new project you rarely need more than this row, because the official scaffolder has already decided the folder shape. See Starting a New Project below.

The language references expand those four answers into folder trees, framework overrides, lint config, and worked patterns. Read one only when the row above is not enough. The table below is the only list of the language references.

| Language | Reference file |
|---|---|
| TypeScript, JavaScript, React, React Native | `references/lang-typescript.md` |
| Python | `references/lang-python.md` |
| Dart, Flutter | `references/lang-dart.md` |

**Then pick the mode.** Three procedures at the end of this file, one per run: Starting a New Project, Adding a New Feature, or Refactoring a Project You Own. The Laws between here and there apply in all three.

---

## The Laws

These hold everywhere. No folder names appear here on purpose. This section is the single source for every Law; the reference files cite them by number and show the per-language mechanics.

### 1. Dependencies point one way
Code may depend only on the layer below it. Never sideways, never upward.

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

When dependencies flow one way you can rewrite or delete anything in an upper layer without breaking what sits below. The lower a layer sits, the more stable it must be, because more code leans on it. Break this and every change risks breaking something unrelated.

If you need an upward dependency, the layering is wrong. Move the shared thing down to a layer both sides can reach.

Automate this. TypeScript and Python have a tool that fails the build on a bad import; Dart's options are younger and `references/lang-dart.md` says what to do. The Step 0 table names them. A law nobody checks is a suggestion. `references/architecture-maturity.md` lists the arrows that must not exist, for auditing.

**Where a store sits.** A store is any holder of state that outlives a single screen. It is its own layer, above service and below the things that read it. Five rules:

- **App wide state only.** State used by exactly one feature stays inside that feature. Promote it to the store layer the day a second feature needs it, not before. A store is the most expensive place to put state, because everything can reach it.
- **A store may call services. It may never import a feature or a shared UI element.** That is the upward dependency the whole law exists to prevent.
- **A store holds state and the transitions between states.** Business rules belong in the shared logic layer, data access belongs in the service layer. If a store is doing arithmetic or validation, that work belongs one layer down where it can be tested without the store.
- **A server state cache is not a store.** A cache of remote data is a thin wrapper over the service layer. It may hold keys, staleness, and retry policy. It holds no business rules, and it never becomes the place features talk to each other through.
- **One store per domain, not one store for the app.** Law 2 applies to a store like any other file.

The folder name is yours. `store/`, `contexts/`, `providers/`, `state/` are all fine, and so is a library or the language's own built in mechanism. What matters is that there is one place for it and that the arrows point the same way. Pick a name on day one and keep it, because two names for one layer is worse than either name.

If your project has no store, the layer simply does not exist and everything above collapses one step. Create one only when two features share state, never because the diagram has a slot for it.

### 2. One responsibility per file
If you cannot describe a file's job in one sentence without the word `and`, split it. Name each file for its one job; a `helpers` or `utils` file inside a feature is the usual way this law gets broken.

### 3. New capability means new file
A new capability gets its own file, however small the addition. Appending it to an existing file is how a one-sentence file grows an `and`.

### 4. The entry point orchestrates, never implements
The screen, widget, route handler, or module entry composes things. It does not define them. No business rules, no raw client calls, no formatting. It calls the UI logic layer where one exists; where the stack has no UI logic layer between entry point and service, such as a backend route handler, it calls one service function and returns.

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
Every environment value, key name, base URL, and flag flows through one typed module that checks required values on startup. No other file reads a raw environment value. A misconfiguration should fail loudly at boot, not silently inside a feature at runtime.

### 12. One error shape
Services return a consistent result rather than throwing to the caller. The result type is declared once, in the type layer, and every service imports it. One place parses raw errors into clean messages. One place logs them. The UI decides how to show an error, never how to parse one. Where the language has a sum type, use it: a Dart `sealed` class, a TypeScript discriminated union, or a Python union of two dataclasses forces the caller to handle the failure branch, which a pair of optional fields does not.

### 13. Length is a signal, not a law
Roughly 400 lines for a file, roughly 120 for an orchestrator, is a prompt to check whether it took on a second responsibility. A long file that still does exactly one thing may stay long. `references/testing-and-exceptions.md` lists the cases where that is so.

### 14. Test by risk and by layer
Spend most effort on the shared logic and service layers, which are pure or near pure and hold the logic that matters. Spend least on screens and widgets, which change often and produce brittle tests. Cover those with a few end to end flows. `references/testing-and-exceptions.md` has the order to write them in.

### 15. Version control before the first line of code
Run `git init` before writing anything. Ignore secrets before the first commit. Commit a working state before restructuring. One logical move per commit.

This is local git. `git init` creates a hidden folder inside the project and nothing leaves the machine. No account, no remote, no upload. Adding a remote later is a separate choice you never have to make.

**Ignore secrets before the first commit, not after.** A commit is permanent. Deleting a key in a later commit does not remove it, it just adds a commit where the key is absent, and the old commit still holds it. Anyone who ever clones the repo gets the whole history.

```
# .gitignore, written before git add
.env
.env.*
!.env.example
*.pem
*.keystore
google-services.json
GoogleService-Info.plist
```

If a key is already sitting in a tracked source file, do these three things in this order before committing anything:

1. **Revoke and reissue the key at the provider.** Do this first. A key that has been on disk in a shared folder, a backup, or an editor's crash recovery is already out of your control, and rewriting history does not call it back.
2. **Move the value into an ignored env file** and read it through the config module, per Law 11. If the file holding it is already tracked by git, adding it to `.gitignore` does nothing on its own. Run `git rm --cached path/to/file` to untrack it first, otherwise git keeps committing the file it is now told to ignore.
3. **Then commit.** If the key was committed in an earlier session, treat step 1 as mandatory rather than as cleanup, because rewriting published history is hard and revoking is not.

Copying the project folder is not a substitute:

- A folder copy is one snapshot and an all or nothing undo. If a restructure is half right, you either keep the mistake or throw away the good work with it. A commit per move lets you drop only the bad one.
- The diff is how you catch a moved file with a stale import, a deleted line you did not mean to delete, or an edit that crept into a pure move. A folder copy shows you nothing.
- Every rule in the refactor section below assumes commits exist. Without them that advice does not work.

The four commands that cover the whole workflow:

```bash
git init                      # once, at the start of the project
git add -A                    # stage everything changed
git commit -m "message"       # save a labelled point you can return to
git diff                      # see exactly what changed since the last commit
```

Two more worth knowing when a move goes wrong:

```bash
git status                    # what is changed, staged, or untracked right now
git restore .                 # throw away uncommitted changes, back to the last commit
```

### A note on the examples in this skill

An example may name a domain. A rule may not. Where a reference file illustrates a point with a booking, a payment, or any other concrete product, that is there to show the shape of the answer, never to supply it. Derive the answer from the project in front of you. If a rule reads as though it only applies to one kind of product, you are reading an illustration as an instruction.

---

## Where Does This Code Go?

Language neutral. Translate the answer into folder names using the Step 0 table, or the matching reference file when you need the full tree.

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

Is it used by two or more features?
  ├─ Pure logic, formatting, error handling → shared logic layer
  ├─ Stateful UI logic → UI logic layer, the shared one
  └─ UI element → shared UI layer

Is it used by exactly one feature?
  └─ Stays inside that feature

Is it the entry point of a feature?
  └─ It is the orchestrator, keep it thin
```

---

## Starting a New Project

Do these seven things before feature one. The order matters more than the speed. Installing the boundary lint rule on day one takes ten minutes while the violation count is zero. Retrofitting it at forty files takes a weekend.

1. **Run the official scaffolder for the stack.** `flutter create`, `create-next-app`, `django-admin startproject`, `npm create vite`, `npx create-expo-app`. Accept the shape it produces. Step 0 rule one already says the framework wins, and on a new project that is the normal case, not an exception.
2. **`git init`, write `.gitignore`, then commit the untouched scaffold.** Before writing a line of your own code. That first commit is the clean floor you can always return to, and it keeps the scaffold separate from your work in every later diff. The ignore file comes before the commit, never after, for the reason in Law 15.
3. **Build the typed config module.** One file reads raw environment values and checks the required ones at startup. A missing key should stop the app at boot with a clear message, not surface as a strange bug inside a feature three weeks later.
4. **Define the one error shape and the single parser that produces it.** Decide now what a service returns when it fails. Retrofitting this means touching every service you ever wrote.
5. **Install the boundary lint rule and set it to fail the build.** The Step 0 table names the tool. Set it to error, not warning. There are no violations yet, so there is no cost, and the rule now guards every file that follows.
6. **Create the empty folder skeleton for the resolved layout.** The folders alone, each holding a `.gitkeep` so the commit can carry them. Empty folders make the layer model visible so you place the first file correctly instead of guessing and settling.
7. **Set up the test runner with one passing test.** A trivial test that asserts something obvious. The point is that the runner works and the command is known, so writing the second test is free.

Commit after each step. Done when six commits exist, one per step from the scaffold commit on, and the test runner, the boundary lint rule, and the app itself all run clean. Then feature one begins.

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
10. Confirm every import points one way and the lint rule agrees, then walk each new file past Laws 2 to 13.

`references/worked-example.md` runs one feature through these steps in this order, plus the config module a new project builds once, showing the actual files as you would write them.

---

## Refactoring a Project You Own

This is the mode with the most risk. The destination is described everywhere in this skill and the route is not. A restructure that half lands leaves a project worse than it was before it started.

**Preconditions.** Two, both required. A clean commit you can return to, and an app that actually runs right now. Without both, a restructure is a gamble rather than a change. If the project has no git, stop and do Law 15 first. If the app is already broken, fix it and commit before restructuring, otherwise you will never know which break came from where.

**Sweep for secrets before that first commit.** A project messy enough to need restructuring is a project likely to have a key pasted into a source file. Search for the obvious ones, `key`, `secret`, `token`, `password`, `Bearer`, before running `git add`. Law 15 has the order to follow when you find one.

**Move bottom up.** Types first, then pure helpers, then services, then whatever sits between services and features, then features.

```
1. type layer                     nothing depends downward, safest to move
2. shared logic layer             depends only on types
3. service layer                  depends on shared logic, types, config
4. store, shared UI, UI logic     if present, depend on services and below
5. feature layer                  depends on everything above, moves last
```

Each layer you move is only depended on by things above it, so the project keeps running the whole way through. Moving features first breaks everything at once and you lose the ability to tell a real error from a leftover import.

**One feature per pass.** Move it, run the app, commit. Then the next. A commit per feature means a bad move costs you one feature, not the day.

**Set the boundary lint rule to warning first.** Flip it to error only when the violation count reaches zero. Setting it to error on day one of a refactor turns the build red for a week, and a build that is always red gets disabled and never re-enabled. A tool with no warning level, such as `import-linter`, runs as a non-blocking CI step until the count is zero, then becomes blocking.

**Move and rename in separate commits.** Move the file first, commit, then rename, commit again. Git shows a pure move as a move and a pure rename as a rename. Combine them and the diff reads as a delete plus an unrelated new file, and a genuine mistake becomes impossible to tell apart from the rename.

**Know the stop signal.** Not everything needs moving. A file that is ugly but stable, and untouched for months, is not worth the risk of moving it. Refactor what is about to change anyway. The value of structure is in the code you are going to edit, not the code you are going to leave alone.

**Done when** the boundary lint rule runs at error with zero violations, the app runs, and every move sits in its own commit.

**Auditing rather than moving.** When the task is to review an existing large codebase rather than restructure it, `references/architecture-maturity.md` has the forbidden-arrow table, the secrets table, and a checklist to run it against.
