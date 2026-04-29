# Hotfixes — Emergency Changes Using the VS Code UI

In this lab you will respond to a production incident entirely within VS Code: cut a hotfix branch from the last good release tag, apply the minimal targeted fix using the diff editor to verify scope, commit and tag the patch release, and merge it back to both `main` and `develop`. Speed and precision are critical — the diff editor ensures you touch nothing you did not intend to.

**Prerequisites:** Lab 2-2 completed; `vscode-prod-app` on `main` with a `v1.0.0` tag. A `develop` branch (create it now if it does not exist: status bar → **+ Create new branch…** → `develop`, from `main`).

---

## Step 1 — Identify the Production Tag to Branch From

Production is running `v1.0.0`. An incident has been reported: `src/health.py` is returning a hardcoded `"status": "ok"` even when the application is unhealthy.

Before cutting the hotfix branch, confirm which commit the tag points to:

1. Open the Command Palette (`Ctrl+Shift+P`) → **Git: View History**.
2. Scroll through the log to find the commit with the `v1.0.0` tag annotation.

Or use the integrated terminal for a quick check:

```bash
git show v1.0.0 --stat
```

```
Tag: v1.0.0
...
commit d4e5f6a
    Initial commit: add README, VERSION, and app entry point
```

> **Why branch from the tag, not from main?** If `main` has received commits since `v1.0.0` was deployed (from other PRs), those commits are not in production. Branching from the tag gives you a clean snapshot of exactly what is running.

---

## Step 2 — Create the Hotfix Branch from the Tag

VS Code's branch picker lets you branch from a tag directly:

1. Click the branch name in the status bar.
2. Select **+ Create new branch from…**
3. In the first prompt, type: `hotfix/1.0.1`
4. In the second prompt (base), type `v1.0.0` and select the tag from the list.

The status bar now shows **hotfix/1.0.1** and your working tree is at the exact state of the `v1.0.0` release.

```bash
git log --oneline -3
```

```
d4e5f6a (HEAD -> hotfix/1.0.1, tag: v1.0.0, main) Initial commit
```

---

## Step 3 — Apply the Minimal Fix Using the Diff Editor

Open `src/health.py`. Make only the targeted change — add a real liveness check:

```python
import os

def check() -> dict:
    """Return application health status."""
    healthy = os.getenv("APP_READY", "false") == "true"
    return {
        "status": "ok" if healthy else "degraded",
        "version": "1.0.0",
    }
```

Before staging, use the diff editor to verify *exactly* what changed:

1. In the Source Control panel, click `src/health.py` to open the diff editor.
2. Scan every changed line. The diff should show:
   - `+` `import os` — added
   - `+` `healthy = os.getenv(...)` — added
   - `~` `"status": "ok"` → `"status": "ok" if healthy else "degraded"` — modified
   - `-` `"debug": True,` — removed (if it was present)

If you see any changes beyond these lines, investigate before staging. A hotfix must be surgical.

> **Rule:** If the diff shows more than the minimum required change, discard the extras (`right-click → Discard Changes`) before committing. Every unreviewed line is a risk during an incident.

Stage the file: hover over `src/health.py` in **Changes** → click **+**.

---

## Step 4 — Commit, Tag, and Push the Hotfix

Write a commit message that references the incident:

```
Fix: health check returns degraded status when APP_READY is unset (INC-910)
```

Commit with **Ctrl+Enter**.

Now tag the patch release:

1. Command Palette → **Git: Create Tag**
2. Tag name: `v1.0.1`
3. Tag message: `Hotfix v1.0.1 — health check degraded status (INC-910)`

Push both the branch and the tag:

1. **… → Push** to push the `hotfix/1.0.1` branch.
2. The built-in Git UI does not push tags automatically. Use the integrated terminal:

```bash
git push origin v1.0.1
```

```bash
git tag -l
```

```
v1.0.0
v1.0.1
```

---

## Step 5 — Merge the Hotfix into Main

1. Switch to `main` via the status bar.
2. Click the sync icon to pull the latest `main`.
3. Open **… → Branch → Merge Branch…**
4. Select `hotfix/1.0.1` from the list.

VS Code merges with `--no-ff` by default when there is a diverged history, creating a merge commit.

Verify the result:

```bash
git log --oneline --graph -5
```

```
*   f1a2b3c (HEAD -> main) Merge branch 'hotfix/1.0.1'
|\
| * e2b3c4d (tag: v1.0.1, hotfix/1.0.1) Fix: health check returns degraded status
|/
* d4e5f6a (tag: v1.0.0) Initial commit
```

Push `main`:

1. Click the sync icon in the status bar (or **… → Push**).

Your CD pipeline should now pick up the new commit on `main` and deploy the hotfix.

---

## Step 6 — Back-Merge the Hotfix into Develop

Without this step, the next release from `develop` will re-introduce the bug.

1. Switch to `develop` via the status bar.
2. **… → Branch → Merge Branch…** → select `hotfix/1.0.1`.
3. If VS Code reports a conflict, the conflict editor opens automatically — resolve it, then click **Accept Merge**.

```bash
git log --oneline develop -4
```

```
g3h4i5j (HEAD -> develop) Merge branch 'hotfix/1.0.1' into develop
e2b3c4d (tag: v1.0.1, hotfix/1.0.1) Fix: health check returns degraded status
d4e5f6a (tag: v1.0.0) Initial commit
```

Clean up the hotfix branch:

1. **… → Branch → Delete Branch…** → select `hotfix/1.0.1`.
2. **… → Branch → Delete Remote Branch…** → select `origin/hotfix/1.0.1`.

The fix is now in both `main` (deployed) and `develop` (protected for future releases). In the final lab you will handle the harder scenario: the hotfix itself caused a problem and you need to revert production using the VS Code UI.
