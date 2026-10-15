# Git and Secrets: The Reasons Behind Law 15

`SKILL.md` holds the rules of Law 15. This file holds the reasons, for when the user is new to git or asks why. Language neutral.

---

## Local git costs nothing

`git init` creates a hidden folder inside the project and nothing leaves the machine. No account, no remote, no upload. Adding a remote later is a separate choice the user never has to make.

---

## Why a commit is permanent

Deleting a key in a later commit does not remove it. It adds a commit where the key is absent, and the old commit still holds it. Anyone who ever clones the repo gets the whole history. That is why the ignore file is written before the first `git add`, never after.

---

## Why a leaked key is revoked first

A key that has sat on disk in a shared folder, a backup, or an editor's crash recovery is already out of your control. Rewriting history removes it from future clones and does nothing about copies that already exist. Revoking at the provider is the only step that makes the old value useless, so it comes first. If the key was committed in an earlier session, revoking is mandatory rather than cleanup, because rewriting published history is hard and revoking is not.

Adding a file to `.gitignore` does nothing on its own when git already tracks it. `git rm --cached path/to/file` untracks it while leaving the file on disk; without it, git keeps committing the file it is now told to ignore.

---

## Why a folder copy is no substitute for commits

- A folder copy is one snapshot and an all or nothing undo. If a restructure is half right, you either keep the mistake or throw away the good work with it. A commit per move lets you drop only the bad one.
- The diff is how you catch a moved file with a stale import, a deleted line you did not mean to delete, or an edit that crept into a pure move. A folder copy shows you nothing.
- Every rule in the refactor procedure assumes commits exist. Without them that advice does not work.

---

## The six commands a newcomer needs

```bash
git init                      # once, at the start of the project
git add -A                    # stage everything changed
git commit -m "message"       # save a labelled point you can return to
git diff                      # see exactly what changed since the last commit
git status                    # what is changed, staged, or untracked right now
git restore .                 # throw away uncommitted changes, back to the last commit
```
