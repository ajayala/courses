# Hotfixes — Emergency Changes to Production

In this lab you will handle the most time-sensitive Git scenario: a critical bug in production that cannot wait for the normal feature-branch cycle. You will cut a hotfix branch directly from the production tag, apply the minimal targeted fix, tag a patch release, and back-merge the fix into both `main` and `develop` so neither branch falls behind.

**Prerequisites:** Lab 2-2 completed; the `prod-app` repository with `main` tagged at `v1.1.0` and a `develop` branch ahead of it.

---

## Step 1 — The Hotfix Branching Model

A hotfix bypasses `develop` entirely because `develop` may contain unfinished features that are not ready for production. The hotfix branch cuts directly from the last known-good production tag.

```
main (v1.1.0) ──────────────────────────────► main (v1.1.1)
                \                            /
                 hotfix/1.1.1 ──────────────
```

```bash
cd prod-app

# Confirm the current production tag
git tag -l --sort=-v:refname | head -3
```

```
v1.1.0
v1.0.0
```

```bash
# List what is on main right now
git log main --oneline -5
```

> **Rule:** Never cut a hotfix from `develop`. Even one unreviewed feature in `develop` could turn a targeted patch into a larger, riskier release.

---

## Step 2 — Cut the Hotfix Branch from the Production Tag

```bash
# Check out the exact production state
git switch main

# Create the hotfix branch from the production tag — not from HEAD if main has moved
git switch -c hotfix/1.1.1 v1.1.0
```

Verify you are on the right base:

```bash
git log --oneline -3
```

```
a1b2c3d (tag: v1.1.0, HEAD -> hotfix/1.1.1) Release v1.1.0: add health check endpoint
...
```

Your hotfix branch is now a clean snapshot of what production is running, with no extra commits from `develop` mixed in.

---

## Step 3 — Apply the Minimal Targeted Fix

The golden rule of hotfixes: change **only what is broken**. Resist the temptation to clean up nearby code or add small improvements — every additional line is a risk.

```bash
# Simulate the bug: rate limit was accidentally set to 0, blocking all traffic
cat rate_limit.cfg
```

```
[rate_limit]
requests_per_minute = 0
burst = 10
enabled = true
```

```bash
# Apply the fix — restore the correct value
sed -i 's/requests_per_minute = 0/requests_per_minute = 60/' rate_limit.cfg

# Verify the fix is correct and nothing else changed
git diff
```

```diff
-requests_per_minute = 0
+requests_per_minute = 60
```

```bash
git add rate_limit.cfg
git commit -m "Fix: restore rate limit to 60 req/min (was incorrectly set to 0)"
```

> **Tip:** Write the commit message so an on-call engineer reading a post-mortem understands immediately what was changed and why. Include the incident ID if you have one: `Fix: restore rate limit to 60 req/min (INC-901)`.

---

## Step 4 — Tag the Patch Release

Tag the hotfix commit *before* merging to `main`. This way the tag points to the exact fix commit, and `main`'s merge commit does not pollute the tag reference.

```bash
git tag -a v1.1.1 -m "Hotfix v1.1.1 — restore rate limit to 60 req/min (INC-901)"

# Confirm the tag points to the right commit
git show v1.1.1 --stat
```

```
Tag: v1.1.1
Tagger: Ada Lovelace <ada@example.com>
Date: Tue Apr 29 ...

Hotfix v1.1.1 — restore rate limit to 60 req/min (INC-901)

commit f2a3b4c
    Fix: restore rate limit to 60 req/min (was incorrectly set to 0)

 rate_limit.cfg | 2 +-
```

---

## Step 5 — Merge the Hotfix into Main

```bash
git switch main
git merge --no-ff hotfix/1.1.1 -m "Merge hotfix/1.1.1 into main"

git log --oneline --graph -5
```

```
*   g3h4i5j Merge hotfix/1.1.1 into main
|\
| * f2a3b4c (tag: v1.1.1) Fix: restore rate limit to 60 req/min
|/
* a1b2c3d (tag: v1.1.0) Release v1.1.0: add health check endpoint
```

Push to the remote immediately so the fix can be deployed:

```bash
git push origin main
git push origin v1.1.1
```

> **Deployment:** At this point your CD pipeline should detect the new tag on `main` and trigger a production deployment automatically. Verify the deployment succeeds before proceeding to back-merge.

---

## Step 6 — Back-Merge the Hotfix into Develop

If you skip this step, the next release from `develop` will re-introduce the bug because `develop` never received the fix.

```bash
git switch develop
git merge --no-ff hotfix/1.1.1 -m "Back-merge hotfix/1.1.1 into develop"
```

If there is a conflict (because `develop` has already moved past the line you fixed), resolve it carefully — the production value (`60`) takes precedence until you explicitly decide to change it.

```bash
# Verify the fix is present on develop
grep "requests_per_minute" rate_limit.cfg
```

```
requests_per_minute = 60
```

```bash
# Clean up the hotfix branch — its work is done
git branch -d hotfix/1.1.1
git push origin --delete hotfix/1.1.1 2>/dev/null || true

git log --oneline --graph --all -8
```

You have completed the full hotfix lifecycle: isolated fix from production tag, patch release tag, merged to `main` for deployment, and back-merged to `develop` to keep the branches in sync. In the next lab you will handle the scenario where the fix itself was wrong and you need to roll back production entirely.
