# Rebasing and Keeping a Clean History

In this lab you will use interactive rebase to squash fixup commits, reorder and reword history, and produce a pristine branch ready to enter `develop`. You will also learn when and how to force-push a rebased branch safely, and understand the fundamental trade-off between merge and rebase strategies.

**Prerequisites:** Lab 2-1 completed; the `prod-app` repository with `feature/JIRA-42-add-rate-limiting` containing a `fixup!` commit.

---

## Step 1 — Why History Matters in Production Repos

`git log` on a production repository is not just archaeology — it is an operational tool. When an incident happens at 2 AM, engineers use `git bisect`, `git log`, and `git blame` to find the commit that introduced the regression. A history full of `WIP`, `fix`, and `oops` commits makes that search slower and more error-prone.

```bash
cd prod-app
git switch feature/JIRA-42-add-rate-limiting

# See the current messy history
git log develop..HEAD --oneline
```

```
d4e5f6a Add PR description for review reference
c3d4e5f fixup! Add RateLimitMiddleware stub
b2c3d4e Add RateLimitMiddleware stub
a1b2c3d Add rate-limiting configuration
```

The `fixup!` commit is implementation noise. A reader of the history does not need to know that a reviewer asked you to add a docstring — they need to know that `RateLimitMiddleware` was added. Your goal: collapse the fixup into its parent.

---

## Step 2 — Interactive Rebase with --autosquash

`git rebase -i --autosquash` automatically moves `fixup!` and `squash!` commits next to their targets and marks them for squashing. All you need to supply is the base commit.

```bash
# Rebase interactively from the point where the branch diverged from develop
git rebase -i develop --autosquash
```

Your editor opens with a todo list. With `--autosquash`, Git has already arranged it:

```
pick a1b2c3d Add rate-limiting configuration
pick b2c3d4e Add RateLimitMiddleware stub
fixup c3d4e5f fixup! Add RateLimitMiddleware stub
pick d4e5f6a Add PR description for review reference
```

Save and close. Git replays the commits, squashing `c3d4e5f` silently into `b2c3d4e`.

```bash
git log develop..HEAD --oneline
```

```
e5f6a7b Add PR description for review reference
f6a7b8c Add RateLimitMiddleware stub
a1b2c3d Add rate-limiting configuration
```

The fixup is gone. The middleware commit now contains both the original stub and the docstring correction.

> **Tip:** Add `rebase.autoSquash = true` to your global git config so `--autosquash` is always on: `git config --global rebase.autoSquash true`

---

## Step 3 — Reword and Reorder Commits

Sometimes you need to rename a commit or move it earlier in the list. Interactive rebase gives you full control.

```bash
git rebase -i develop
```

The todo list:

```
pick a1b2c3d Add rate-limiting configuration
pick f6a7b8c Add RateLimitMiddleware stub
pick e5f6a7b Add PR description for review reference
```

Change `pick` to `reword` on a commit to rename it:

```
pick a1b2c3d Add rate-limiting configuration
reword f6a7b8c Add RateLimitMiddleware stub
pick e5f6a7b Add PR description for review reference
```

Save and close the todo file. Git will open a second editor session for each `reword` commit. Update the message:

```
Add RateLimitMiddleware with per-client request throttling
```

```bash
git log develop..HEAD --oneline
```

```
e5f6a7b Add PR description for review reference
g7h8i9j Add RateLimitMiddleware with per-client request throttling
a1b2c3d Add rate-limiting configuration
```

---

## Step 4 — Rebase onto the Latest Develop

Just before merging, do a final sync with `develop` to pick up any commits that landed while you were polishing.

```bash
# Simulate another teammate merge on develop
git switch develop
echo "METRICS_PORT=9090" >> .env
git add .env
git commit -m "Add metrics port to env config"
git switch feature/JIRA-42-add-rate-limiting

# Rebase onto the now-ahead develop
git rebase develop
```

```bash
git log --oneline --graph
```

```
* e5f6a7b (HEAD -> feature/JIRA-42-add-rate-limiting) Add PR description for review reference
* g7h8i9j Add RateLimitMiddleware with per-client request throttling
* a1b2c3d Add rate-limiting configuration
* h9i0j1k Add metrics port to env config   ← teammate commit, now below yours
* ...
```

Your commits are cleanly on top — no merge commit in the middle.

---

## Step 5 — Force-Push a Rebased Branch Safely

Rebasing rewrites commit hashes. If you already pushed the branch to the remote, you must force-push to update it. Use `--force-with-lease` — it refuses to push if someone else has pushed to the branch since you last fetched, protecting you from overwriting their work.

```bash
# Safe force-push — fails if the remote has moved ahead of your last fetch
git push --force-with-lease origin feature/JIRA-42-add-rate-limiting
```

| Flag | Behaviour | When to use |
|------|-----------|-------------|
| `--force` | Overwrites remote unconditionally | Never on shared branches |
| `--force-with-lease` | Overwrites only if remote matches your last-fetched state | Solo feature branches after rebase |

> **Warning:** Never force-push `main`, `develop`, or any branch that other people have checked out. Force-pushing rewrites history for everyone — their local branches will diverge and they will need to hard-reset.

---

## Step 6 — Merge vs Rebase: Choosing the Right Strategy

Both strategies integrate changes, but they produce very different histories. Your team should adopt one consistently.

```bash
# Merge approach: preserves branch topology
git switch develop
git merge --no-ff feature/JIRA-42-add-rate-limiting -m "Merge JIRA-42: add rate limiting"
git log --oneline --graph -6
```

```
*   f1a2b3c Merge JIRA-42: add rate limiting
|\
| * e5f6a7b Add PR description for review reference
| * g7h8i9j Add RateLimitMiddleware with per-client request throttling
| * a1b2c3d Add rate-limiting configuration
|/
* h9i0j1k Add metrics port to env config
```

| Strategy | History shape | Best for |
|----------|--------------|---------|
| `merge --no-ff` | Branchy — shows feature existed | Teams that value branch context |
| `rebase + merge --ff` | Linear — clean single thread | Teams that prioritise bisect speed |
| `squash merge` | One commit per PR | Teams that want minimal history noise |

You now have a fully polished branch — squashed, rebased, and ready to merge. The history tells a clear story with no noise. In the next module you will face the hardest scenario: something has gone wrong in production and you need to act fast.
