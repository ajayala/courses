# How Developer Tools Leak Secrets

In this lab you will study the subtler category of secret leak — the ones that bypass `.gitignore` and pre-commit hooks entirely because they come from the tools developers use every day: linters, loggers, debuggers, CI pipelines, and Docker. These leaks are harder to spot precisely because the tools are trusted and familiar.

**Prerequisites:** Lab 2-2 completed; `safe-app/` and `safety-journal/` exist.

---

## Step 1 — Linters and Static Analysis Tools

Linters are the canonical example from this course's topic. Here is exactly how secrets leak through them.

**The scenario:**

A developer configures ESLint or Flake8 with a plugin that checks for hardcoded strings. The linter runs in CI. The CI log — often readable by everyone with repository access, sometimes publicly visible on open-source projects — includes the lint output.

```yaml
# .github/workflows/lint.yml — a common but risky pattern
- name: Run linter
  run: flake8 --show-source .   # ← --show-source prints the actual line
```

If a secret was hardcoded in the source:

```
app/config.py:14:1: E501 line too long (89 > 79 characters)
    STRIPE_KEY = "sk_live_4xKqR2mN8pLzT7vW3cY9oJQzXmBpRsNd2KfHyLcVgAeUiToWj"
                  ↑ The entire key is now in the CI log
```

**Other linter vectors:**

| Tool | How it can leak | Mitigation |
|------|----------------|-----------|
| ESLint with `--print-ast` | Prints the full AST including string literals | Never print AST in CI |
| Pylint `--verbose` | Includes file contents in some error outputs | Control verbosity in CI |
| Bandit (security linter) | Reports the line containing hardcoded passwords | Review CI log access |
| Semgrep | Output includes the matched code line | Restrict CI log visibility |
| Custom lint rules | May log matched content for debugging | Never log secret patterns |

```bash
cd safety-journal
cat > tool-risks.md << 'EOF'
# Developer Tool Secret Leak Risks

## Linters
How they leak:
Config change to prevent leaks:

## Online Tools
How they leak:
Config change to prevent leaks:

## Loggers
How they leak:
Config change to prevent leaks:

## Error pages and stack traces
How they leak:
Config change to prevent leaks:

## CI/CD pipelines
How they leak:
Config change to prevent leaks:

## Docker
How they leak:
Config change to prevent leaks:

## Debuggers and IDE integrations
How they leak:
Config change to prevent leaks:
EOF
```

---

## Step 2 — Online Linting and Formatting Tools

Online tools — Prettier Playground, ESLint demos, black/autopep8 playgrounds, beautifier.io, codebeautify.org, online SQL formatters — are convenient, but they introduce a risk that is easy to overlook: **you are sending your code to a third-party server.**

**What can happen to code you paste:**

| What the service may do | Why it matters |
|------------------------|---------------|
| Log requests for debugging | Your code (including any secrets) lives in their server logs |
| Collect analytics on submitted code | Your secret becomes data in their analytics pipeline |
| Retain submissions for future reference | Code submitted today may be queryable months later |
| Use submissions to train AI models | Many services include this in their terms of service |
| Cache formatted output (shared cache) | A future user querying the same input may receive your code |
| Suffer a data breach | Your code is now part of a dataset that could be stolen |

**The realistic scenario:**

```
A developer is writing a database migration script.
The connection string is temporarily hardcoded while testing:

    postgresql://admin:SuperSecret123@prod-db.company.com/myapp

They paste the file into an online SQL formatter to tidy up the queries.
The formatter service logs the request. Their server is compromised six
months later. The attacker finds the credential in a log file.
```

This is not hypothetical. Several high-profile breaches have been traced back to secrets submitted to paste sites and online tools.

**Common online tools and their risk level:**

| Tool | Typical use | Risk if code contains secrets |
|------|-------------|-------------------------------|
| prettier.io/playground | JavaScript / CSS formatting | Medium — logs are possible; terms allow data use |
| eslint.org/play | JavaScript linting | Medium — same as above |
| black.vercel.app | Python formatting | Medium — third-party hosted, not official |
| pythonformat.com | Python formatting | High — unclear data handling policies |
| beautifier.io | HTML / CSS / JS / JSON | High — commercial service, broad terms |
| codebeautify.org | JSON / SQL / XML / YAML | High — free service, revenue from ads/data |
| sqlfiddle.com | SQL queries | High — executes code server-side |
| jsfiddle.net / codepen.io | JavaScript | Medium–High — submissions are often public by default |

> **The safe rule:** Treat any code that leaves your machine as potentially public. Use locally installed tools instead.

**Safe local alternatives:**

```bash
# Python — install black locally
pip install black
black .                    # formats all Python files in place

# JavaScript / TypeScript / CSS / JSON — install Prettier locally
npm install --save-dev prettier
npx prettier --write .     # formats all supported files in place

# JavaScript — ESLint locally
npm install --save-dev eslint
npx eslint --fix .

# SQL — use your IDE's built-in formatter (VS Code has SQL formatting extensions)
# or a local CLI tool such as sql-formatter:
npm install -g sql-formatter
sql-formatter -l postgresql query.sql

# JSON — jq is available locally on most systems
cat data.json | jq .       # pretty-prints JSON without leaving your machine
```

**IDE format-on-save is the safest pattern of all** — it runs the formatter locally, automatically, before you even think about pasting code anywhere:

```json
// .vscode/settings.json (commit this to the repo)
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "[python]": {
    "editor.defaultFormatter": "ms-python.black-formatter"
  }
}
```

```bash
cat >> tool-risks.md << 'EOF'

## Online tools I will avoid when code contains secrets
- [ ] Never paste files containing credentials, tokens, or connection strings into online formatters
- [ ] Never use paste sites (pastebin, hastebin, gist) to share debugging snippets that include config
- [ ] Check the terms of service of any online tool before submitting non-trivial code
- [ ] Prefer locally installed formatters; configure format-on-save in the IDE
- [ ] If an online tool must be used (e.g., a demo), create a sanitised version of the file first
EOF
```

---

## Step 3 — Loggers: The Most Pervasive Leak Vector

Logging is where secrets escape most frequently in production systems, because log configuration is rarely reviewed with the same scrutiny as code.

**Pattern 1: Logging full request objects**

```python
import logging

logger = logging.getLogger(__name__)

def handle_payment(request):
    logger.debug("Processing payment request: %s", request.__dict__)
    #                                                  ↑
    # request.__dict__ includes headers:
    # {'Authorization': 'Bearer sk_live_4xKqR2mN8p...',
    #  'X-API-Key': 'prod-api-key-abc123', ...}
    # This goes to your log aggregator — Datadog, Splunk, CloudWatch
```

**Pattern 2: Logging exceptions that include environment**

```python
try:
    connect_to_database(DATABASE_URL)
except Exception as e:
    logger.exception("Database connection failed: %s", e)
    # SQLAlchemy error messages often include the full connection string:
    # "could not connect to postgresql://admin:SuperSecret@prod-db:5432/myapp"
```

**Pattern 3: Logging configuration at startup**

```python
# A well-intentioned "log your config so you can debug deploys" pattern
import os
logger.info("Starting with config: %s", dict(os.environ))
#                                          ↑ dumps ALL environment variables
```

**The safe logging pattern:**

```bash
cat > safe-app/safe_logging.py << 'EOF'
import logging
import os

logger = logging.getLogger(__name__)

# Define an explicit allowlist of headers safe to log
SAFE_HEADERS = {"Content-Type", "Accept", "User-Agent", "X-Request-ID"}

SAFE_CONFIG_KEYS = {"APP_ENV", "LOG_LEVEL", "PORT", "REGION"}


def log_request_safely(request):
    safe_headers = {k: v for k, v in request.headers.items() if k in SAFE_HEADERS}
    logger.info("Request received", extra={"headers": safe_headers, "path": request.path})


def log_config_at_startup():
    safe_config = {k: os.getenv(k, "not set") for k in SAFE_CONFIG_KEYS}
    logger.info("Application starting", extra={"config": safe_config})


def log_exception_safely(exc: Exception, context: str):
    # Log the exception type and a safe message — never the full string
    # which may contain connection strings
    logger.error(
        "Error in %s: %s",
        context,
        type(exc).__name__,
        exc_info=False,   # Do not include the full traceback with values
    )
EOF
```

> **Structured logging and secret scrubbing:** Tools like `structlog` (Python) and `pino` (Node.js) support processors that scrub known-sensitive field names before writing log output. This is worth configuring at the framework level rather than per-call.

---

## Step 4 — Error Pages and Stack Traces in Production

Every web framework has a "debug mode" that shows beautiful, detailed error pages with full stack traces and local variable values. These are indispensable in development — and catastrophically dangerous in production.

**Django example:**

```python
# settings.py — the single most dangerous misconfiguration in Django
DEBUG = True   # ← In production this shows full stack traces to users
               #   including the values of local variables
               #   which often include database URLs, tokens, and keys
```

When `DEBUG=True` in production, any unhandled exception shows a page like:

```
Environment Variables:
  DATABASE_URL  = postgresql://admin:SuperSecret123@prod-db.company.com/myapp
  STRIPE_KEY    = sk_live_4xKqR2mN8pLzT7vW...
  JWT_SECRET    = eyJhbGciOiJIUzI1NiIsInR5cCI6...

Local Variables in handle_request():
  api_key       = "sk_live_..."
  db_connection = <Connection to postgresql://admin:...>
```

**The safe configuration:**

```bash
cat > safe-app/check_production_safety.py << 'EOF'
"""
Run this script before deploying to production.
It checks for common dangerous misconfigurations.
"""
import os
import sys

errors = []
warnings = []

app_env = os.getenv("APP_ENV", "development")

if app_env == "production":
    # Debug mode must be off in production
    if os.getenv("DEBUG", "false").lower() in ("true", "1", "yes"):
        errors.append("DEBUG is enabled in production — this exposes secrets in error pages")

    # JWT secrets should not use development defaults
    jwt_secret = os.getenv("JWT_SECRET", "")
    if jwt_secret in ("", "dev-secret", "local-dev-secret-not-used-in-production"):
        errors.append("JWT_SECRET appears to be a development placeholder in production")

    # Database should not be localhost in production
    db_url = os.getenv("DATABASE_URL", "")
    if "localhost" in db_url or "127.0.0.1" in db_url:
        warnings.append("DATABASE_URL points to localhost — is this intentional in production?")

if errors:
    print("ERRORS (deployment should be blocked):")
    for e in errors:
        print(f"  ✗ {e}")
    sys.exit(1)

if warnings:
    print("WARNINGS (review before deploying):")
    for w in warnings:
        print(f"  ⚠ {w}")

print("✓ Production safety checks passed")
EOF
```

---

## Step 5 — CI/CD Pipelines: The Trusted But Leaky Environment

CI/CD pipelines are a high-risk leak surface because they sit at the intersection of secrets (needed to deploy) and logs (written to shared systems).

**Common CI leaks:**

```yaml
# GitHub Actions — dangerous patterns

# DANGEROUS: echoing secrets into logs
- run: echo "Deploying with key ${{ secrets.AWS_KEY }}"
#              ↑ GitHub masks known secrets in logs, but only if it knows the exact value

# DANGEROUS: printing environment variables
- run: env
#       ↑ Dumps all env vars including secrets to the log

# DANGEROUS: verbose curl with auth headers
- run: curl -v -H "Authorization: Bearer ${{ secrets.API_TOKEN }}" https://api.example.com
#             ↑ -v (verbose) prints all headers including Authorization
```

```yaml
# Safe equivalents

# SAFE: don't echo secrets; let the tool receive them directly
- name: Deploy
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  run: aws s3 sync ./dist s3://my-bucket/

# SAFE: redirect verbose output away from logs when auth is involved
- run: curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $TOKEN" https://api.example.com
```

```bash
cat > safe-app/.github/workflows/safe-ci.yml << 'EOF'
name: Safe CI Example

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        env:
          # Secrets injected as env vars — never interpolated directly in run: commands
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
          STRIPE_KEY: ${{ secrets.STRIPE_TEST_KEY }}
        run: python -m pytest

      - name: Check production safety
        run: |
          APP_ENV=production python safe-app/check_production_safety.py || true

      # NEVER DO THIS:
      # - run: echo "Using key ${{ secrets.STRIPE_KEY }}"
      # - run: env
      # - run: curl -v -H "Authorization: Bearer ${{ secrets.TOKEN }}" ...
EOF
```

---

## Step 6 — Docker: Secrets Baked Into Images

Docker adds several new secret leak surfaces that developers do not always anticipate.

**Docker build ARGs in image history:**

```dockerfile
# DANGEROUS — build args are stored in image history
FROM python:3.12-slim
ARG DATABASE_PASSWORD
ENV DATABASE_PASSWORD=${DATABASE_PASSWORD}   # baked into the layer permanently

RUN pip install -r requirements.txt
```

```bash
# Anyone with access to the image can read baked-in secrets
docker history my-app:latest --no-trunc
docker inspect my-app:latest | grep -i password
```

**The safe pattern — secrets at runtime, not build time:**

```dockerfile
# SAFE — no secrets at build time; received at container start
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
# Secrets arrive via docker run -e DATABASE_PASSWORD=... or docker-compose env_file
CMD ["gunicorn", "app:app"]
```

**Docker Compose secret exposure:**

```yaml
# DANGEROUS — secret visible in docker inspect output
services:
  web:
    environment:
      DATABASE_PASSWORD: SuperSecret123   # literal value in compose file

# SAFE — value from environment or .env file (gitignored)
services:
  web:
    environment:
      DATABASE_PASSWORD: ${DATABASE_PASSWORD}   # from shell env or .env
    env_file:
      - .env   # .env is gitignored; safe for local dev
```

---

## Step 7 — Complete the Tool Risks Reference

Return to `safety-journal/tool-risks.md` and fill in every section — including the new Online Tools section — with your own explanations and mitigations.

```bash
cat >> tool-risks.md << 'EOF'

## Safe Practices Summary

### Logging rules I will follow
- [ ] Never log raw request headers, bodies, or full exception messages
- [ ] Use an explicit allowlist for which fields are safe to log
- [ ] Never call `logger.info(dict(os.environ))` or equivalent
- [ ] Use structured logging with a secret-scrubbing processor
- [ ] Set log level to INFO in production (DEBUG prints more data)

### CI/CD rules I will follow
- [ ] Never interpolate secrets directly into `run:` commands
- [ ] Never run `env` or `printenv` in a CI step
- [ ] Never use `-v` (verbose) with tools that handle auth headers
- [ ] Always pass secrets as environment variables to the process, not as arguments
- [ ] Check that CI logs are not publicly visible for private repositories

### Docker rules I will follow
- [ ] Never use ARG to pass secrets into an image
- [ ] Never set ENV to a secret value in a Dockerfile
- [ ] Always use runtime environment injection (docker run -e or env_file)
- [ ] Run docker inspect on built images before pushing to verify no secrets
- [ ] Use .dockerignore to exclude .env files from the build context

EOF
```

```bash
cat tool-risks.md
```

In the final lab you will learn what to do when — despite all these precautions — a secret is found in the repository. Speed of response is the primary variable: the faster you act, the smaller the blast radius.
