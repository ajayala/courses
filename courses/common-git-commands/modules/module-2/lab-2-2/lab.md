# Undoing Changes and Recovering Work

In this lab you will learn to confidently reverse mistakes at every stage of the Git workflow — from discarding an accidental edit before staging, to recovering commits that seemed lost forever. Git almost never deletes history permanently; this lab shows you where it hides.

**Prerequisites:** Lab 2-1 completed; a repository with at least two commits and a `greet.py` file.

---

## Step 1 — Discard Unstaged Edits with git restore

You edited a file and immediately regret it. `git restore` resets the working-tree copy to whatever the index (or HEAD) has — without touching staged changes or history.

```bash
# Simulate an accidental edit
echo "WRONG CONTENT" >> greet.py

# Verify the damage
git diff greet.py

# Discard the edit — restore from HEAD
git restore greet.py

# Confirm the file is clean
git status
```

Expected result: `nothing to commit, working tree clean`.

> **Warning:** `git restore` is destructive for unstaged edits — the overwritten content is not recoverable via Git. Only use it when you are certain you want to throw away the changes.

---

## Step 2 — Unstage a File with git restore --staged

You ran `git add` on a file you did not mean to include in the next commit. `git restore --staged` moves it back to the working tree without losing your edits.

```bash
# Stage both files accidentally
git add greet.py config.txt

git status
# Both appear under "Changes to be committed"

# Unstage config.txt only
git restore --staged config.txt

git status
# greet.py is staged; config.txt is back to "Untracked"
```

Your edits to `config.txt` are still on disk — only the staging decision was reversed.

---

## Step 3 — Amend the Last Commit

You just committed but spotted a typo in the message, or forgot to include a file. `git commit --amend` lets you rewrite the most recent commit before anyone else has seen it.

```bash
# Make sure greet.py is committed and clean
git add greet.py
git commit -m "Add greet module with tyop in message"

# Fix the message (editor opens with the old message pre-filled)
git commit --amend -m "Add greet module"

# Verify the corrected history
git log --oneline -3
```

> **Important:** Amend rewrites the commit hash. Never amend a commit that has already been pushed to a shared remote — it will create a diverged history for everyone else.

---

## Step 4 — Revert a Commit Safely

`git revert` is the safe way to undo a commit that is already in shared history. Instead of rewriting history, it creates a *new* commit that applies the inverse of the target commit.

```bash
# Check the commit you want to undo
git log --oneline

# Revert the most recent commit (HEAD)
git revert HEAD --no-edit
```

Expected output:

```
[main f7a3b1c] Revert "Add greet module"
 1 file changed, 6 deletions(-)
```

```bash
git log --oneline
```

Both the original commit and its reversal are now in the history — anyone who pulled before the revert will have a clean fast-forward merge when they update.

---

## Step 5 — Reset to a Previous State

`git reset` moves the HEAD (and the branch pointer) to a different commit. It has three modes with different effects on the working tree and index.

```bash
git log --oneline
# e.g.
# f7a3b1c Revert "Add greet module"
# a1b2c3d Add greet module
# d4e5f6a Initial commit
```

| Mode | Index | Working tree | Use when |
|------|-------|-------------|---------|
| `--soft` | unchanged | unchanged | Keep changes staged, just move HEAD |
| `--mixed` (default) | reset | unchanged | Unstage changes but keep edits on disk |
| `--hard` | reset | reset | Completely discard all changes |

```bash
# Undo the revert commit, keeping its changes staged (safe)
git reset --soft HEAD~1

git status
# Changes to be committed: deletions from the reverted file

# If you only want to unstage (not discard)
git reset HEAD~1
```

> **Warning:** `--hard` discards working-tree edits permanently. Use it only on commits that have never been shared.

---

## Step 6 — Recover Lost Commits with git reflog

The reflog is Git's safety net. Every time HEAD moves — commits, resets, checkouts — Git logs it here. Commits that appear "lost" after a hard reset are almost always findable in the reflog.

```bash
git reflog
```

```
a1b2c3d HEAD@{0}: reset: moving to HEAD~1
f7a3b1c HEAD@{1}: revert: Revert "Add greet module"
a1b2c3d HEAD@{2}: commit: Add greet module
d4e5f6a HEAD@{3}: commit (initial): Initial commit
```

To recover a commit you thought was gone:

```bash
# Pick the hash of the "lost" commit from the reflog
git checkout -b recovery-branch f7a3b1c

# Or simply reset to it
git reset --hard f7a3b1c
```

> **Tip:** The reflog is local — it is not pushed to remote repositories. Entries expire after 90 days by default. Act quickly if you need to recover something old.

You now have a complete toolkit for undoing mistakes at every stage: before staging, after staging, after committing locally, and even after a hard reset. In the next module you will put these skills to work across multiple branches.
