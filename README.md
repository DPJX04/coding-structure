# coding-structure

Instructions any coding agent can follow to keep a codebase simple to read and cleanly organised.

It does two things first:

1. **Simple code.** Each function does one job, has a name that says what it does, carries a one-line comment, and stays short and flat.
2. **Every file in its layer folder.** No file gets dumped in the source root or in whichever folder is nearest. A flat folder of mixed files is treated as the problem to fix, never as a layout to copy.

Around those two rules sit a layered architecture (16 numbered Laws), folder layouts for TypeScript/JavaScript, Python, and Dart/Flutter, and step-by-step procedures for the four kinds of request an agent gets.

## Files

| File | What it holds | When an agent reads it |
|---|---|---|
| `SKILL.md` | The entry point: layout rules, project sizing, the four procedures, and the Laws | Always, first |
| `references/lang-typescript.md` | TypeScript, JavaScript, React, React Native, and Node layouts, lint config, framework overrides | Before scaffolding or adding a first feature in that stack |
| `references/lang-python.md` | Python packaging, layout, `import-linter` config, Django and FastAPI overrides | Same, for Python |
| `references/lang-dart.md` | Flutter and Dart layout, widget rules, analyzer config | Same, for Dart |
| `references/worked-example.md` | One feature built end to end, file by file | When adding a feature |
| `references/testing-and-exceptions.md` | What to test first, and when a rule may be broken | When writing tests or bending a rule |
| `references/architecture-maturity.md` | Forbidden imports, where secrets may live, an audit checklist | When reviewing a codebase without changing it |
| `references/git-and-secrets.md` | The reasons behind the git and secrets rules | When the user is new to git or asks why |

`SKILL.md` says when to open each reference, so an agent only needs to be pointed at `SKILL.md`.

## How an agent uses it

`SKILL.md` runs in this order:

1. **Step 0, resolve the layout.** Use the framework's layout if there is one. Otherwise use the project's existing layout if it is already layered, and the language's folder tree if not.
2. **Step 1, size the project** as a script, a small app, or a full app. Build only the structure that size needs.
3. **Step 2, pick the mode** and follow its procedure:
   - **Changing Existing Code:** fixes and small edits. The diff stays small and nothing gets restructured unasked.
   - **Adding a New Feature:** built bottom up, layer by layer.
   - **Starting a New Project:** git, config, error shape, tests, then the first feature.
   - **Refactoring a Project You Own:** baseline, secret sweep, a file map approved by the user, then moves from the bottom layer up.

The agent stops and waits for the user at two points only: when it finds a leaked key, and before a refactor moves any file.

## Install

The skill is plain Markdown with no dependencies. Any agent that can read a file can use it.

**Claude Code.** Copy the folder to `~/.claude/skills/coding-structure/` for all projects, or to `.claude/skills/coding-structure/` inside one project. Claude loads it when a request matches the `description` in `SKILL.md`.

**Claude apps (claude.ai, desktop).** Zip the folder with `SKILL.md` at the top level and upload it under your skill settings. To update it, upload the new zip in place of the old one.

**Any other agent** (Codex, Cursor, Gemini CLI, GitHub Copilot, Aider, and others). Put the folder in the repo or somewhere the agent can read, then add one pointer to the instruction file that agent loads:

| Agent | Instruction file |
|---|---|
| OpenAI Codex, and others that read it | `AGENTS.md` |
| Cursor | `.cursor/rules/coding-structure.mdc`, with `alwaysApply: true` in its frontmatter |
| Gemini CLI | `GEMINI.md` |
| GitHub Copilot | `.github/copilot-instructions.md` |

Pointer text to paste into that file, with the path adjusted to where you placed the folder:

```
Before writing, changing, or moving any code file, read
coding-structure/SKILL.md and follow it. It decides where each file goes,
how simple each function must be, and when to stop and ask me.
```

Keep the pointer short. It is loaded on every request, and `SKILL.md` carries the detail.

## How it was tested

The skill was run three times against three fixed tasks, each time by a fresh agent that saw only the skill and the task:

- **A flat, messy Node project** with no git, vague names, dead code, and a hardcoded live API key, given the request "clean it up".
- **A one-line bug fix** in an already layered Python project.
- **A new Python command-line app**, built from an empty folder.

Every result was checked by hand: the app's output, the tests, git history for leaked keys, file placement, and function comments and length. Each round's gaps were fixed in the skill before the next run. In the final round all three tasks passed, and the refactor of the flat project took 13 commits, down from 31 in the second round.

Not yet tested: Dart/Flutter projects, framework projects such as Next.js or Django, and whether an agent picks the skill up from its description alone without being pointed at it.

## Changing the skill

- Every rule lives in exactly one place. The Laws in `SKILL.md` are the source, and the references cite them by number.
- After a change, rerun the three tasks above with a fresh agent and check the results by hand. A rule that reads well can still steer an agent wrong.
