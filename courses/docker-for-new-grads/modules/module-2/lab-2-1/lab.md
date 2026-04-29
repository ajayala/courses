# Writing Dockerfiles and Building Images

In this lab you will write a Dockerfile from scratch to package a real Python web application, build it into an image, run it as a container, and push it to Docker Hub. You will also learn how Docker's layer cache works — a detail that separates slow builds from fast ones and matters every time you push to CI.

**Prerequisites:** Lab 1-1 completed; Docker running; a Docker Hub account (free at hub.docker.com). `docker-course/` directory exists.

---

## Step 1 — The Application You Will Containerise

You will package a small Flask web API. Flask is a lightweight Python web framework — if you have not used it before, the code is simple enough to read without prior knowledge.

```bash
cd docker-course
mkdir flask-app && cd flask-app
```

Create the application files:

```bash
cat > app.py << 'EOF'
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route("/")
def index():
    return jsonify({"message": "Hello from Docker!", "env": os.getenv("APP_ENV", "development")})

@app.route("/health")
def health():
    return jsonify({"status": "ok"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
EOF
```

```bash
cat > requirements.txt << 'EOF'
flask==3.0.3
gunicorn==22.0.0
EOF
```

Without Docker, running this requires Python and pip installed locally, and any version mismatch breaks it. With Docker, none of that matters.

---

## Step 2 — Anatomy of a Dockerfile

A **Dockerfile** is a text file of instructions that Docker reads top-to-bottom to build an image. Each instruction creates a new **layer** stacked on top of the previous ones.

```dockerfile
# Every Dockerfile starts with a base image
FROM python:3.12-slim

# Set the working directory inside the container
WORKDIR /app

# Copy files from your machine into the image
COPY requirements.txt .

# Run a command during the build (installs dependencies)
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application code
COPY . .

# Document which port the app listens on (informational)
EXPOSE 5000

# The command to run when a container starts
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

| Instruction | What it does |
|-------------|-------------|
| `FROM` | Sets the base image — every image builds on another |
| `WORKDIR` | Sets the working directory for subsequent instructions |
| `COPY` | Copies files from build context (your machine) into the image |
| `RUN` | Executes a shell command during build and saves the result as a layer |
| `EXPOSE` | Documents (but does not publish) a port — informational only |
| `ENV` | Sets environment variables in the image |
| `CMD` | Default command to run when the container starts |

> **`RUN` vs `CMD`:** `RUN` happens at *build time* and produces a layer in the image. `CMD` happens at *run time* when someone starts a container. A common mistake is using `RUN` to start the application — it runs during the build and then stops.

---

## Step 3 — Write and Build the Dockerfile

```bash
cat > Dockerfile << 'EOF'
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
EOF
```

Build the image. The `.` at the end is the **build context** — the directory Docker sends to the daemon:

```bash
# -t = tag (name:version)
docker build -t flask-app:1.0.0 .
```

Watch the output carefully. Each step corresponds to a Dockerfile instruction:

```
Step 1/7 : FROM python:3.12-slim
Step 2/7 : WORKDIR /app
Step 3/7 : COPY requirements.txt .
Step 4/7 : RUN pip install --no-cache-dir -r requirements.txt
  ...installing packages...
Step 5/7 : COPY . .
Step 6/7 : EXPOSE 5000
Step 7/7 : CMD ["gunicorn"...]
Successfully built a1b2c3d4e5f6
Successfully tagged flask-app:1.0.0
```

```bash
docker images flask-app
```

---

## Step 4 — Run the Container and Test It

```bash
docker run -d -p 5000:5000 --name flask-app flask-app:1.0.0
```

Test the running app:

```bash
curl http://localhost:5000
```

```json
{"env": "development", "message": "Hello from Docker!"}
```

```bash
curl http://localhost:5000/health
```

```json
{"status": "ok"}
```

Pass an environment variable to the container at runtime — no rebuild needed:

```bash
docker stop flask-app && docker rm flask-app

docker run -d -p 5000:5000 \
  --name flask-app \
  --env APP_ENV=production \
  flask-app:1.0.0

curl http://localhost:5000
```

```json
{"env": "production", "message": "Hello from Docker!"}
```

> **Key insight:** Environment variables are how you configure the same image for different environments (dev, staging, prod) without building separate images. Build once, configure at runtime.

---

## Step 5 — How Layer Caching Works (and Why It Matters)

Docker caches each layer. If nothing has changed in a layer's instruction or its inputs, Docker reuses the cached layer instead of rebuilding it — making subsequent builds much faster.

The order of instructions matters enormously. Watch what happens:

```bash
# Edit app.py — just change the message string
sed -i 's/Hello from Docker!/Hello from Layer Cache!/' app.py

# Rebuild and observe which steps are cached
docker build -t flask-app:1.0.1 .
```

```
Step 1/7 : FROM python:3.12-slim        ---> Using cache
Step 2/7 : WORKDIR /app                ---> Using cache
Step 3/7 : COPY requirements.txt .     ---> Using cache
Step 4/7 : RUN pip install ...         ---> Using cache  ✓ (dependencies unchanged)
Step 5/7 : COPY . .                    ---> (cache miss — app.py changed)
Step 6/7 : EXPOSE 5000                 ---> (rebuilt)
Step 7/7 : CMD ...                     ---> (rebuilt)
```

Because `requirements.txt` is copied and installed *before* `COPY . .`, changing `app.py` does not invalidate the pip install layer. This is why the pattern `COPY requirements.txt → RUN pip install → COPY . .` is standard — dependencies change far less often than application code.

**The anti-pattern to avoid:**

```dockerfile
# BAD — copying everything first means any file change invalidates pip install
COPY . .
RUN pip install -r requirements.txt
```

```dockerfile
# GOOD — pip install is only re-run when requirements.txt changes
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

---

## Step 6 — Use .dockerignore to Keep Images Clean

Without a `.dockerignore`, Docker sends everything in the build context to the daemon — including `.git/`, `__pycache__/`, virtual environments, and any secrets you accidentally left in the directory.

```bash
cat > .dockerignore << 'EOF'
# Python
__pycache__/
*.pyc
*.pyo
.venv/
venv/
*.egg-info/

# Version control
.git/
.gitignore

# Secrets and local config
.env
*.env
.env.local

# Editor artifacts
.vscode/
.idea/
*.swp

# Tests and docs (not needed at runtime)
tests/
docs/
*.md
EOF
```

```bash
# Verify the ignore file is working — build context size should be tiny
docker build -t flask-app:1.0.2 . 2>&1 | grep "Sending build context"
```

> **Security note:** Never `COPY . .` without a `.dockerignore` in a real project. It is surprisingly easy to include API keys, database passwords, or SSH keys in an image and push it to a public registry.

---

## Step 7 — Tag and Push to Docker Hub

Tagging and pushing makes your image available for teammates, CI systems, and production servers to pull.

```bash
# Log in to Docker Hub
docker login

# Tag with your Docker Hub username
docker tag flask-app:1.0.0 YOUR_DOCKERHUB_USERNAME/flask-app:1.0.0
docker tag flask-app:1.0.0 YOUR_DOCKERHUB_USERNAME/flask-app:latest

# Push both tags
docker push YOUR_DOCKERHUB_USERNAME/flask-app:1.0.0
docker push YOUR_DOCKERHUB_USERNAME/flask-app:latest
```

```bash
# Verify: pull your image on a clean slate
docker rmi YOUR_DOCKERHUB_USERNAME/flask-app:latest
docker pull YOUR_DOCKERHUB_USERNAME/flask-app:latest
docker run -d -p 5001:5000 YOUR_DOCKERHUB_USERNAME/flask-app:latest
curl http://localhost:5001/health
```

Your application is now publicly runnable by anyone with `docker run YOUR_DOCKERHUB_USERNAME/flask-app:latest` — no Python, no pip, no setup required. In the next lab you will tackle the next real challenge: what happens when your app needs a database?
