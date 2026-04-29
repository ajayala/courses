# Setting Up Git and Your First Repository

In this lab you will install and configure Git on your machine, initialise your first repository, and make your very first commit. By the end you will understand the basic lifecycle of a file in Git and be able to view your project's history with `git log`.

**Prerequisites:** A terminal (bash, zsh, or PowerShell) and Git installed (`git --version` should print a version number).

---

## Step 1 — Configure Your Identity

Before Git will let you commit anything, it needs to know who you are. This identity is embedded in every commit you make, so other contributors know who authored each change.

```bash
git config --global user.name "Ada Lovelace"
git config --global user.email "ada@example.com"
```

You only need to do this once per machine. The `--global` flag writes the values to `~/.gitconfig`. You can verify the result:

```bash
git config --list --global
```

> **Tip:** Omit `--global` to set identity for a single repository only — useful when you contribute to work and personal projects from the same machine.

---

## Step 2 — Initialise a Repository

A Git repository is just a directory with a hidden `.git` folder inside it. The `git init` command creates that folder and sets up everything Git needs to start tracking history.

```bash
mkdir my-git-project
cd my-git-project
git init
```

Expected output:

```
Initialized empty Git repository in /home/user/my-git-project/.git/
```

> **Why a hidden folder?** Keeping Git's internals in `.git/` means your working directory stays clean — only your project files are visible at the top level.

---

## Step 3 — Explore the .git Directory

Take a moment to look at what `git init` created. Understanding the structure removes the mystery from Git's behaviour.

```bash
ls -la .git/
```

| Entry | Purpose |
|-------|---------|
| `HEAD` | Points to the currently checked-out branch or commit |
| `config` | Per-repository configuration |
| `objects/` | Stores every version of every file, compressed |
| `refs/` | Named pointers (branches, tags) into the object store |

```bash
cat .git/HEAD
```

You will see `ref: refs/heads/main` (or `master` on older Git versions), meaning HEAD currently points to the `main` branch — which does not yet exist because there are no commits.

---

## Step 4 — Create Your First File

Git tracks files you tell it about. Start by creating a simple text file in your project.

```bash
echo "# My Git Project" > README.md
echo "Learning git one command at a time." >> README.md
```

Check Git's view of the world:

```bash
git status
```

Expected output:

```
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        README.md

nothing added to commit but untracked files present
```

Git sees the file but is not yet tracking it — it is *untracked*.

---

## Step 5 — Stage the File

Staging (the *index* or *staging area*) is a preparation zone between your working files and the permanent history. You choose exactly what goes into the next commit by staging it first.

```bash
git add README.md
```

Run `git status` again:

```bash
git status
```

```
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   README.md
```

The file has moved from *untracked* to *staged*. Nothing is permanent yet — you can still remove it from the staging area with `git restore --staged README.md`.

---

## Step 6 — Make Your First Commit

A commit permanently records the staged snapshot into the repository's history. Every commit gets a unique SHA-1 hash, an author, a timestamp, and a message.

```bash
git commit -m "Initial commit: add README"
```

Expected output:

```
[main (root-commit) a3f9c12] Initial commit: add README
 1 file changed, 2 insertions(+)
 create mode 100644 README.md
```

> **Write good messages:** A commit message should complete the sentence *"If applied, this commit will…"*. Short, imperative phrases like `Add login form` or `Fix null pointer in parser` are conventional.

---

## Step 7 — View Your Commit History

`git log` is your window into a project's past. Even with just one commit, it is worth exploring its output.

```bash
git log
```

```
commit a3f9c12d4e8b1f5a9c2d7e3b6f0a4c8d2e1b5f9a
Author: Ada Lovelace <ada@example.com>
Date:   Tue Apr 29 10:00:00 2026 +0000

    Initial commit: add README
```

For a more compact view, use:

```bash
git log --oneline
```

```
a3f9c12 Initial commit: add README
```

You have now set up Git, initialised a repository, staged a file, made your first commit, and inspected the history. In the next lab you will go deeper into staging — learning how to stage partial changes, write richer commit messages, and compare differences between versions.
