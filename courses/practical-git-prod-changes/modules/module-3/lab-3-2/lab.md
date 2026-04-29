# Rolling Back a Broken Deployment

In this lab you will handle the worst-case scenario: a change has reached production and is causing an outage. You will learn to identify the offending commit, choose the right rollback strategy (`revert` vs `reset`), handle the special case of reverting a merge commit, and use Git tags to communicate the incident state to your team.

**Prerequisites:** Lab 3-1 completed; the `prod-app` repository with `main` tagged at `v1.1.1` and the rate-limiting middleware merged into `develop`.

---

## Step 1 — Identify the Bad Commit

When production is broken, the first step is pinpointing *what* changed and *when*. `git log`, `git bisect`, and `git diff` are your instruments.

```bash
cd prod-app
git switch main

# Review recent commits on main
git log --oneline -8
```

```
g3h4i5j Merge hotfix/1.1.1 into main
f2a3b4c (tag: v1.1.1) Fix: restore rate limit to 60 req/min
a1b2c3d (tag: v1.1.0) Release v1.1.0: add health check endpoint
```

```bash
# Compare the current state to the last known-good tag
git diff v1.1.0 HEAD -- rate_limit.cfg middleware.py
```

For larger changes, use `git bisect` to binary-search for the regression:

```bash
git bisect start
git bisect bad HEAD          # current state is broken
git bisect good v1.1.0       # last known-good state
# Git checks out a middle commit; test it, then run:
# git bisect good   (if it works)
# git bisect bad    (if it is broken)
# Repeat until Git prints: "X is the first bad commit"
git bisect reset             # exit bisect mode when done
```

> **Tip:** Automate bisect with a test script: `git bisect run ./test.sh`. Git will check out commits and run the script automatically, marking them good or bad based on the exit code.

---

## Step 2 — Revert vs Reset: Choosing Your Rollback Weapon

This is the most important decision during a rollback. Get it wrong and you can create more problems than you solve.

| Situation | Use | Why |
|-----------|-----|-----|
| Commit is on shared `main` | `git revert` | Adds a new commit; safe for shared history |
| Commit is only local / on a feature branch | `git reset --hard` | Rewrites history; safe because nobody else has it |
| Need to roll back to a specific tag | `git revert <range>` | Reverts a sequence of commits one by one |

```bash
# Simulate a bad deployment: a commit that introduces broken config
echo "requests_per_minute = 0" >> rate_limit.cfg
git add rate_limit.cfg
git commit -m "BREAKING: set rate limit to 0 (mistaken experiment)"

git log --oneline -4
```

The bad commit is now on `main` and has been pushed. You must use `git revert`.

---

## Step 3 — Revert the Breaking Commit

`git revert` creates a new commit that is the inverse of the target. The bad commit stays in history (for auditability) but its changes are undone.

```bash
# Revert the most recent commit
git revert HEAD --no-edit
```

```
[main h1i2j3k] Revert "BREAKING: set rate limit to 0 (mistaken experiment)"
 1 file changed, 1 deletion(-)
```

```bash
git log --oneline -5
```

```
h1i2j3k Revert "BREAKING: set rate limit to 0 (mistaken experiment)"
i2j3k4l BREAKING: set rate limit to 0 (mistaken experiment)
g3h4i5j Merge hotfix/1.1.1 into main
...
```

```bash
# Verify the config is correct
grep "requests_per_minute" rate_limit.cfg
```

```
requests_per_minute = 60
requests_per_minute = 60
```

> **Audit trail:** The bad commit remains visible in the log. This is intentional — it lets you reconstruct exactly what happened during the incident, which is essential for post-mortems.

---

## Step 4 — Reverting a Merge Commit

Reverting a merge commit requires an extra flag: `-m` (mainline), which tells Git which parent to treat as the "before" state.

```bash
# Identify a merge commit to revert
git log --oneline --merges -3
```

```
g3h4i5j Merge hotfix/1.1.1 into main
```

```bash
# Revert the merge — parent 1 is typically the branch you merged INTO (main)
git revert -m 1 g3h4i5j --no-edit
```

```
[main j3k4l5m] Revert "Merge hotfix/1.1.1 into main"
```

> **Warning:** After reverting a merge commit, re-merging the same branch will appear as a no-op because Git thinks those commits are already present. To re-introduce the changes, you must revert the revert commit first: `git revert <revert-commit-hash>`.

---

## Step 5 — Tag the Rollback State

Tagging the rollback makes it easy for your team and your CD pipeline to identify the safe state. It also gives you a named anchor for future `git diff` and `git log` commands.

```bash
# Tag the post-rollback commit
git tag -a v1.1.1-rollback -m "Rollback: reverted broken rate-limit experiment (INC-902)"

# Optionally, reset the stable tag to this point
git tag -f v1.1.1-stable HEAD -m "Stable production state after INC-902 rollback"

git tag -l --sort=-v:refname | head -5
```

```bash
# Push the rollback tag so CI/CD can deploy it
git push origin main
git push origin v1.1.1-rollback
```

> **Communication:** Post the rollback tag in your incident channel immediately: "Rolled back to `v1.1.1-rollback` — production is stable. RCA in progress."

---

## Step 6 — Fix Forward: The Post-Incident Commit

A rollback buys time — it does not fix the underlying issue. The correct long-term response is a *fix-forward* commit: a proper change that goes through the normal feature-branch review process, just on an expedited timeline.

```bash
# Create a fix-forward branch from the rolled-back main
git switch -c hotfix/1.1.2-rate-limit-guard

# The actual fix: remove the duplicate/erroneous line and add a guard comment
cat > rate_limit.cfg << 'EOF'
[rate_limit]
# Minimum safe value is 10. Setting to 0 disables all traffic. See INC-902.
requests_per_minute = 60
burst = 10
enabled = true
EOF

git add rate_limit.cfg
git commit -m "Guard against zero rate-limit value; add inline warning comment (INC-902)"
```

```bash
# Tag and merge as a proper patch release
git switch main
git merge --no-ff hotfix/1.1.2-rate-limit-guard -m "Merge hotfix/1.1.2-rate-limit-guard (INC-902 fix forward)"
git tag -a v1.1.2 -m "Release v1.1.2 — rate-limit guard (INC-902)"

# Back-merge to develop
git switch develop
git merge --no-ff hotfix/1.1.2-rate-limit-guard -m "Back-merge hotfix/1.1.2-rate-limit-guard into develop"

git branch -d hotfix/1.1.2-rate-limit-guard

git log --oneline --graph --all -10
```

Congratulations — you have completed the full production incident response cycle using Git: identified the bad commit, reverted it safely, tagged the rollback state, and shipped a proper fix forward through the standard branch model. These techniques, combined with the branching discipline from earlier modules, give you a complete toolkit for making production changes with confidence.
