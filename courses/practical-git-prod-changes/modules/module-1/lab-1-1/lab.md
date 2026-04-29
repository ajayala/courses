# Branching Strategy and Protecting Main

In this lab you will set up a production-safe branching model, understand why `main` must always be deployable, and practise the discipline of never committing directly to production branches. By the end you will have a repository configured with branch conventions, protective aliases, and a tagged release — mirroring how professional teams guard their production code.

**Prerequisites:** Git installed and configured with your identity (`git config --global user.name` returns a value). Basic familiarity with `git commit` and `git log`.

---

## Step 1 — Why Main Must Always Be Deployable

In a production workflow, `main` (or `master`) represents what is currently running in production. Every commit on that branch must be in a state that can be deployed *right now* — no half-finished features, no debug code, no "WIP" commits.

```bash
# Initialise a demo project
mkdir prod-app && cd prod-app
git init
git switch -c main
echo "version: 1.0.0" > VERSION
echo "# Production App" > README.md
git add VERSION README.md
git commit -m "Initial release: v1.0.0"
```

Tag this as your first release so you can always find it:

```bash
git tag -a v1.0.0 -m "Release v1.0.0 — initial production deploy"
git log --oneline
```

> **Rule of thumb:** If you would not be comfortable deploying `main` to production at any random moment, your branching discipline has broken down.

---

## Step 2 — The Three-Branch Model

Most production teams use a small set of long-lived branches. Here is the model this course uses:

| Branch | Purpose | Who commits directly |
|--------|---------|---------------------|
| `main` | Production — always deployable | Nobody (PRs only) |
| `develop` | Integration — features merge here first | Nobody (PRs only) |
| `feature/*` | One branch per task or ticket | You |
| `hotfix/*` | Emergency production fixes | You (then PR to both main and develop) |

```bash
# Create the develop integration branch
git switch -c develop
echo "# Dev notes" > DEVNOTES.md
git add DEVNOTES.md
git commit -m "Add developer notes placeholder"

git log --oneline --graph --all
```

The branching model is now in place. `main` and `develop` start at the same point and will diverge as features are added.

---

## Step 3 — Simulate Branch Protection with a Pre-Push Hook

Hosting platforms (GitHub, GitLab) enforce branch protection server-side. Locally, a Git hook can warn you before you accidentally push directly to a protected branch.

```bash
cat > .git/hooks/pre-push << 'EOF'
#!/bin/sh
protected_branches="main develop"
current_branch=$(git symbolic-ref HEAD | sed 's|refs/heads/||')
for branch in $protected_branches; do
  if [ "$current_branch" = "$branch" ]; then
    echo "ERROR: Direct push to '$branch' is not allowed."
    echo "       Open a pull request from a feature branch instead."
    exit 1
  fi
done
exit 0
EOF
chmod +x .git/hooks/pre-push
```

Test the hook:

```bash
# This should be blocked
git push origin main 2>&1 || true
```

```
ERROR: Direct push to 'main' is not allowed.
       Open a pull request from a feature branch instead.
```

> **Note:** Hooks are not pushed to the remote and are not enforced for other team members. Always configure branch protection rules on your hosting platform as the authoritative guard.

---

## Step 4 — Create a Feature Branch the Right Way

Every change — no matter how small — starts from a fresh feature branch cut from the latest `develop`.

```bash
# Always start from the latest develop
git switch develop
git pull --rebase origin develop 2>/dev/null || true  # no-op if no remote yet

# Name your branch after the ticket or change
git switch -c feature/add-health-endpoint
```

| Convention | Example | Why |
|-----------|---------|-----|
| `feature/<ticket>` | `feature/JIRA-123` | Links branch to work item |
| `feature/<short-description>` | `feature/add-health-endpoint` | Human-readable for small teams |
| `hotfix/<version>` | `hotfix/1.0.1` | Signals urgency and target version |
| `chore/<description>` | `chore/update-dependencies` | Non-feature housekeeping |

```bash
# Do your work
echo "GET /health -> 200 OK" > health.txt
git add health.txt
git commit -m "Add health check endpoint documentation"

git log --oneline --graph --all
```

---

## Step 5 — Merge to Develop via a Simulated PR

In a real team, you would open a Pull Request and wait for approval. In this lab, you will simulate the merge to develop — using `--no-ff` to preserve the branch history.

```bash
git switch develop

# --no-ff always creates a merge commit even when fast-forward is possible
git merge --no-ff feature/add-health-endpoint -m "Merge feature/add-health-endpoint into develop"

git log --oneline --graph --all
```

```
*   d3a1b2c Merge feature/add-health-endpoint into develop
|\
| * c4e5f6a Add health check endpoint documentation
|/
* b7d8e9f Add developer notes placeholder
* a1b2c3d Initial release: v1.0.0
```

The graph shows exactly when the feature branch existed and when it was integrated — invaluable for tracing a regression.

---

## Step 6 — Promote Develop to Main and Tag the Release

When `develop` has been tested and is ready to ship, you promote it to `main` and tag the release.

```bash
git switch main
git merge --no-ff develop -m "Release v1.1.0: add health check endpoint"
git tag -a v1.1.0 -m "Release v1.1.0"

git log --oneline --graph --all
```

```bash
# Verify both tags exist
git tag -l
```

```
v1.0.0
v1.1.0
```

> **Tip:** Annotated tags (`-a`) store the tagger, date, and message in the object database. They are the correct choice for release tags. Lightweight tags (no `-a`) are just aliases to a commit, suitable only for local bookmarks.

You now have a repository that mirrors professional production discipline: a protected `main`, an integration `develop`, feature branches, a pre-push guard, and versioned release tags. In the next lab you will practise the complete feature-branch-to-PR cycle in detail, including how to keep your branch up to date as teammates merge their work.
