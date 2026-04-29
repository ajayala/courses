# Branches, Staging Hunks, and the Diff Editor

In this lab you will create and switch branches using the VS Code status bar, use the inline diff editor to stage individual lines rather than whole files, and keep your feature branch in sync with `main` — all through the VS Code UI. Precise staging is the difference between a clean commit history and a messy one.

**Prerequisites:** Lab 1-1 completed; `vscode-prod-app` open in VS Code with at least one commit and a `v1.0.0` tag.

---

## Step 1 — Create a Branch from the Status Bar

The branch name in the bottom-left status bar is a clickable button that opens a branch picker.

1. Click the branch name in the status bar (e.g. **main**).
2. A quick-pick menu appears at the top of the screen. Select **+ Create new branch…**
3. Type the branch name: `feature/add-health-check`
4. Press Enter. VS Code creates the branch and switches to it immediately.

The status bar now shows **feature/add-health-check**.

> **Alternatively:** Open the Command Palette (`Ctrl+Shift+P`) and run **Git: Create Branch…** — useful when you want to create without switching, or to create from a specific commit.

To switch between existing branches, click the status bar branch name and select from the list — no need to type `git switch`.

```bash
# Verify the branch was created and is active
git branch
```

```
* feature/add-health-check
  main
```

---

## Step 2 — Make Changes and Open the Diff Editor

Add a health check module to the project.

1. In the Explorer panel, right-click `src/` → **New File** → name it `health.py`.
2. Paste the following content:

```python
def check() -> dict:
    """Return application health status."""
    return {
        "status": "ok",
        "version": "1.0.0",
        "debug": True,
    }
```

3. Also open `src/app.py` and add an import line at the top:

```python
from health import check
```

In the Source Control panel you will now see two files under **Changes**: `health.py` (U — untracked) and `app.py` (M — modified).

Click either filename in the Source Control panel to open the **diff editor** — the left pane shows the previous version, the right pane shows your edits. Green highlights are additions, red highlights are deletions.

---

## Step 3 — Stage Individual Hunks (Partial Staging)

You want to commit `health.py` and the import line in `app.py`, but *not* the `"debug": True` line — that is temporary and should not go into production.

1. In the Source Control panel, click `health.py` to open its diff.
2. In the diff editor gutter (left edge of the right pane), you will see icons next to each changed line:
   - Click **Stage Change** (the `+` icon in the gutter) next to the lines you want to stage.
   - Or right-click a highlighted region → **Stage Selected Ranges**.

Stage all lines in `health.py` *except* `"debug": True,`.

3. Click `app.py` in the diff editor. Stage the `from health import check` line only.

> **Tip:** To stage an entire file without opening the diff, hover over the filename in **Changes** and click the **+** icon that appears. For partial staging the diff editor is required.

After selective staging, `health.py` and `app.py` should appear in both **Staged Changes** (the parts you staged) and **Changes** (the `debug` line you left out).

---

## Step 4 — Commit the Staged Changes

With only the appropriate changes staged:

1. Click the **Message** box at the top of the Source Control panel.
2. Type:
   ```
   Add health check endpoint
   ```
3. Press **Ctrl+Enter** to commit.

Now stage and commit the debug line as a separate, clearly labelled commit so it is easy to remove later:

1. Hover over `health.py` in **Changes** → click **+** to stage the remaining `debug` line.
2. Message:
   ```
   WIP: enable debug flag for local testing only
   ```
3. Commit.

```bash
git log --oneline
```

```
b2c3d4e WIP: enable debug flag for local testing only
a1b2c3d Add health check endpoint
<initial commit>
```

The two logical changes are now in separate commits — easy to drop the WIP commit before merging.

---

## Step 5 — Rebase onto Main from the More Actions Menu

A teammate has merged a change into `main` while you were working. Rebase your branch on top of it.

First, simulate the teammate's commit:

```bash
git switch main
echo "REGION=us-east-1" > config.env
git add config.env
git commit -m "Add default region config"
git switch feature/add-health-check
```

Now rebase via VS Code:

1. Open the Source Control panel **… (More Actions)** menu.
2. Select **Branch → Rebase Branch…**
3. In the quick-pick, choose **main**.

VS Code runs `git rebase main` behind the scenes. If there are no conflicts, the panel returns to a clean state. Your two feature commits now sit on top of the teammate's commit.

```bash
git log --oneline --graph
```

```
* b2c3d4e (HEAD -> feature/add-health-check) WIP: enable debug flag
* a1b2c3d Add health check endpoint
* c3d4e5f Add default region config   ← teammate commit
* d4e5f6a Initial commit
```

> **Conflict resolution in VS Code:** If the rebase hits a conflict, VS Code opens the conflict editor automatically, highlighting the conflicting sections with **Accept Current Change**, **Accept Incoming Change**, and **Accept Both Changes** buttons inline in the file.

---

## Step 6 — Drop the WIP Commit with Undo and Restage

Before requesting a review, drop the debug commit. VS Code's **Undo Last Commit** moves its changes back to the staging area without losing them.

1. Open **… → Undo Last Commit**.

The WIP commit disappears from history; its changes reappear in **Staged Changes**.

2. Hover over `health.py` in **Staged Changes** → click **−** (Unstage).
3. Right-click `health.py` in **Changes** → **Discard Changes** → confirm.

```bash
git log --oneline
```

```
a1b2c3d (HEAD -> feature/add-health-check) Add health check endpoint
c3d4e5f Add default region config
d4e5f6a Initial commit
```

The branch now has a single, clean commit with no debug code — ready for review. In the next lab you will publish this branch, use the Timeline view to explore file history, and walk through the Pull Request flow from inside VS Code.
