# Feature Branches and the Pull Request Flow

In this lab you will practise the complete lifecycle of a production change: cutting a branch from the latest `develop`, making focused commits, keeping your branch current as teammates merge their work, and finally preparing the branch for review. You will also learn how to address reviewer feedback cleanly — without cluttering the history with "fix review comment" noise.

**Prerequisites:** Lab 1-1 completed; the `prod-app` repository with `main` and `develop` branches and tag `v1.1.0`.

---

## Step 1 — Start from the Latest Develop

Before cutting a new branch, always pull the latest `develop`. Starting from a stale base means you will have to resolve conflicts later — usually under pressure.

```bash
cd prod-app
git switch develop
git pull --rebase origin develop 2>/dev/null || true

# Check you are up to date
git log --oneline -5
```

Cut a branch named after the ticket or change you are making:

```bash
git switch -c feature/JIRA-42-add-rate-limiting
```

> **Tip:** Prefixing with a ticket number (`JIRA-42`) links the branch to your issue tracker and makes automated tooling (dashboards, bots) work out of the box. If your team does not use ticket numbers, use a short imperative description: `feature/add-rate-limiting`.

---

## Step 2 — Make Focused, Atomic Commits

Each commit should represent one logical change — something a reviewer can understand in isolation. Avoid "catch-all" commits that bundle unrelated edits.

```bash
# First logical unit: the rate-limiting config
cat > rate_limit.cfg << 'EOF'
[rate_limit]
requests_per_minute = 60
burst = 10
enabled = true
EOF

git add rate_limit.cfg
git commit -m "Add rate-limiting configuration"
```

```bash
# Second logical unit: the middleware stub
cat > middleware.py << 'EOF'
class RateLimitMiddleware:
    def __init__(self, app, limit=60):
        self.app = app
        self.limit = limit

    def __call__(self, environ, start_response):
        # TODO: implement token bucket
        return self.app(environ, start_response)
EOF

git add middleware.py
git commit -m "Add RateLimitMiddleware stub"
```

```bash
git log --oneline
```

| Good commit | Bad commit |
|------------|-----------|
| `Add rate-limiting configuration` | `WIP stuff` |
| `Add RateLimitMiddleware stub` | `Changes` |
| `Fix off-by-one in token bucket` | `Final final FINAL` |

---

## Step 3 — Keep Your Branch Up to Date with Rebase

While you were working, a teammate merged their changes into `develop`. Rebasing replays your commits on top of the latest `develop`, keeping a linear history and surfacing conflicts early.

```bash
# Simulate a teammate commit on develop
git switch develop
echo "LOG_LEVEL=info" > .env
git add .env
git commit -m "Add default environment config"
git switch feature/JIRA-42-add-rate-limiting

# Rebase onto the updated develop
git rebase develop
```

Expected output:

```
Successfully rebased and updated refs/heads/feature/JIRA-42-add-rate-limiting.
```

```bash
git log --oneline --graph
```

Your two feature commits now sit cleanly on top of the teammate's commit — no merge commit, no diverged history.

> **When to use merge instead of rebase:** If your feature branch is shared with other contributors, rebase rewrites commits they may have checked out. In that case, merge (`git merge develop`) is safer. Rebase is ideal for solo feature branches.

---

## Step 4 — Simulate a Code Review: Add a Fixup Commit

A reviewer asks you to remove the `# TODO` comment and add a docstring to the middleware. Rather than amending history (which would require a force-push), add a *fixup* commit — you will squash it before merging.

```bash
cat > middleware.py << 'EOF'
class RateLimitMiddleware:
    """WSGI middleware that enforces a per-client request rate limit."""

    def __init__(self, app, limit=60):
        self.app = app
        self.limit = limit

    def __call__(self, environ, start_response):
        return self.app(environ, start_response)
EOF

git add middleware.py
git commit -m "fixup! Add RateLimitMiddleware stub"
```

Using the `fixup!` prefix tells `git rebase -i --autosquash` exactly which commit this belongs to — you will use that in the next lab.

```bash
git log --oneline
```

```
c3d4e5f fixup! Add RateLimitMiddleware stub
b2c3d4e Add RateLimitMiddleware stub
a1b2c3d Add rate-limiting configuration
```

---

## Step 5 — Write a PR Description That Reviewers Can Act On

A good PR description answers three questions: *what changed*, *why*, and *how to verify it*. This is not just courtesy — it is the artefact that lives in git history long after the branch is gone.

```bash
# Save your PR description as a file for reference
cat > .pr-description.md << 'EOF'
## What
Adds a `RateLimitMiddleware` WSGI component and supporting configuration.

## Why
Unthrottled endpoints caused a cascading failure in production on 2026-04-15 (INC-887).
This is the first step in the rate-limiting initiative (JIRA-42).

## How to verify
1. Run `python -c "from middleware import RateLimitMiddleware; print('OK')"`
2. Check `rate_limit.cfg` contains `enabled = true`

## Checklist
- [x] Config file added
- [x] Middleware class created with docstring
- [ ] Token bucket implementation (follow-up: JIRA-43)
EOF

git add .pr-description.md
git commit -m "Add PR description for review reference"
```

> **Tip:** On GitHub and GitLab you paste this text into the PR body UI. Saving it as a file in the branch gives you a local draft and ensures the description survives even if the PR is closed and reopened.

---

## Step 6 — Final Pre-Merge Checks

Before requesting a merge, run a quick self-review checklist.

```bash
# Review every commit that will enter develop
git log develop..HEAD --oneline
```

```
d4e5f6a Add PR description for review reference
c3d4e5f fixup! Add RateLimitMiddleware stub
b2c3d4e Add RateLimitMiddleware stub
a1b2c3d Add rate-limiting configuration
```

```bash
# Diff against develop — what exactly is changing?
git diff develop...HEAD --stat
```

```
 .pr-description.md | 16 ++++++++++++++++
 middleware.py       |  8 ++++++++
 rate_limit.cfg      |  5 +++++
 3 files changed, 29 insertions(+)
```

```bash
# Make sure there are no leftover debug lines or conflict markers
git diff develop...HEAD | grep -E "^(\+.*<<<|>>>|===)" || echo "Clean — no conflict markers"
```

You now have a well-structured feature branch that is ready for review: atomic commits, kept up to date via rebase, reviewer feedback addressed via a fixup commit, and a clear PR description. In the next lab you will tidy the history by squashing the fixup commit and using interactive rebase to produce a clean, merge-ready branch.
