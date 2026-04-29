# Timeline, Stash, and the GitHub Pull Request Flow

In this lab you will publish your feature branch to GitHub, use the Timeline view to browse file history, stash work-in-progress changes using the Source Control menu, and walk through the complete Pull Request lifecycle from inside VS Code using the GitHub Pull Requests extension.

**Prerequisites:** Lab 2-1 completed; `feature/add-health-check` branch with one clean commit. A GitHub account and the **GitHub Pull Requests** extension installed in VS Code (`ms-vscode.vscode-pull-request-github`).

---

## Step 1 — Publish the Branch to GitHub

If you have not already connected VS Code to GitHub:

1. Open the Command Palette (`Ctrl+Shift+P`) → **GitHub: Sign In**.
2. Follow the browser prompt to authenticate.

To publish your branch:

1. In the status bar, click the **Publish Branch** icon (cloud with an up arrow) that appears next to the branch name.
2. VS Code asks whether to make it public or private — choose **Public**.
3. The branch is pushed and the status bar icon changes to a sync icon.

> **Alternatively:** Open **… → Push** in the Source Control panel. If the branch has no upstream, VS Code will prompt you to publish it.

```bash
# Verify the remote branch exists
git branch -vv
```

```
* feature/add-health-check  a1b2c3d [origin/feature/add-health-check] Add health check endpoint
  main                       d4e5f6a [origin/main] Initial commit
```

---

## Step 2 — Explore File History with the Timeline View

The **Timeline** panel shows every commit that touched the currently open file — without leaving the editor.

1. Open `src/health.py` in the editor.
2. In the Explorer panel (`Ctrl+Shift+E`), scroll down to the **Timeline** section at the bottom.
3. Expand it — each entry is a commit that modified this file.

Click any entry to open a diff showing exactly what changed in that commit.

> **Tip:** The Timeline also shows **Local History** entries — VS Code's own autosave snapshots. These are separate from Git commits and can recover edits you never committed. Look for entries prefixed with a clock icon vs the Git branch icon.

To see the full commit history across all files, open the Command Palette and run **Git: View History**. This opens a read-only log panel.

---

## Step 3 — Stash Work in Progress

You are mid-edit on a new feature when you need to switch branches urgently. Stash saves your uncommitted changes to a temporary stack.

Make an incomplete change:

```python
# Add to the bottom of src/health.py — do NOT save a commit yet
def detailed_check() -> dict:
    # TODO: implement
    pass
```

Save the file. The Source Control panel badge shows **1 change**.

**Stash via the UI:**

1. Open **… → Stash → Stash (Include Untracked)**.
2. VS Code prompts for a stash message: `WIP: detailed health check stub`.
3. Press Enter.

The Source Control badge drops to **0** — your working tree is clean and you can safely switch branches.

To restore the stash later:

1. Open **… → Stash → Pop Stash…**
2. VS Code shows a list of saved stashes — select yours and press Enter.

Your `detailed_check` stub reappears in `health.py`.

> **Stash list tip:** Run **Git: View Stash** from the Command Palette to see all stashes. Each entry shows the branch it was created on and the message you gave it.

```bash
# Discard the stub — it was just for practice
git checkout -- src/health.py
```

---

## Step 4 — Create a Pull Request from VS Code

With the **GitHub Pull Requests** extension installed and the branch published:

1. Open the Command Palette → **GitHub Pull Requests: Create Pull Request**.
2. A side panel opens with fields:
   - **Title:** `Add health check endpoint`
   - **Base branch:** `main`
   - **Compare branch:** `feature/add-health-check` (pre-selected)
   - **Description:** paste a brief summary of the change.
3. Click **Create**.

The extension opens a PR view panel inside VS Code showing the PR number, reviewers, checks, and changed files.

> **PR description template:** To pre-fill the description box with a template, create `.github/pull_request_template.md` in your repository. The extension picks it up automatically.

---

## Step 5 — Review and Approve a PR

The GitHub Pull Requests extension lets you review code without leaving VS Code.

1. Open the **GitHub Pull Requests** panel in the Activity Bar (PR icon).
2. Expand **All** and click your open PR.
3. Click a changed file to open the diff editor — the same diff you saw in Lab 2-1, now with GitHub comment threads overlaid.

To add a review comment:
- Hover over a line in the diff → click the **+** bubble that appears → type your comment → click **Add single comment** or **Start review**.

To approve and merge:
1. Click the **…** next to the PR title → **Approve**.
2. Click **Merge Pull Request** → choose your merge strategy:

| Option | Equivalent | When to use |
|--------|-----------|-------------|
| Create a merge commit | `git merge --no-ff` | Preserve branch history |
| Squash and merge | Squash commits → `git merge` | Clean single commit per PR |
| Rebase and merge | `git rebase` → fast-forward | Linear history |

3. Click **Confirm Merge**. The branch is merged and the PR is closed.

---

## Step 6 — Sync and Clean Up After Merge

After the PR is merged on GitHub, update your local repository.

1. Switch to `main` via the status bar branch picker.
2. Click the **sync** icon (circular arrows) in the status bar — this runs `git pull` and downloads the merged commit.

```bash
git log --oneline -4
```

```
e5f6a7b (HEAD -> main, origin/main) Add health check endpoint
c3d4e5f Add default region config
d4e5f6a Initial commit
```

Delete the stale local branch:

1. Open **… → Branch → Delete Branch…**
2. Select `feature/add-health-check` → press Enter.

Delete the remote branch:

1. Open **… → Branch → Delete Remote Branch…**
2. Select `origin/feature/add-health-check` → press Enter.

```bash
git branch -vv
```

```
* main  e5f6a7b [origin/main] Add health check endpoint
```

Your local and remote branches are in sync and the merged feature branch is cleaned up. In the next module you will handle production emergencies — creating hotfixes and reverting broken deployments entirely through the VS Code UI.
