# Protecting Your Repository — .gitignore, Pre-commit Hooks, and Secret Scanning

In this lab you will harden the `safe-app` repository against accidental secret commits. You will learn the limits of `.gitignore`, install a pre-commit hook that scans every commit for secrets before they leave your machine, and configure GitHub's built-in secret scanning as a backstop. Defence in depth: catch mistakes at multiple layers rather than relying on any single tool.

**Prerequisites:** Lab 2-1 completed; `safe-app/` exists with a `.gitignore` and `.env.example`. Python 3 and pip available.

---

## Step 1 — The Limits of .gitignore

`.gitignore` is your first defence — but it has three important limitations that developers regularly misunderstand.

**Limitation 1: It only works on untracked files**

If a file was ever committed to the repository, adding it to `.gitignore` does nothing. The file remains tracked and all future changes to it continue to be staged. You must explicitly stop tracking it:

```bash
# Stop tracking a file that was previously committed
git rm --cached .env
git commit -m "Stop tracking .env (should never have been committed)"
# Now add it to .gitignore so it stays untracked
```

**Limitation 2: Pattern matching is exact**

```gitignore
.env          ← ignores only a file literally named ".env"
              ← does NOT ignore: .env.local, .env.production, settings.env, myapp.env
```

A comprehensive pattern covers all variants:

```bash
cd safe-app

cat > .gitignore << 'EOF'
# Environment and secret files — all variants
.env
.env.*
*.env
.envrc

# Private keys and certificates
*.pem
*.key
*.p12
*.pfx
*.crt
*.cer

# Cloud credential files
serviceAccountKey.json
*-service-account.json
credentials.json
*credentials*.json
.aws/credentials

# IDE and tool secrets
.netrc
.npmrc-local

# Build output that may capture env vars
*.log
npm-debug.log*
EOF
```

**Limitation 3: `.gitignore` does not protect secrets already in history**

Deleting a secret file and adding it to `.gitignore` leaves the secret in every previous commit. Tools like `git log -p`, `git show <hash>`, and `git stash show -p` will happily reveal it. Removing secrets from history requires `git filter-repo` — covered in Lab 3-2.

```bash
# Test your .gitignore — none of these should appear as "staged" after git add
touch .env .env.local .env.production secrets.env test.pem
git add . 2>/dev/null
git status | grep -E "\\.env|\\.pem" && echo "FAIL — file not ignored" || echo "✓ All secret patterns correctly ignored"
# Clean up test files
rm -f .env .env.local .env.production secrets.env test.pem
```

---

## Step 2 — Pre-commit Hooks: Catch Secrets Before They Commit

A pre-commit hook is a script that runs automatically before every `git commit`. If it exits with a non-zero code, the commit is aborted. This is the right place to scan for secrets — before they enter the repository at all, let alone reach the remote.

**Install `detect-secrets`** — a tool from Yelp that uses entropy analysis and pattern matching to find secrets:

```bash
pip install detect-secrets pre-commit
```

**Create a baseline** — a snapshot of any "secrets" already in the codebase that you have reviewed and confirmed are safe (like placeholders in `.env.example`):

```bash
detect-secrets scan > .secrets.baseline
```

Inspect the baseline to confirm it contains only expected items (like the placeholder values in `.env.example`):

```bash
cat .secrets.baseline
```

---

## Step 3 — Configure the Pre-commit Framework

`pre-commit` is a framework that manages git hooks as code. Configuration lives in `.pre-commit-config.yaml` — committed to the repository so every team member gets the same hooks automatically.

```bash
cat > .pre-commit-config.yaml << 'EOF'
repos:
  # Detect secrets using entropy analysis and pattern matching
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']

  # General hygiene hooks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files
        args: ['--maxkb=500']
      - id: detect-private-key      # Catches PEM-format private keys
      - id: check-merge-conflict
EOF
```

Install the hooks into the local `.git/hooks/` directory:

```bash
pre-commit install
```

```
pre-commit installed at .git/hooks/pre-commit
```

From this point, every `git commit` automatically runs all configured hooks. A commit that would introduce a secret is blocked before it reaches the staging area.

---

## Step 4 — Test the Hook: Try to Commit a Secret

Verify the hook works by trying to commit a fake secret:

```bash
# Add a fake API key to a Python file
cat > test_leak.py << 'EOF'
# This simulates accidentally leaving a key in code
API_KEY = "AKIAIOSFODNN7EXAMPLEFAKEKEY"
DATABASE_URL = "postgresql://admin:SuperSecret999@prod.example.com/db"
EOF

git add test_leak.py
git commit -m "Test that hook blocks secrets"
```

Expected output:

```
detect-secrets...............................................................Failed
- hook id: detect-secrets
- exit code: 1

ERROR: Potential secrets detected in staged files.

Secret Type: Secret Keyword
Location:    test_leak.py:3
```

The commit is **blocked**. Clean up the test file:

```bash
git restore test_leak.py 2>/dev/null || rm test_leak.py
git restore --staged . 2>/dev/null || true
```

> **What if a detection is a false positive?** Run `detect-secrets audit .secrets.baseline` to interactively mark known-safe items. Only do this after confirming the flagged value is genuinely not a secret — never mark something as safe just to make the hook pass.

---

## Step 5 — GitHub Secret Scanning: The Remote Backstop

Even with local hooks, someone might bypass them (`git commit --no-verify`) or use a different machine without hooks installed. GitHub's **Secret Scanning** feature scans every push against hundreds of known secret patterns and notifies you — or blocks the push entirely.

**Enabling Secret Scanning on GitHub:**

1. Go to your repository → **Settings → Security → Code security and analysis**
2. Enable **Secret scanning**
3. Optionally enable **Push protection** — this blocks the push before it reaches GitHub if a known secret pattern is detected

Push protection supports patterns from over 200 providers including AWS, Google, Stripe, GitHub, Slack, Twilio, and SendGrid. When enabled, a push containing a matching pattern is rejected with:

```
remote: ——————————————————————————————————————————————
remote:  GitHub Personal Access Token detected in commit abc1234
remote:  Secret type: github_personal_access_token
remote:  Secret value: ghp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
remote:  To push, visit: https://github.com/.../security/secret-scanning/...
remote: ——————————————————————————————————————————————
```

**Document the setup for your project:**

```bash
cat > SECURITY.md << 'EOF'
# Security Configuration

## Secret Prevention
This repository uses a defence-in-depth approach to prevent secrets from
being committed or pushed.

### Layer 1 — .gitignore
All secret file patterns are excluded. See `.gitignore` for the full list.
Never commit `.env` or any file containing real credentials.

### Layer 2 — Pre-commit hooks
`detect-secrets` scans every commit for high-entropy strings and known
secret patterns. Install hooks after cloning:
```
pip install pre-commit
pre-commit install
```

### Layer 3 — GitHub Secret Scanning with Push Protection
GitHub scans every push for 200+ provider token patterns and blocks
pushes containing known secrets.

## If You Find a Secret in the Repository
1. Do NOT push any additional commits — they make the history harder to clean
2. Contact the security team or team lead immediately
3. See INCIDENT-RUNBOOK.md for the full response procedure

## Reporting a Security Vulnerability
Please do not open a public GitHub issue. Email security@company.com.
EOF
```

---

## Step 6 — Audit Your Dependencies

Leaked secrets are not the only source of vulnerability. Outdated dependencies with known CVEs are equally dangerous — and equally preventable.

```bash
# Python: scan for known vulnerabilities in installed packages
pip install pip-audit
pip-audit -r requirements.txt
```

```bash
# Node.js equivalent (if working on a JS project):
# npm audit
# npm audit fix
```

```bash
cat > .github/dependabot.yml << 'EOF'
version: 2
updates:
  - package-ecosystem: pip
    directory: "/"
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
EOF
```

Dependabot automatically opens pull requests to update dependencies when vulnerabilities are discovered. With this configuration, you get weekly checks without any manual effort.

```bash
# Final check — confirm committed files contain no secrets
git diff --cached --name-only | xargs -I{} grep -l "password\|secret\|api_key\|token" {} 2>/dev/null \
  && echo "WARNING: Possible secrets in staged files — review before committing" \
  || echo "✓ No obvious secret keywords found in staged files"
```

In the next module you will look at the subtler leaks — the ones that bypass all of these controls because they come from developer tools, build pipelines, and logging frameworks rather than from code files.
