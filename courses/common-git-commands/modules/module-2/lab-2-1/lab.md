# Staging, Committing, and Viewing History

In this lab you will learn to work precisely with Git's three-area model — the working tree, the staging index, and the commit history. You will stage partial changes, craft meaningful commit messages, and use `git log` and `git diff` to navigate your project's past.

**Prerequisites:** Git configured with your identity (Lab 1-1 completed), and a terminal open inside a Git repository.

---

## Step 1 — The Three Areas: Working Tree, Index, and HEAD

Git thinks about your project in three distinct places at all times. Understanding these areas is the key to never being confused by Git's output.

```bash
# See all three areas at a glance
git status
```

| Area | What lives here |
|------|----------------|
| **Working tree** | Files on disk as you edit them right now |
| **Index (staging area)** | Snapshot prepared for the next commit |
| **HEAD** | The most recent commit on the current branch |

When you run `git add`, you move changes from the working tree into the index. When you run `git commit`, you move the index snapshot into a permanent commit and advance HEAD.

> **Why the staging area exists:** It lets you craft atomic commits — grouping only the changes that logically belong together — even when your working tree contains several unrelated edits.

---

## Step 2 — Check Repository Status

`git status` is the command you will run dozens of times per day. It tells you exactly what state each file is in.

```bash
# Create a couple of files to work with
echo "def greet(name):" > greet.py
echo "    return f'Hello, {name}!'" >> greet.py
echo "# TODO: add config" > config.txt

git status
```

Expected output:

```
On branch main
Untracked files:
  (use "git add <file>..." to include in what will be committed)
        config.txt
        greet.py

nothing added to commit but untracked files present (use "git add" to track)
```

Now stage only `greet.py` and check status again:

```bash
git add greet.py
git status
```

```
On branch main
Changes to be committed:
        new file:   greet.py

Untracked files:
        config.txt
```

You can see the two files are in different states simultaneously.

---

## Step 3 — Stage Changes Selectively with git add -p

When a file has multiple edits and you only want to commit some of them, `git add -p` (patch mode) lets you review and choose individual *hunks* (contiguous blocks of changes).

```bash
# Add more content to greet.py
echo "" >> greet.py
echo "def farewell(name):" >> greet.py
echo "    return f'Goodbye, {name}!'" >> greet.py
echo "" >> greet.py
echo "def shout(name):" >> greet.py
echo "    return greet(name).upper()" >> greet.py

git add -p greet.py
```

Git will show each hunk and ask what to do:

```
Stage this hunk [y,n,q,a,d,s,?]?
```

| Key | Action |
|-----|--------|
| `y` | Stage this hunk |
| `n` | Skip this hunk |
| `s` | Split into smaller hunks |
| `q` | Quit without staging remaining hunks |

> **Tip:** `git add -p` is one of Git's most powerful daily-use features. It keeps your commits focused and your history meaningful.

---

## Step 4 — Write Meaningful Commit Messages

A great commit message is a gift to your future self and your teammates. The conventional format is a short subject line followed by an optional body.

```bash
git commit -m "Add greet and farewell functions"
```

For a multi-line message, open your editor by omitting `-m`:

```bash
git commit
```

Your editor opens with a template. Write:

```
Add shout helper to greet module

greet() and farewell() return plain strings. shout() wraps greet()
and uppercases the result, useful for CLI banners.
```

> **Convention:** Subject line ≤ 72 characters, imperative mood (`Add`, `Fix`, `Remove`), blank line before the body, body explains *why* not *what*.

---

## Step 5 — Explore Commit History with git log

`git log` has many flags. Learn these four and you will cover most daily needs.

```bash
# Default: full details
git log

# Compact one-line summary
git log --oneline

# Visual branch graph
git log --oneline --graph --all

# Filter by author
git log --oneline --author="Ada"
```

To see what files changed in each commit:

```bash
git log --oneline --stat
```

```
a1b2c3d Add shout helper to greet module
 greet.py | 4 ++++
 1 file changed, 4 insertions(+)

d4e5f6a Add greet and farewell functions
 greet.py | 3 +++
 1 file changed, 3 insertions(+)
```

> **Tip:** `git log -n 5` limits output to the five most recent commits — handy on a project with thousands of commits.

---

## Step 6 — Compare Changes with git diff

`git diff` shows exactly *what* changed and *where*. The command behaves differently depending on which areas you compare.

```bash
# Difference between working tree and the index (unstaged changes)
echo "# config version 1" > config.txt
git diff

# Difference between the index and HEAD (staged but not yet committed)
git add config.txt
git diff --staged

# Difference between two commits
git diff HEAD~1 HEAD
```

Reading diff output:

```diff
diff --git a/config.txt b/config.txt
index e69de29..3b18e51 100644
--- a/config.txt
+++ b/config.txt
@@ -0,0 +1 @@
+# config version 1
```

Lines beginning with `+` were added; lines beginning with `-` were removed. The `@@` header shows the line numbers affected.

You now know how to inspect repository state at every level — working tree, index, and history. In the next lab you will learn how to *undo* changes at each of these stages without losing work.
