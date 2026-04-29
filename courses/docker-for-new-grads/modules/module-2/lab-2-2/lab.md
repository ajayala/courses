# Volumes, Networking, and Docker Compose

In this lab you will solve the two problems that arise as soon as you try to do something real with Docker: data disappears when containers stop, and multiple containers need to talk to each other. You will also learn Docker Compose — the tool that replaces long `docker run` commands with a single declarative file, and the thing most professional teams actually use day-to-day.

**Prerequisites:** Lab 2-1 completed; `docker-course/flask-app/` with a working Dockerfile.

---

## Step 1 — The Data Problem: Containers Are Ephemeral

By default, a container's filesystem is temporary. When the container is removed, everything written inside it is gone — including database rows, uploaded files, and log files.

Prove it:

```bash
# Start a container and write a file inside it
docker run --name temp-test ubuntu:24.04 bash -c "echo 'important data' > /data/notes.txt"

# The container has exited — remove it
docker rm temp-test

# Start a fresh container from the same image — the file is gone
docker run --rm ubuntu:24.04 bash -c "cat /data/notes.txt 2>&1 || echo 'File not found'"
```

```
cat: /data/notes.txt: No such file or directory
File not found
```

This is by design — immutability makes containers predictable and reproducible. But it means you need an explicit mechanism for data that must outlive a container: **volumes**.

---

## Step 2 — Named Volumes: Persistent Storage

A **named volume** is a directory managed by Docker that lives outside the container filesystem. It persists until you explicitly delete it.

```bash
# Create a named volume
docker volume create app-data

# Mount it into a container at /data
docker run --rm \
  --volume app-data:/data \
  ubuntu:24.04 \
  bash -c "echo 'important data' > /data/notes.txt && echo 'Written!'"

# Start a completely new container — the data survives
docker run --rm \
  --volume app-data:/data \
  ubuntu:24.04 \
  cat /data/notes.txt
```

```
important data
```

```bash
# Inspect where Docker stores the volume on the host
docker volume inspect app-data
```

| Volume type | Syntax | Use case |
|-------------|--------|---------|
| **Named volume** | `--volume mydata:/container/path` | Databases, persistent app state |
| **Bind mount** | `--volume /host/path:/container/path` | Dev: mount source code for live reload |
| **tmpfs mount** | `--tmpfs /container/path` | Secrets, temp data — in memory only |

> **Bind mounts in development:** Mounting your source code directory into a container means edits on your machine appear inside the container immediately — no rebuild needed. This is how most teams run local development servers.

```bash
# Example: bind mount for live development
docker run -d -p 5000:5000 \
  --volume "$(pwd)/docker-course/flask-app:/app" \
  flask-app:1.0.0
# Edit app.py on your machine — gunicorn auto-reloads inside the container
```

---

## Step 3 — Container Networking: How Containers Talk

By default, each container gets its own isolated network namespace. Containers cannot reach each other by default — you have to wire them together.

Docker provides several network drivers:

```
bridge (default)   — containers on same host communicate via a virtual switch
host               — container shares the host's network stack directly
none               — no networking at all
custom bridge      — user-defined bridge; containers reference each other by name
```

The key thing to know: **on a user-defined bridge network, containers can reach each other by container name as the hostname.** This is how your Flask app finds the database — not by IP address (which changes every restart) but by name.

```bash
# Create a custom network
docker network create app-network

# Start a Redis container on it
docker run -d \
  --name redis \
  --network app-network \
  redis:7-alpine

# Start another container on the same network and ping Redis by name
docker run --rm \
  --network app-network \
  alpine \
  ping -c 3 redis
```

```
PING redis (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.123 ms
```

The container name `redis` resolved as a hostname automatically — no IP addresses, no config files.

---

## Step 4 — Docker Compose: Orchestrating Multiple Containers

Running multiple `docker run` commands with all the right flags is error-prone and hard to share. **Docker Compose** lets you define your entire multi-container application in a single YAML file and manage it with simple commands.

```bash
cd docker-course
mkdir compose-app && cd compose-app
```

Create the Compose file:

```bash
cat > docker-compose.yml << 'EOF'
services:
  web:
    build: ../flask-app
    ports:
      - "5000:5000"
    environment:
      - APP_ENV=development
      - DATABASE_URL=postgresql://user:password@db:5432/appdb
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: appdb
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-network

volumes:
  db-data:

networks:
  app-network:
    driver: bridge
EOF
```

> **Reading this file:** `web` is your Flask app, built from the local Dockerfile. `db` is a Postgres container pulled from Docker Hub. They are both on `app-network` so `web` can reach `db` by hostname. `db-data` is a named volume so the database survives container restarts.

---

## Step 5 — Start and Manage the Compose Stack

```bash
# Start all services — builds images if needed, then starts containers
docker compose up --build

# Or run in the background
docker compose up --build -d
```

```bash
# Check the status of all services
docker compose ps
```

```
NAME              IMAGE         STATUS         PORTS
compose-app-db-1  postgres:16   Up (healthy)   5432/tcp
compose-app-web-1 compose-app   Up             0.0.0.0:5000->5000/tcp
```

```bash
# Stream logs from all services
docker compose logs -f

# Logs from one service only
docker compose logs -f web

# Open a shell in the web container
docker compose exec web bash

# Run a one-off command (e.g. run database migrations)
docker compose run --rm web python manage.py migrate
```

```bash
# Stop everything (containers stop, data is preserved in volumes)
docker compose down

# Stop AND delete volumes (destroys database data — use with care!)
docker compose down -v
```

| Command | What it does |
|---------|-------------|
| `docker compose up -d` | Start all services in background |
| `docker compose down` | Stop and remove containers (volumes preserved) |
| `docker compose ps` | List service status |
| `docker compose logs -f` | Stream all logs |
| `docker compose exec <svc> <cmd>` | Run a command in a running service |
| `docker compose run --rm <svc> <cmd>` | Run a one-off command in a new container |

---

## Step 6 — Environment Variables and the .env File

Hardcoding passwords in `docker-compose.yml` is a security risk — the file goes into version control. Use a `.env` file instead.

```bash
cat > .env << 'EOF'
POSTGRES_USER=user
POSTGRES_PASSWORD=supersecretpassword
POSTGRES_DB=appdb
APP_ENV=development
EOF

# IMPORTANT: add to .gitignore — never commit this file
echo ".env" >> .gitignore
```

Update `docker-compose.yml` to use the variables:

```bash
cat > docker-compose.yml << 'EOF'
services:
  web:
    build: ../flask-app
    ports:
      - "5000:5000"
    environment:
      - APP_ENV=${APP_ENV}
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-network

volumes:
  db-data:

networks:
  app-network:
    driver: bridge
EOF
```

```bash
# Docker Compose reads .env automatically — verify variable substitution
docker compose config | grep DATABASE_URL
```

> **`.env` vs environment variables in CI:** In production and CI, secrets are injected as real environment variables (from GitHub Secrets, AWS Secrets Manager, Vault, etc.) — not from a `.env` file. The `.env` file is a *local development convenience only*.

In the next lab you will learn how to dramatically reduce image sizes with multi-stage builds — a technique that matters for deployment speed, security, and cloud costs.
