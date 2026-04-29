# Docker in CI/CD and the Professional Workflow

In this lab you will wire your Docker image into a GitHub Actions CI/CD pipeline — so every `git push` automatically builds, tests, and publishes the image. You will also learn restart policies, resource limits, and how to read a real team's Docker setup when you join a new project. This is the workflow you will live in as a professional engineer.

**Prerequisites:** Lab 3-1 completed; `docker-course/flask-app/Dockerfile.production` exists. A GitHub repository and Docker Hub account.

---

## Step 1 — How Docker Fits into a CI/CD Pipeline

Without Docker, CI/CD pipelines install dependencies directly on the CI runner — a fragile process that breaks when runner environments update or differ from production.

With Docker:

```
Developer pushes code
        │
        ▼
CI runner pulls code
        │
        ▼
Docker builds image (from Dockerfile — identical to production)
        │
        ▼
Docker runs tests inside the image (same environment as production)
        │
        ▼
If tests pass: image is tagged and pushed to registry
        │
        ▼
Production server pulls the new image and restarts the container
```

The key insight: the Docker image is the **deployment artefact**. The same image that passes tests is the exact bytes that run in production — no "it passed tests but broke in prod because the environments differed."

---

## Step 2 — Set Up Your GitHub Repository

```bash
cd docker-course/flask-app
git init
git add .
git commit -m "Initial Flask app with production Dockerfile"
```

Add your GitHub repository as the remote (create an empty repo at github.com first):

```bash
git remote add origin git@github.com:YOUR_USERNAME/flask-docker-app.git
git branch -M main
git push -u origin main
```

Store your Docker Hub credentials as GitHub Secrets:

1. Go to your GitHub repository → **Settings → Secrets and variables → Actions**
2. Add `DOCKERHUB_USERNAME` — your Docker Hub username
3. Add `DOCKERHUB_TOKEN` — a Docker Hub access token (create at hub.docker.com → Account Settings → Security → New Access Token)

> **Why an access token and not your password?** Access tokens can be scoped (read-only, write, delete) and revoked individually without changing your password. Always use tokens for CI systems — never raw passwords.

---

## Step 3 — Write the GitHub Actions Workflow

```bash
mkdir -p .github/workflows

cat > .github/workflows/docker-publish.yml << 'EOF'
name: Build, Test, and Publish Docker Image

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/flask-app

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build image for testing
        uses: docker/build-push-action@v5
        with:
          context: .
          file: Dockerfile.production
          push: false
          load: true
          tags: flask-app:test
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Run health check test
        run: |
          docker run -d -p 5000:5000 --name test-container flask-app:test
          sleep 5
          curl --fail http://localhost:5000/health || (docker logs test-container && exit 1)
          docker stop test-container

  publish:
    runs-on: ubuntu-latest
    needs: build-and-test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extract metadata (tags and labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: Dockerfile.production
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
EOF
```

```bash
git add .github/
git commit -m "Add GitHub Actions CI/CD pipeline for Docker image"
git push
```

> **Reading this workflow:** The `build-and-test` job runs on every push and pull request. The `publish` job runs only on pushes to `main` (not on PRs) and only if `build-and-test` passed (`needs: build-and-test`). This prevents broken images from reaching the registry.

---

## Step 4 — Restart Policies and Resource Limits

In production, containers crash. A restart policy tells Docker what to do when that happens.

```bash
# No restart (default) — stays dead after crash
docker run -d flask-app:production

# Restart unless manually stopped — good for most services
docker run -d --restart unless-stopped flask-app:production

# Always restart — includes after Docker daemon restart (e.g. server reboot)
docker run -d --restart always flask-app:production

# Retry up to 5 times with backoff — good for services with startup dependencies
docker run -d --restart on-failure:5 flask-app:production
```

| Policy | Restarts on crash | Restarts on daemon restart | Use case |
|--------|------------------|--------------------------|---------|
| `no` | No | No | One-off tasks |
| `on-failure[:N]` | Yes (up to N times) | No | Services with dependencies |
| `unless-stopped` | Yes | Yes (unless manually stopped) | Most production services |
| `always` | Yes | Yes (always) | Critical infrastructure |

**Resource limits** prevent a runaway container from consuming all host resources:

```bash
docker run -d \
  --restart unless-stopped \
  --memory 256m \
  --memory-swap 256m \
  --cpus 0.5 \
  -p 5000:5000 \
  --name flask-prod \
  flask-app:production
```

```bash
# Monitor resource usage in real time
docker stats flask-prod
```

```
CONTAINER ID   NAME         CPU %     MEM USAGE / LIMIT   MEM %
a1b2c3d4e5f6   flask-prod   0.12%     24.5MiB / 256MiB    9.6%
```

In Compose, add these to the service definition:

```bash
cat >> ../compose-app/docker-compose.yml << 'EOF'

# Add under the web service:
# deploy:
#   resources:
#     limits:
#       memory: 256m
#       cpus: '0.5'
#     reservations:
#       memory: 128m
EOF
```

---

## Step 5 — Reading a Real Team's Docker Setup

When you join a new team, you will encounter an existing Docker setup. Here is how to read it quickly.

```bash
# What images does the project use?
grep "image:" docker-compose.yml

# What does the project build?
grep "build:" docker-compose.yml

# What ports are exposed?
grep "ports:" -A 2 docker-compose.yml

# What environment variables does it expect?
grep -E "^\s+-\s+[A-Z_]+=" docker-compose.yml

# What volumes are used for persistence?
grep "volumes:" -A 5 docker-compose.yml

# Are there multiple Compose files (common pattern: base + override)?
ls docker-compose*.yml
```

Common patterns you will see:

| Pattern | What it means |
|---------|--------------|
| `docker-compose.yml` + `docker-compose.override.yml` | Base config + local dev overrides (override is auto-applied) |
| `docker-compose.yml` + `docker-compose.prod.yml` | Base config + production settings (applied with `-f`) |
| `Dockerfile` + `Dockerfile.dev` | Separate images for production vs local development |
| `.env.example` in the repo | Template for your local `.env` — copy it and fill in values |

```bash
# Apply multiple compose files
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Step 6 — Your Docker Mental Model

Close the course by writing a summary that consolidates your mental model. This is the kind of document you would write for a new teammate.

```bash
cd docker-course
cat > docker-mental-model.md << 'EOF'
# Docker Mental Model

## The Core Problem Docker Solves
[Write 1-2 sentences in your own words]

## The Four Core Concepts
- Image:
- Container:
- Dockerfile:
- Registry:

## Daily Workflow (local development)
1.
2.
3.

## CI/CD Workflow (what happens on git push)
1.
2.
3.
4.

## The Three Things That Make an Image Production-Ready
1.
2.
3.

## Commands I Will Use Every Day
[List 5-8 commands with one-line descriptions]

## Questions I Still Have
-
EOF
```

Fill in every section. This document is for you — write it at the level of detail that would have helped you at the start of this course.

```bash
cat docker-mental-model.md
```

You have completed the Docker for New Graduates course. You can now explain why containers exist, write a production-quality Dockerfile, compose multi-service applications, optimise images for size and security, and plug Docker into a CI/CD pipeline. Every team you join will use at least some of these tools — and now you will not need anyone to explain them to you from scratch.
