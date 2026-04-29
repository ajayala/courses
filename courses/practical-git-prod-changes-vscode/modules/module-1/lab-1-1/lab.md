# Source Control Panel and Your First Commit

In this lab you will set up a production-ready project folder in VS Code, initialise a Git repository entirely through the UI, and make your first commit — all without opening a terminal. You will also configure the key VS Code Git settings that make the Source Control panel behave consistently across your team.

**Prerequisites:** VS Code installed (version 1.85 or later). Git installed and on your PATH (`git --version` in a terminal should return a version number). No terminal knowledge required for this lab.

---

## Step 1 — Open the Source Control Panel

The Source Control panel is VS Code's built-in Git UI. Every Git operation in this course begins here.

```
Keyboard shortcut: Ctrl+Shift+G  (macOS: ⌘+Shift+G)
```

Alternatively, click the **branch icon** in the Activity Bar on the left — it looks like a Y-shaped fork.

> **What you see:** If no repository is open, the panel shows two buttons: **Open Folder** and **Clone Repository**. If a repository is open but has no changes, the panel shows a badge of zero and an empty list.

Configure VS Code to make Git behaviour explicit and predictable. Open the Command Palette (`Ctrl+Shift+P`) and type **Preferences: Open User Settings (JSON)**, then add:

```json
{
  "git.enableSmartCommit": false,
  "git.confirmSync": false,
  "git.autofetch": true,
  "git.fetchOnPull": true,
  "editor.formatOnSave": true
}
```

`enableSmartCommit: false` prevents VS Code from accidentally committing all changes when the staging area is empty — a common surprise for new users.

---

## Step 2 — Initialise a Repository

1. Open a new empty folder: **File → Open Folder** → create a folder called `vscode-prod-app` and open it.
2. In the Source Control panel, click **Initialise Repository**.

VS Code runs `git init` behind the scenes. The panel now shows **No changes** and the status bar at the bottom shows the current branch name (`main` or `master`).

Now create the initial project files. Open the integrated terminal (`Ctrl+`` ` ``) just for file creation:

```bash
echo "version: 1.0.0" > VERSION
echo "# VS Code Prod App" > README.md
mkdir src
echo "# entry point" > src/app.py
```

Return to the Source Control panel (`Ctrl+Shift+G`). You will see three files listed under **Changes** with a **U** badge (Untracked).

> **File state badges in the Source Control panel:**
>
> | Badge | Meaning |
> |-------|---------|
> | U | Untracked — Git has never seen this file |
> | M | Modified — tracked file with unstaged edits |
> | A | Added — staged new file |
> | D | Deleted |
> | C | Conflict |

---

## Step 3 — Stage Files

Staging in the UI works by hovering over a file in the **Changes** section and clicking the **+** (Stage Changes) icon that appears to the right.

Stage all three files:

1. Hover over `README.md` → click **+**
2. Hover over `VERSION` → click **+**
3. Hover over `src/app.py` → click **+**

All three files move from the **Changes** section to the **Staged Changes** section with an **A** badge.

> **Tip:** To stage everything at once, hover over the **Changes** heading and click the **+** that appears next to it. To unstage a file, hover over it in Staged Changes and click the **−** icon.

To stage only *part* of a file's changes, click the file name to open the diff editor — you will practise this in Lab 2-1.

---

## Step 4 — Write a Commit Message and Commit

At the top of the Source Control panel is a text box labelled **Message (press Ctrl+Enter to commit)**. This is where you type your commit message.

```
Initial commit: add README, VERSION, and app entry point
```

Press **Ctrl+Enter** (macOS: **⌘+Enter**) to commit, or click the blue **Commit** button.

> **Multi-line messages:** Click the expand icon (⤢) to the right of the message box to open a full editor for longer commit messages with a subject line and body.

Verify the commit was created by checking the status bar — the branch name should now appear without any pending-changes indicator, and the Source Control panel badge shows **0**.

```bash
# Verification (terminal)
git log --oneline -3
```

```
a1b2c3d Initial commit: add README, VERSION, and app entry point
```

---

## Step 5 — Tag the Release from the Command Palette

VS Code exposes Git tag creation through the Command Palette — there is no dedicated button in the panel UI.

1. Open the Command Palette: `Ctrl+Shift+P`
2. Type **Git: Create Tag** and press Enter.
3. Enter the tag name: `v1.0.0`
4. Enter the tag message: `Release v1.0.0 — initial production deploy`

> **Annotated vs lightweight tags:** VS Code's **Git: Create Tag** command creates an annotated tag (it prompts for a message), which is correct for release tags. Lightweight tags can only be created from the terminal.

Verify the tag:

```bash
git tag -l
```

```
v1.0.0
```

---

## Step 6 — Explore the Source Control Panel Layout

Before moving on, take a moment to familiarise yourself with each section of the panel.

```
SOURCE CONTROL
├── Message box          ← commit message input
├── Commit button        ← commits staged changes
├── … (More Actions)    ← pull, push, fetch, stash, rebase, etc.
│
├── STAGED CHANGES       ← files ready to commit
└── CHANGES              ← modified/untracked files not yet staged
```

Click the **…** (More Actions) button at the top of the Source Control panel. This menu exposes operations that have no dedicated button:

| Menu item | Equivalent terminal command |
|-----------|---------------------------|
| Pull | `git pull` |
| Push | `git push` |
| Fetch | `git fetch` |
| Stash | `git stash` |
| Rebase Branch… | `git rebase <branch>` |
| Undo Last Commit | `git reset --soft HEAD~1` |

You have initialised a repository, staged files, committed, and tagged a release — all without leaving VS Code. In the next lab you will create branches from the status bar, use the diff editor to stage individual lines, and keep your feature branch up to date with a one-click rebase.
