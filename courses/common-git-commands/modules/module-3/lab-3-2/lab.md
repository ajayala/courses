# Working with Remote Repositories

In this lab you will connect your local repository to a remote, push your work so others can see it, fetch updates from teammates, and explore the clone workflow used to start contributing to an existing project. These commands form the backbone of every collaborative Git workflow.

**Prerequisites:** Lab 3-1 completed; a repository on `main` with several commits. A GitHub, GitLab, or Gitea account (or a locally hosted bare repository) to use as the remote.

---

## Step 1 — Add a Remote

A *remote* is a named URL pointing to another copy of the repository — usually on a hosting service. The conventional name for the primary remote is `origin`.

```bash
# Add a remote called "origin"
git remote add origin https://github.com/your-username/my-git-project.git

# List configured remotes
git remote -v
```

```
origin  https://github.com/your-username/my-git-project.git (fetch)
origin  https://github.com/your-username/my-git-project.git (push)
```

> **Tip:** You can have multiple remotes. Teams sometimes add an `upstream` remote pointing to the canonical repository they forked from, so they can pull in official updates.

If you do not have a remote server, create a local bare repository to practise with:

```bash
git init --bare /tmp/remote-repo.git
git remote add origin /tmp/remote-repo.git
```

---

## Step 2 — Push Your Branch

`git push` uploads your local commits to the remote. The first time you push a branch, use `-u` to set the *tracking relationship* — after that, plain `git push` is enough.

```bash
# Push main and set the upstream tracking branch
git push -u origin main
```

Expected output:

```
Enumerating objects: 15, done.
Counting objects: 100% (15/15), done.
Writing objects: 100% (15/15), 1.45 KiB | 1.45 MiB/s, done.
To https://github.com/your-username/my-git-project.git
 * [new branch]      main -> origin/main
Branch 'main' set up to track remote branch 'main' from 'origin'.
```

From now on, `git push` (no arguments) will push `main` to `origin/main`.

```bash
# Push a feature branch the same way
git switch -c feature/readme-update
echo "## Usage" >> README.md
git add README.md
git commit -m "Add usage section to README"
git push -u origin feature/readme-update
```

---

## Step 3 — Fetch vs Pull

These two commands both download remote changes, but they behave differently.

```bash
# fetch: download remote changes into origin/* refs — does NOT modify your working tree
git fetch origin

# Check what arrived without merging
git log --oneline main..origin/main

# pull: fetch + merge (or rebase) in one step
git pull origin main
```

| Command | Downloads | Merges into working branch | Safe to run any time |
|---------|-----------|---------------------------|---------------------|
| `git fetch` | Yes | No | Yes |
| `git pull` | Yes | Yes | Usually |

> **Best practice:** Many experienced developers prefer `git fetch` followed by an explicit `git merge` or `git rebase`. This gives you a chance to inspect the incoming changes before integrating them.

```bash
# Inspect before integrating
git fetch origin
git log --oneline HEAD..origin/main
git merge origin/main
```

---

## Step 4 — Set Up Tracking Branches

A *tracking branch* tells Git which remote branch corresponds to your local branch. With tracking set, Git can report ahead/behind counts and allow short `git push` / `git pull` invocations.

```bash
# Check tracking configuration
git branch -vv
```

```
* main                  a1b2c3d [origin/main] Add changelog
  feature/readme-update c8d9e0f [origin/feature/readme-update] Add usage section to README
```

The `[origin/main]` annotation shows the tracking remote branch. If a branch is ahead, Git shows `[origin/main: ahead 2]`.

```bash
# Manually set tracking for an existing branch
git branch --set-upstream-to=origin/main main

# Shorthand on push
git push -u origin main
```

---

## Step 5 — Clone a Repository

`git clone` is how you start working on an existing project. It creates a local copy with `origin` already configured and `main` (or the default branch) checked out.

```bash
cd /tmp
git clone https://github.com/your-username/my-git-project.git cloned-project
cd cloned-project

git log --oneline
git remote -v
git branch -vv
```

Cloning also fetches all branches (as remote-tracking refs). To check out a remote branch locally:

```bash
git switch feature/readme-update
# Git automatically sets up tracking for known remote branches
```

> **Tip:** `git clone --depth 1 <url>` creates a *shallow clone* with only the latest commit. Useful for CI pipelines and large repositories where full history is not needed.

---

## Step 6 — The Pull Request Workflow

A pull request (PR) is not a Git concept — it is a code-review feature provided by hosting platforms (GitHub, GitLab, Bitbucket). But it maps directly onto Git operations you now know.

The workflow:

```
1. Fork or clone the repository
2. Create a feature branch
3. Commit your changes
4. Push the branch to the remote
5. Open a Pull Request on the hosting platform
6. Team reviews and approves
7. Platform merges the PR (usually a merge commit or squash)
8. Delete the remote branch
```

```bash
# After the PR is merged, clean up locally
git switch main
git pull origin main
git branch -d feature/readme-update
git push origin --delete feature/readme-update
```

Verify the remote branch is gone:

```bash
git remote prune origin
git branch -vv
```

`git remote prune origin` removes stale remote-tracking refs for branches that no longer exist on the remote.

Congratulations — you have completed the Common Git Commands course. You can now initialise repositories, track and undo changes with precision, branch and merge confidently, and collaborate via remote repositories. The commands you have practised here cover the vast majority of day-to-day Git usage on any professional team.
