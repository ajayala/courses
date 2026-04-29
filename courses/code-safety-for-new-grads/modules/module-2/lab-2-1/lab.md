# Secrets, Configuration, and the Environment Variable Pattern

In this lab you will learn the correct way to separate secrets from code, understand why the environment variable pattern is the industry standard, build a sample application that does it right, and examine the failure modes that undermine even well-intentioned implementations. Getting this right once means you carry the habit into every project you ever work on.

**Prerequisites:** Lab 1-1 completed; `safety-journal/` exists. Python 3 available (`python3 --version`).

---

## Step 1 — The Wrong Ways First

Understanding *why* the correct approach works requires seeing why the alternatives fail. These patterns appear in tutorials, StackOverflow answers, and legacy codebases. You will encounter all of them.

**Anti-pattern 1: Hardcoded values**

```python
# config.py — the worst possible pattern
DATABASE_URL = "postgresql://admin:SuperSecret123@prod-db.company.com:5432/myapp"
STRIPE_SECRET_KEY = "sk_live_4xKqR2mN8pLzT7vW..."
JWT_SECRET = "my-super-secret-key-do-not-share"
```

Why it fails: the file goes into version control. Even if the repository is private today, it may become public, be cloned to a laptop that is stolen, or have its access permissions changed. The secret is permanently bonded to the code.

**Anti-pattern 2: Config files committed to the repository**

```yaml
# config/production.yml — checked in to git
database:
  host: prod-db.company.com
  user: admin
  password: SuperSecret123     # ← in the repo forever
aws:
  access_key_id: AKIAIOSFODNN7EXAMPLE
  secret_access_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

Why it fails: even if `production.yml` is in `.gitignore` *now*, someone could have committed it before the ignore rule was added. Or a developer could accidentally commit it when staging files with `git add .`.

**Anti-pattern 3: Different files per environment but all committed**

```
config/
  development.yml   ← committed (no real secrets)
  staging.yml       ← committed (real secrets — mistake)
  production.yml    ← committed (real secrets — mistake)
```

```bash
cd safety-journal
cat > config-antipatterns.md << 'EOF'
# Configuration Anti-patterns Reference

## Anti-patterns I Will Avoid
| Pattern | Why it is dangerous | What to do instead |
|---------|--------------------|--------------------|
| Hardcoded values in source code | | |
| Config files with secrets committed | | |
| Per-environment secret files in repo | | |
| Secrets in Dockerfile or docker-compose.yml | | |
| Secrets in CI/CD pipeline YAML files | | |
EOF
```

---

## Step 2 — The Twelve-Factor App: Config Principle

The **Twelve-Factor App** is a methodology for building software-as-a-service applications. Factor III — Configuration — defines the pattern the entire industry has converged on:

> "Store config in the environment. Config varies between deploys; code does not. A litmus test for whether an app has all config correctly factored out of the code is whether the codebase could be made open source at any moment, without compromising any credentials."

That last sentence is the test. Run it mentally on every file you commit: *if this file were public right now, would anything bad happen?*

The environment variable pattern:

```
Code (in repository)           Environment (outside repository)
─────────────────────          ────────────────────────────────
import os                      DATABASE_URL=postgresql://...
                               STRIPE_KEY=sk_live_...
DB = os.getenv("DATABASE_URL") JWT_SECRET=abc123...
KEY = os.getenv("STRIPE_KEY")
                               ↑ These live in:
                               - Shell environment on dev machine
                               - CI/CD secret store (GitHub Secrets, etc.)
                               - Cloud secret manager in production
                               - .env file (local dev only, never committed)
```

The separation is clean: the code describes *where* to get the value (`DATABASE_URL`), never *what* the value is.

---

## Step 3 — Build a Correctly Configured Application

Build a small Flask application that uses environment variables for all configuration.

```bash
cd ..   # back to workspace root
mkdir safe-app && cd safe-app
```

```bash
cat > app.py << 'EOF'
import os
from flask import Flask, jsonify

app = Flask(__name__)

# All configuration read from environment — never hardcoded
DATABASE_URL = os.environ["DATABASE_URL"]       # raises KeyError if missing
STRIPE_KEY   = os.environ["STRIPE_KEY"]
JWT_SECRET   = os.environ["JWT_SECRET"]
APP_ENV      = os.getenv("APP_ENV", "development")  # optional with default

@app.route("/config-check")
def config_check():
    # NEVER return secret values — only confirm they are present
    return jsonify({
        "database_configured": bool(DATABASE_URL),
        "stripe_configured":   bool(STRIPE_KEY),
        "jwt_configured":      bool(JWT_SECRET),
        "environment":         APP_ENV,
    })

if __name__ == "__main__":
    app.run(debug=(APP_ENV == "development"))
EOF
```

> **`os.environ["KEY"]` vs `os.getenv("KEY")`:** Use `os.environ["KEY"]` (raises `KeyError` if missing) for required secrets — this makes the application fail loudly at startup rather than silently using `None` and causing mysterious errors later. Use `os.getenv("KEY", default)` only for optional configuration with a safe default.

---

## Step 4 — The .env File: Safe Local Development

The `.env` file is a local convenience for development — it stores environment variables in a file so you do not have to `export` them in your shell every session. It is **never committed to version control**.

```bash
# Create the .env file — this contains real local dev values
cat > .env << 'EOF'
DATABASE_URL=postgresql://localuser:localpass@localhost:5432/myapp_dev
STRIPE_KEY=sk_test_4eC39HqLyjWDarjtT1zdp7dc
JWT_SECRET=local-dev-secret-not-used-in-production
APP_ENV=development
EOF

# IMMEDIATELY add it to .gitignore
cat > .gitignore << 'EOF'
# Secret files — never committed
.env
.env.*
*.env

# Common secret-containing files
*.pem
*.key
*.p12
*.pfx
serviceAccountKey.json
*credentials*.json
EOF
```

Now create the `.env.example` file — the safe, committed version that shows teammates what variables are needed without revealing values:

```bash
cat > .env.example << 'EOF'
# Copy this file to .env and fill in real values for local development.
# Never commit .env — only .env.example belongs in version control.

DATABASE_URL=postgresql://user:password@localhost:5432/myapp_dev
STRIPE_KEY=sk_test_...       # Get from Stripe dashboard > Developers > API keys
JWT_SECRET=                  # Generate with: python3 -c "import secrets; print(secrets.token_hex(32))"
APP_ENV=development
EOF
```

```bash
# Verify .env is ignored — this should print nothing
git init && git add . && git status | grep ".env$" || echo "✓ .env is correctly ignored"
```

> **The `.env.example` contract:** Every developer clones the repo, copies `.env.example` to `.env`, fills in their own values, and never commits `.env`. The `.env.example` is the onboarding document for secrets. Keep it accurate — an out-of-date `.env.example` is as bad as no example at all.

---

## Step 5 — Secrets in Production: Beyond .env Files

`.env` files are for local development only. In staging and production, secrets are injected through proper secret management systems — never through committed files.

| Environment | Where secrets live | How app receives them |
|-------------|-------------------|----------------------|
| Local dev | `.env` file (not committed) | `python-dotenv` loads into `os.environ` |
| CI/CD (GitHub Actions) | GitHub Secrets | Injected as environment variables in workflow |
| CI/CD (GitLab) | CI/CD Variables | Injected as environment variables in pipeline |
| Cloud (AWS) | AWS Secrets Manager / Parameter Store | SDK call at startup, or ECS task IAM role |
| Cloud (GCP) | Secret Manager | SDK call, or mounted as env via Cloud Run |
| Kubernetes | Kubernetes Secrets (ideally via Vault) | Mounted as env vars or files into pod |

```bash
cat > requirements.txt << 'EOF'
flask==3.0.3
python-dotenv==1.0.1
EOF
```

Update `app.py` to load `.env` automatically in development:

```bash
cat > app.py << 'EOF'
import os
from flask import Flask, jsonify
from dotenv import load_dotenv

# load_dotenv() reads .env into os.environ — only affects local dev
# In production, real env vars are already set; .env is absent
load_dotenv()

app = Flask(__name__)

DATABASE_URL = os.environ["DATABASE_URL"]
STRIPE_KEY   = os.environ["STRIPE_KEY"]
JWT_SECRET   = os.environ["JWT_SECRET"]
APP_ENV      = os.getenv("APP_ENV", "development")

@app.route("/config-check")
def config_check():
    return jsonify({
        "database_configured": bool(DATABASE_URL),
        "stripe_configured":   bool(STRIPE_KEY),
        "jwt_configured":      bool(JWT_SECRET),
        "environment":         APP_ENV,
    })

if __name__ == "__main__":
    app.run(debug=(APP_ENV == "development"))
EOF
```

> **How `load_dotenv()` is safe in production:** `load_dotenv()` does not overwrite existing environment variables. In production, `DATABASE_URL` is already set by the platform before the app starts — `load_dotenv()` finds no `.env` file and does nothing. The same code works in both environments.

---

## Step 6 — Complete the Anti-patterns Reference

Go back to `safety-journal/config-antipatterns.md` and fill in every row of the table. Then add a second table documenting the correct pattern for each scenario.

```bash
cat >> safety-journal/config-antipatterns.md << 'EOF'

## The Correct Pattern Summary
| Scenario | Correct approach |
|----------|-----------------|
| Local development | .env file (gitignored) + python-dotenv / dotenv npm package |
| CI/CD pipeline | Platform secret store (GitHub Secrets, GitLab CI Variables) |
| Production (cloud) | Managed secret service (AWS Secrets Manager, GCP Secret Manager) |
| Production (Kubernetes) | Kubernetes Secrets mounted as env vars, ideally via Vault |
| Sharing required variables | .env.example committed — values blank or using placeholders |
| Rotating a secret | Update in secret store → redeploy → confirm old value stops working |

## Files That Must ALWAYS Be in .gitignore
- .env and all variants (.env.local, .env.production, *.env)
- *.pem, *.key, *.p12, *.pfx (private keys and certificates)
- serviceAccountKey.json (GCP service accounts)
- *credentials*.json (AWS, GCP credential files)
- ~/.ssh/id_rsa (never in a project repo — but worth saying)
- Any file named secrets.*, *.secret, or containing "password" in the name
EOF
```

```bash
cat safety-journal/config-antipatterns.md
```

In the next lab you will add the final line of defence before secrets reach the remote: `.gitignore` hardening, pre-commit hooks that scan for secrets before every commit, and GitHub's built-in secret scanning — the tools that catch mistakes even when developers are in a hurry.
