# Reverting and Rolling Back via the Source Control Panel

In this lab you will identify a bad commit using the Timeline view and VS Code's Git history, revert it through the Command Palette, use the diff editor to verify the rollback is correct before pushing, and ship a proper fix-forward commit through the UI. You will leave with a clean history, a new patch tag, and the confidence to respond to production incidents without ever touching the terminal.

**Prerequisites:** Lab 3-1 completed; `vscode-prod-app` on `main` with `v1.0.1` tagged. The `develop` branch exists and has the hotfix back-merged.

---

## Step 1 — Simulate a Bad Deployment

A new change has just been merged to `main` — but it introduced a regression: the health check now crashes on startup.

Add the bad commit:

```bash
# In the integrated terminal
git switch main
cat >> src/health.py << 'EOF'


def startup_probe() -> bool:
    return 1 / 0  # BUG: division by zero
EOF
git add src/health.py
git commit -m "Add startup probe (INC-911 investigation)"
```

Push to main (the equivalent of a broken deployment going out):

```bash
git push origin main
```

Alerts fire. You need to roll back.

---

## Step 2 — Find the Bad Commit with the Timeline View

The fastest way to identify *what* changed is the Timeline view — it shows per-file history without leaving the editor.

1. Open `src/health.py` in the editor.
2. In the Explorer panel, scroll down to **Timeline** and expand it.
3. Click the most recent Git entry — the diff editor opens showing exactly what that commit added.

You immediately see the `1 / 0` line. Confirm the commit hash shown in the Timeline entry — you will need it in the next step.

> **For multi-file investigations:** Open the Command Palette → **Git: View History** to see the full project log. Click any commit entry to expand the list of changed files, then click a file to see its diff.

You can also use the **Source Control Graph** (VS Code 1.90+):

1. Open the Source Control panel.
2. Click the **Graph** icon at the top-right of the panel.
3. Each node in the graph is a commit — hover to see the message and author, click to open the diff.

---

## Step 3 — Revert the Commit from the Command Palette

VS Code exposes `git revert` through the Command Palette — it creates a new commit that inverts the target, leaving history intact for audit purposes.

1. Open the Command Palette: `Ctrl+Shift+P`
2. Type **Git: Revert Commit…** and press Enter.
3. A quick-pick list shows recent commits. Select the bad commit (`Add startup probe (INC-911 investigation)`).
4. VS Code runs `git revert --no-edit` and creates the revert commit automatically.

The Source Control panel shows **0 changes** — the revert committed cleanly.

```bash
git log --oneline -4
```

```
h1i2j3k (HEAD -> main) Revert "Add startup probe (INC-911 investigation)"
g2j3k4l Add startup probe (INC-911 investigation)
f1a2b3c Merge branch 'hotfix/1.0.1'
e2b3c4d (tag: v1.0.1) Fix: health check returns degraded status
```

> **If VS Code reports a conflict during revert:** The conflict editor opens automatically. Resolve by accepting the version that removes the bad lines, then stage and commit the resolution.

---

## Step 4 — Verify the Rollback with the Diff Editor

Before pushing, confirm the revert removed *exactly* the right lines — no more, no less.

1. In the Source Control panel, click the **… → Compare with…** option — or from the Command Palette run **Git: Open Changes**.
2. Compare `HEAD` against `v1.0.1` (the last known-good tag):

```bash
git diff v1.0.1 HEAD -- src/health.py
```

Expected diff: zero changes — `HEAD` and `v1.0.1` are now identical for `health.py`.

If the diff is clean, push the revert:

1. Click the **sync** icon in the status bar (or **… → Push**).

> **Communicate immediately:** Paste the commit hash in your incident channel: "Reverted `g2j3k4l` — main is stable. Deploying now."

---

## Step 5 — Tag the Stable Rollback State

Give the rollback commit a named anchor so your CD pipeline and team can reference it unambiguously.

1. Command Palette → **Git: Create Tag**
2. Tag name: `v1.0.1-stable`
3. Tag message: `Post-rollback stable state after INC-911`

Push the tag:

```bash
git push origin v1.0.1-stable
```

```bash
git tag -l --sort=-v:refname | head -4
```

```
v1.0.1-stable
v1.0.1
v1.0.0
```

> **Why tag the rollback?** It gives your monitoring, deployment, and on-call tooling a stable named target to reference in dashboards and alerts. "Deployed `v1.0.1-stable`" is unambiguous in a post-mortem; a raw commit hash is not.

---

## Step 6 — Ship the Fix Forward

A revert buys time. The root cause (the bad code) still exists in history and will come back if the branch is ever re-merged. The correct resolution is a *fix-forward* commit — a proper change that removes the bad code and adds a correct implementation, reviewed through the normal PR flow.

1. Status bar → **+ Create new branch from…** → name it `hotfix/1.0.2-startup-probe` → base it on `v1.0.1-stable`.

2. Open `src/health.py` and add a safe startup probe:

```python
def startup_probe() -> bool:
    """Returns True if the application has completed initialisation."""
    return os.getenv("APP_READY", "false") == "true"
```

3. Open the diff editor to confirm the change is clean — no stray edits.

4. Stage → commit:
   ```
   Add safe startup probe using APP_READY env var (INC-911 fix forward)
   ```

5. Publish the branch: click **Publish Branch** in the status bar.

6. Create a PR via the GitHub Pull Requests panel → merge to `main` → back-merge to `develop`.

7. Tag the final release:
   - Command Palette → **Git: Create Tag** → `v1.0.2`
   - Message: `Release v1.0.2 — safe startup probe (INC-911)`

```bash
git log --oneline --graph main -6
```

```
*   j4k5l6m Merge hotfix/1.0.2-startup-probe into main
|\
| * i3j4k5l Add safe startup probe using APP_READY env var (INC-911 fix forward)
|/
* h1i2j3k Revert "Add startup probe (INC-911 investigation)"
* g2j3k4l Add startup probe (INC-911 investigation)
* f1a2b3c Merge branch 'hotfix/1.0.1'
```

You have completed the full incident lifecycle — entirely inside VS Code. You can now identify regressions using the Timeline and Source Control Graph, revert a broken commit with a single Command Palette action, verify correctness with the diff editor before pushing, and ship a clean fix-forward through the standard PR flow — all without leaving the editor.
