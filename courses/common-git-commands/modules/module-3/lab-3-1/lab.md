# Branching and Merging

In this lab you will create and switch between branches, make isolated changes on a feature branch, and then integrate those changes back into `main` using Git's merge strategies. You will also practise resolving a merge conflict — a core skill for any collaborative project.

**Prerequisites:** Lab 2-2 completed; a repository on `main` with at least two commits.

---

## Step 1 — Create and Switch Branches

A branch is just a lightweight movable pointer to a commit. Creating one is instant because Git only writes a 41-byte file.

```bash
# List existing branches
git branch

# Create a new branch called feature/greeting-cli
git branch feature/greeting-cli

# Switch to it
git switch feature/greeting-cli

# Or do both in one step (preferred)
git switch -c feature/greeting-cli
```

Verify you are on the new branch:

```bash
git branch
```

```
* feature/greeting-cli
  main
```

The `*` marks the active branch. Your working tree and index are shared — Git simply changes which commit the branch pointer will advance to on your next commit.

---

## Step 2 — Make Changes on the Feature Branch

Work on your feature branch without affecting `main`. Any commits you make here stay isolated until you merge.

```bash
# Add a CLI entry point
cat > cli.py << 'EOF'
import sys
from greet import greet, farewell

if __name__ == "__main__":
    action = sys.argv[1] if len(sys.argv) > 1 else "greet"
    name = sys.argv[2] if len(sys.argv) > 2 else "World"
    if action == "farewell":
        print(farewell(name))
    else:
        print(greet(name))
EOF

git add cli.py
git commit -m "Add CLI entry point for greeting module"
```

Check that `main` is unaffected:

```bash
git switch main
ls
# cli.py is NOT listed — it exists only on the feature branch

git switch feature/greeting-cli
ls
# cli.py is back
```

---

## Step 3 — Fast-Forward Merge

When the target branch (`main`) has not received any new commits since the feature branch was created, Git can simply advance the pointer — this is a *fast-forward* merge. No new commit is created.

```bash
# Switch back to main
git switch main

# Merge the feature branch
git merge feature/greeting-cli
```

Expected output:

```
Updating d4e5f6a..c8b3a1f
Fast-forward
 cli.py | 9 +++++++++
 1 file changed, 9 insertions(+)
 create mode 100644 cli.py
```

```bash
git log --oneline --graph
```

The history is a straight line — no merge commit, just an advanced pointer.

> **When to avoid fast-forward:** Use `git merge --no-ff` to always create a merge commit, which preserves the fact that a feature branch existed. Many teams enforce this for clarity.

---

## Step 4 — Three-Way Merge with a Diverged Branch

When both branches have new commits, Git cannot fast-forward. It finds the common ancestor and creates a *merge commit* that ties the two lines together.

```bash
# Create a diverging situation
git switch -c feature/config-loader

echo "CONFIG_FILE = '.gitconfig'" > settings.py
git add settings.py
git commit -m "Add settings module"

# Meanwhile, add a commit to main
git switch main
echo "# Changelog" > CHANGELOG.md
git add CHANGELOG.md
git commit -m "Add changelog"

# Now merge — both branches have moved
git merge feature/config-loader -m "Merge feature/config-loader into main"
```

```bash
git log --oneline --graph
```

```
*   e1f2a3b Merge feature/config-loader into main
|\
| * b4c5d6e Add settings module
* | a7b8c9d Add changelog
|/
* c8b3a1f Add CLI entry point for greeting module
```

---

## Step 5 — Resolve a Merge Conflict

A conflict occurs when two branches edit the *same lines* of the same file. Git cannot auto-merge them and asks you to decide.

```bash
# Create two branches that both modify greet.py line 1
git switch -c feature/shout-default
sed -i 's/return f'"'"'Hello, {name}!'"'"'/return f'"'"'HELLO, {name}!'"'"'/' greet.py
git add greet.py
git commit -m "Make greet shout by default"

git switch main
sed -i 's/return f'"'"'Hello, {name}!'"'"'/return f'"'"'Hi there, {name}!'"'"'/' greet.py
git add greet.py
git commit -m "Use casual greeting"

# Attempt the merge
git merge feature/shout-default
```

Git reports a conflict. Open `greet.py` and you will see conflict markers:

```
<<<<<<< HEAD
    return f'Hi there, {name}!'
=======
    return f'HELLO, {name}!'
>>>>>>> feature/shout-default
```

Edit the file to keep the version you want, then remove the markers:

```bash
# After editing, mark the conflict resolved
git add greet.py
git commit -m "Merge feature/shout-default: keep casual greeting"
```

> **Tip:** VS Code's built-in merge editor (open the conflicted file and click *Resolve in Merge Editor*) gives you a side-by-side view with accept buttons for each hunk.

---

## Step 6 — Delete Merged Branches

Once a branch is merged, its pointer is no longer needed — the history is preserved in the merge commit.

```bash
# Delete a fully merged branch
git branch -d feature/greeting-cli
git branch -d feature/config-loader

# Force-delete an unmerged branch (use with care)
git branch -D feature/shout-default

# Verify
git branch
```

```
* main
```

Keeping your branch list tidy makes it easier to navigate a project. Deleted local branches do not affect remote branches — you will manage those in the next lab, where you learn to push, fetch, and pull from a remote repository.
