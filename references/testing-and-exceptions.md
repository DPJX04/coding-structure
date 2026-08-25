# Exceptions and Testing

Two things a maturing codebase needs that the base rules do not handle: when to break a rule on purpose, and how to test a large codebase when you cannot test every detail. Language neutral, with per language notes where the mechanics differ.

---

## Part 1: Exceptions and Escape Hatches

The Laws are defaults with teeth, but the point of the structure is consistency of intent, not blind obedience to a row in a table. A rule exists to stop a real problem. When breaking it does not cause that problem, breaking it is correct.

Three Laws admit no exception. Laws 8, 9, and 10 hold always, because trusting the client, shipping a secret, or skipping boundary validation causes harm that a refactor cannot undo.

### When a file may run long

Law 13: length is a signal, not a law. A long file that still does exactly one thing is not the problem. Leave it.

| Case | Why it is fine |
|---|---|
| Generated types or serialization code | Output of a tool, never hand edited, splitting adds no value |
| Large config or constant maps | One responsibility, just many data rows |
| A single state machine with many states | Splitting states across files hides the logic |
| A schema or validator with many fields | One concern, scales with the data shape, not with complexity |

The test is Law 2. A file that passes it may stay long.

Flutter note: a deeply nested widget tree is not an exception. If a build method runs long, extract widget classes. See `lang-dart.md`.

### When code may live outside a feature

Some code belongs to no feature and is not reusable enough for the shared layers either. Put it where a stranger would look first.

- A cross cutting concern such as an error boundary, theme provider, or root router goes in the app root or the shared UI layer, not forced into a feature.
- A one off script for seeding or migrating goes in a top level `scripts/` folder, outside the source root entirely.
- Platform glue, such as a native module wrapper, goes in the config or a dedicated platform folder, not in a feature.

### The door file

A barrel or `__init__.py` that holds only exports is one responsibility and never an exception. The moment it holds logic it has broken Law 6 for real.

### The principle behind every exception

When you break a rule, the replacement must still answer the question the rule was protecting: where does a stranger look to find this, and does this file do one clear thing? If both answers stay clean, the exception is sound. Write a one line comment at the top saying why it breaks the norm, so the next person does not try to fix something that is not broken.

---

## Part 2: Testing a Large Codebase

The mistake on a large codebase is trying to test every detail. You cannot, and you should not. Test by risk and by layer. The architecture already sorts your code into layers that are easy to test and layers that are not, so let the structure tell you where to spend effort.

### Test value, not coverage

A coverage percentage is a vanity number. What protects a product is testing the paths where a bug costs money, breaks trust, or corrupts data. A bug in a date label is annoying. A double booking during a client demo loses a customer.

### Match the test type to the layer

| Layer | Test type | Effort | Why |
|---|---|---|---|
| Shared logic | Unit tests | Cheap, high value | Pure functions, no mocks, fast to write, catch real logic bugs |
| Service or repository | Integration tests | Medium, highest value | Where data correctness lives, test against a real test database |
| UI logic, hooks or controllers | State tests | Medium | Loading and error states, the bridge between UI and data |
| Screens, components, widgets | Light smoke tests only | Low priority | UI changes often, brittle tests slow you down, cover with end to end flows |

Where tests live is per language: the Step 0 table in `SKILL.md` has the answer, and each language reference shows the tree.

### The order to write tests

Do not start at the top of the file list. Start at the highest risk path and work down. Step one has no fixed answer, you derive it from the product in front of you.

1. **The one rule that corrupts data if broken.** Every product has one. Identify it before writing a single test. Booking software: overlapping reservations. Payments: charging twice. Inventory: overselling stock. Payroll: the wrong rate applied. Messaging: delivering to the wrong recipient. Whatever yours is, test it first and test its edges hardest, including the boundary case that looks fine and is not. For overlapping reservations that boundary is one booking ending at the exact moment another starts.
2. **Write operations in the service layer.** Anything that inserts or updates. A silent failure here corrupts the source of truth. Test against a throwaway database rather than mocks, so schema and constraint errors surface.
3. **Auth and session.** Token refresh, expired sessions, device pairing. A break here locks people out.
4. **Any state machine.** Pure transitions are fast and cheap to test, and a wrong state shows a wrong status to a user.
5. **Date and time logic.** Timezone handling is a classic source of off by one hour bugs. Test around midnight and across day boundaries.
6. **Everything else, by how often it touches money or data.**

### The thin layer that catches most bugs

If you only have time for a small test suite before launch:

- Every shared helper carrying the math your one critical rule depends on, plus all date and time logic.
- Every write function in the service layer, against a real test database.
- One end to end test per core flow. Three tests that prove the product works at all.

That set finishes in days, not weeks, and it guards the paths that matter.

### Logic you cannot test in isolation

When behaviour depends on accumulated context, such as an agent holding state across a conversation, pull the state into a plain object that ordinary code builds and mutates, and unit test that object's transitions. This is the general move: when something is hard to test, extract the part that is not.
