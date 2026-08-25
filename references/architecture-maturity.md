# Architecture Maturity: Auditing a Large Codebase

`SKILL.md` holds the Laws. This file is the audit companion for a large or commercial codebase: the dependency arrows that must not exist, where each kind of secret may live, and a checklist to run the repo against. Language neutral. Translate layer names into folders with the matching language reference.

What decides whether a big codebase stays simple is not where files sit, it is which files are allowed to depend on which. Everything here is about checking that.

---

## The Arrows That Must Not Exist

The allowed directions are the diagram under Law 1 in `SKILL.md`, and that is the only authoritative copy. This is the same diagram read the other way: per layer, every arrow it may not draw, with the damage each one does. If an import follows one of them, the thing being reached for belongs lower down; the rule does not need an exception.

| From | May not import | Why |
|---|---|---|
| feature | a sibling feature, config | Law 5. The one rule that prevents spaghetti at scale. Neither feature can be understood, tested, or removed alone. |
| UI logic | feature, shared UI, config | Upward. A hook or controller that renders, or knows which feature owns it, is no longer a bridge. |
| shared UI | feature, UI logic, service, config | Reusable UI now knows business logic, or fetches and slips past the validation gate. It takes data as a prop or reads a store; it never fetches. |
| store | feature, UI logic, shared UI, config | Upward, and it turns the store into a hidden channel between features. |
| service | feature, UI logic, shared UI, store | Upward. The stable layer would depend on the volatile one. |
| shared logic | anything but the type layer | A helper that reaches a database, a store, or a screen is not a helper, and can no longer be tested alone. |
| config | anything | Config reads the environment and nothing else. A dependency here makes boot order matter. |
| type | anything | The bottom of the stack. |
| any file | a raw environment value | Config must come from one place. Law 11. |

The machine checks this, not a reviewer. The Step 0 table in `SKILL.md` names the tool per language; the TypeScript and Python references show it configured, and the Dart reference says what to do while its tooling catches up.

---

## Where Each Kind of Secret May Live

Sort every key by who may see it. Anything granting real power must never reach a client bundle, git history, or a public app.

| Secret type | Where it lives | Where it must never appear |
|---|---|---|
| Public client key, designed to be shared | Client app, with strict access rules behind it | Nowhere unsafe, but it protects nothing alone |
| Admin or service key that bypasses access rules | Server or serverless function only | Client bundle, git, logs |
| Third party API key, such as an AI or email provider | Behind a server function the client calls | Client bundle, network calls visible to the user |

The common failure: a client app calls a paid third party API directly, so the key ships in the bundle. Someone lifts it and spends your money. The fix is always the same. The client calls your server, your server holds the key.

---

## Maturity Checklist

Audit a codebase against this. Each unchecked box is a place it gets harder to maintain as it grows.

```
Layout
[ ] The layout matches the framework or language convention, not a copied one
[ ] The layout is consistent across the whole repo

Dependencies
[ ] No service, shared helper, or shared UI element imports from a feature
[ ] No shared helper imports from a service
[ ] No feature imports from a sibling feature
[ ] No two features talk to each other through a store
[ ] A lint or CI rule fails the build on any of the above

Isolation
[ ] Every feature exposes a single public door, internals are private
[ ] Outside code imports the door, never a file behind it
[ ] The door holds exports only, no logic

Security
[ ] Powerful keys exist only server side, never in a client bundle or git
[ ] No client app calls a paid third party API directly
[ ] Identity and permissions are decided server side, not from client input
[ ] Every external input is validated at the service boundary
[ ] Public paths with no login are rate limited

Foundations
[ ] One typed config module, no scattered env reads, fails loud at startup
[ ] One error shape declared in the type layer, one central parser and logger
[ ] No escape hatch types in the lower layers, shared types live in their own layer
[ ] Strict type checking is on for the service, shared logic, and type layers
```

Maintainability at scale is not about having more rules. It is about dependencies pointing one way, features sealed behind one door, security designed in, and a machine checking all three.
