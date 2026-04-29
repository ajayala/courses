# Multi-Stage Builds and Image Optimisation

In this lab you will learn why image size matters in production, use multi-stage builds to separate the build environment from the runtime environment, add a non-root user for security, and scan your image for known vulnerabilities. These techniques are the difference between an image a team is proud to ship and one that causes problems at 2 AM.

**Prerequisites:** Lab 2-2 completed; `docker-course/flask-app/` with a working Dockerfile.

---

## Step 1 — Why Image Size Matters

A bloated image is not just aesthetically bad — it has real operational consequences.

| Problem | Why it happens | Real impact |
|---------|---------------|------------|
| Slow CI/CD pipelines | Large images take longer to push and pull | Developers wait longer for feedback |
| High cloud storage costs | Registry storage is billed by GB | Unnecessary ongoing spend |
| Larger attack surface | More packages = more potential vulnerabilities | Security incidents |
| Slow cold starts | Kubernetes pulls images on new nodes | Traffic spikes take longer to absorb |

Check the size of your current image:

```bash
docker images flask-app
```

```
REPOSITORY   TAG       IMAGE ID       SIZE
flask-app    1.0.0     a1b2c3d4e5f6   156MB
```

156 MB for a tiny Flask app seems large. Here is why: the `python:3.12-slim` base image includes the full Python toolchain — compilers, headers, build tools — that are only needed during installation, not at runtime.

```bash
# See what is taking up space inside the image
docker run --rm flask-app:1.0.0 du -sh /usr/local/lib/python3.12 | head -5
docker run --rm flask-app:1.0.0 pip list | wc -l
```

---

## Step 2 — Layer Optimisation: Quick Wins

Before reaching for multi-stage builds, several single-stage optimisations are worth knowing.

**Combine RUN commands to reduce layer count:**

```dockerfile
# BAD: three layers, each storing intermediate state
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# GOOD: one layer, cleanup happens in the same step
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

**Use specific base image tags — never `latest`:**

```dockerfile
# BAD: what version is this? Changes without warning.
FROM python:latest

# GOOD: reproducible, auditable, pinned to a specific release
FROM python:3.12.3-slim-bookworm
```

**Use `-slim` or `-alpine` variants:**

```
python:3.12          ~1.0 GB   Full Debian — compilers, tools, headers
python:3.12-slim     ~150 MB   Minimal Debian — common runtime deps only
python:3.12-alpine   ~55 MB    Alpine Linux — extremely minimal (but differences in libc)
```

> **Alpine caveats:** Alpine uses `musl` libc instead of `glibc`. Some Python packages (especially those with C extensions like numpy, pandas) do not have Alpine wheels and must compile from source — slow and sometimes broken. Use `-slim` unless you know Alpine works for your dependencies.

---

## Step 3 — Multi-Stage Builds

A **multi-stage build** uses multiple `FROM` instructions in one Dockerfile. Each stage is independent — you can copy artefacts from one stage to another, leaving behind everything that was only needed for the build.

The pattern for Python:

```
Stage 1 (builder):  Full build environment — install gcc, build wheels, compile extensions
Stage 2 (runtime):  Minimal image — copy only the installed packages and app code
```

Create the optimised Dockerfile:

```bash
cd docker-course/flask-app

cat > Dockerfile.optimised << 'EOF'
# ── Stage 1: Build ──────────────────────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /build

# Install build dependencies (gcc for C extensions)
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

# Install into a local directory (not system Python) so we can copy it cleanly
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt


# ── Stage 2: Runtime ────────────────────────────────────────────────────────
FROM python:3.12-slim AS runtime

WORKDIR /app

# Copy only the installed packages from the builder stage
COPY --from=builder /install /usr/local

# Copy only the application code — no build tools, no cache
COPY app.py .

# Run as a non-root user for security (see Step 4)
RUN useradd --no-create-home --shell /bin/false appuser
USER appuser

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
EOF
```

Build and compare sizes:

```bash
docker build -t flask-app:slim -f Dockerfile.optimised .
docker images flask-app
```

```
REPOSITORY   TAG     SIZE
flask-app    1.0.0   156MB
flask-app    slim     89MB
```

The runtime image carries no compilers, no pip cache, no build headers — only what is needed to run the application.

---

## Step 4 — Running as a Non-Root User

By default, processes inside containers run as `root`. This means a container escape or code injection vulnerability has root access to your host system.

```bash
# Verify the current image runs as root
docker run --rm flask-app:1.0.0 whoami
```

```
root
```

The `Dockerfile.optimised` already adds a non-root user. Verify it:

```bash
docker run --rm flask-app:slim whoami
```

```
appuser
```

```bash
# Confirm the process cannot write to system directories
docker run --rm flask-app:slim bash -c "touch /etc/passwd 2>&1 || echo 'Permission denied — non-root user working correctly'"
```

> **When non-root matters:** Most production Kubernetes clusters enforce a `runAsNonRoot` security policy — images that run as root are rejected. Building this habit now saves you from surprises when you join a team with security policies.

---

## Step 5 — Scanning for Vulnerabilities with Docker Scout

`docker scout` is Docker's built-in vulnerability scanner. It checks your image against known CVE databases and tells you which packages are the source of each vulnerability.

```bash
# Scan your image (requires Docker Desktop or Scout CLI)
docker scout cves flask-app:slim
```

```
✓ SBOM of image already cached, 87 packages indexed

  Your image  flask-app:slim  │  0C  2H  5M  12L
                               │  Critical  High  Medium  Low
```

```bash
# Get actionable recommendations (newer base images that fix CVEs)
docker scout recommendations flask-app:slim
```

```bash
# Compare the two versions side by side
docker scout compare flask-app:slim --to flask-app:1.0.0
```

> **What to do with scan results:** Critical and High severity CVEs in packages your app actually uses warrant immediate action — update the base image or the package. Medium and Low are worth tracking but rarely block a release. CVEs in packages you do not use (dev tools in a build stage that did not make it into the runtime image) can be noted and ignored.

---

## Step 6 — Build the Final Optimised Image

Apply all the optimisations together: pinned base image, combined RUN commands, multi-stage build, non-root user, and `.dockerignore`.

```bash
cat > Dockerfile.production << 'EOF'
# ── Stage 1: Build ──────────────────────────────────────────────────────────
FROM python:3.12.3-slim-bookworm AS builder

WORKDIR /build

RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt


# ── Stage 2: Runtime ────────────────────────────────────────────────────────
FROM python:3.12.3-slim-bookworm AS runtime

WORKDIR /app

COPY --from=builder /install /usr/local
COPY app.py .

RUN useradd --no-create-home --shell /bin/false appuser && \
    chown -R appuser:appuser /app

USER appuser

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')"

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
EOF
```

```bash
docker build -t flask-app:production -f Dockerfile.production .
docker images flask-app
```

```bash
# Run it and verify the health check kicks in
docker run -d -p 5000:5000 --name flask-prod flask-app:production
sleep 5
docker inspect flask-prod --format '{{.State.Health.Status}}'
```

```
healthy
```

In the next lab you will connect all of this to a real CI/CD pipeline — so that every `git push` automatically builds, tests, and publishes your image.
