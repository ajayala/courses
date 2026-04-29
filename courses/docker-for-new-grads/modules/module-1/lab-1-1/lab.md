# Why Containers Exist and How Docker Works

In this lab you will learn the problem Docker was built to solve, understand how containers differ from virtual machines, and run your first containers — from a simple hello-world to an interactive shell inside a real Linux environment. By the end you will be comfortable with the container lifecycle and the handful of commands you will use every single day.

**Prerequisites:** Docker Desktop installed and running (check with `docker --version`). A terminal. No prior Docker experience required.

---

## Step 1 — The Problem: "It Works on My Machine"

Picture this: you spend a week building a feature that works perfectly on your laptop. You push the code. Your teammate pulls it. It crashes immediately. The error is a version mismatch for a library you did not know was installed globally on your machine but not on theirs.

This is the oldest problem in software deployment, and it happens because code runs inside an *environment* — a specific combination of:

```
Operating system version
Runtime version (Python 3.11 vs 3.9, Node 20 vs 18)
Installed libraries and their exact versions
Environment variables
File system layout and permissions
```

Before Docker, teams tried to solve this with documentation ("install these 14 things in this order"), virtual machines, or scripts that were always slightly wrong on someone's laptop.

Docker's answer: **ship the environment with the code**.

```bash
# Verify Docker is installed and running
docker --version
docker info | head -10
```

> **Mental model:** A Docker container is like a sealed lunchbox. Everything your application needs — the runtime, the libraries, the config — is packed inside. When someone else opens the lunchbox, they get exactly what you packed, regardless of what is in their kitchen.

---

## Step 2 — Virtual Machines vs Containers

Both VMs and containers solve the environment problem, but they do it very differently.

```
Virtual Machine                    Container
──────────────────────────         ──────────────────────────
  Your App                           Your App
  Guest OS (full Linux kernel)       Container Runtime
  Hypervisor (VMware, VirtualBox)    Host OS Kernel (shared)
  Host Hardware                      Host Hardware

Size:    Gigabytes                 Size:    Megabytes
Startup: Minutes                   Startup: Milliseconds
Isolation: Strong (full OS)        Isolation: Good (kernel namespaces)
```

Containers share the host OS kernel rather than emulating a full operating system. This makes them:
- **Much faster to start** — a container spins up in under a second
- **Much smaller** — a Python container is ~50 MB; a Python VM is ~1 GB
- **More resource-efficient** — you can run dozens of containers where you could only run a few VMs

| Use case | Better choice |
|----------|--------------|
| Running a web service in production | Container |
| Testing your app in isolated environments | Container |
| Running a different OS entirely (Windows on Linux) | VM |
| Strong security isolation (untrusted workloads) | VM |

---

## Step 3 — Docker Architecture: Four Things to Know

Docker has four core concepts. Learn these and the rest of Docker falls into place.

```
┌─────────────────────────────────────────────────────────┐
│                   Docker Architecture                   │
│                                                         │
│  Registry (Docker Hub, GHCR)                            │
│    └── stores Images                                    │
│                                                         │
│  Docker Daemon (runs on your machine)                   │
│    ├── pulls Images from registries                     │
│    ├── builds Images from Dockerfiles                   │
│    └── runs Images as Containers                        │
│                                                         │
│  Docker Client (the `docker` CLI you type into)         │
│    └── sends commands to the Daemon                     │
└─────────────────────────────────────────────────────────┘
```

| Concept | What it is | Analogy |
|---------|-----------|---------|
| **Image** | A read-only blueprint for a container | A recipe |
| **Container** | A running instance of an image | A meal made from the recipe |
| **Dockerfile** | Instructions for building an image | The recipe card |
| **Registry** | A store of published images | A recipe book library |

You can run many containers from the same image, just as you can cook the same recipe many times.

---

## Step 4 — Pull and Run Your First Containers

```bash
# Pull and run the simplest possible container
docker run hello-world
```

Docker pulls the `hello-world` image from Docker Hub (the default registry), creates a container, runs it, and prints a message. The container then exits — it did its job.

Now run something more interesting — an interactive shell inside an Ubuntu container:

```bash
docker run --interactive --tty ubuntu:24.04 bash
```

You are now *inside* a container. It looks like a Linux terminal because it is one:

```bash
# Inside the container
cat /etc/os-release
ls /
echo "I am container $(hostname)"
exit
```

When you type `exit`, the container stops. It was a complete, isolated Ubuntu environment that appeared in under a second and cost almost no disk space compared to a real Ubuntu VM.

> **Shorthand:** `--interactive --tty` is almost always written as `-it`. You will see `-it` everywhere — it means "give me an interactive terminal."

---

## Step 5 — Explore a Running Container

Run a container in the *background* (detached mode) and explore it while it is running.

```bash
# -d = detached (run in background), -p = publish port (host:container)
docker run -d -p 8080:80 --name my-nginx nginx
```

```bash
# Verify it is running
docker ps
```

```
CONTAINER ID   IMAGE   COMMAND                  STATUS         PORTS                  NAMES
a1b2c3d4e5f6   nginx   "/docker-entrypoint.…"  Up 3 seconds   0.0.0.0:8080->80/tcp   my-nginx
```

Open a browser and visit `http://localhost:8080` — you are served by nginx running inside the container.

```bash
# View the container's logs
docker logs my-nginx

# Open a shell inside the running container
docker exec -it my-nginx bash

# Inside the container — explore the nginx config
cat /etc/nginx/nginx.conf
exit

# Inspect all container metadata as JSON
docker inspect my-nginx | head -40
```

> **`docker exec` vs `docker run`:** `docker run` starts a *new* container from an image. `docker exec` runs a command *inside an already-running* container. You will use `exec` for debugging — it is the equivalent of SSH-ing into a server.

---

## Step 6 — The Container Lifecycle

Every container moves through a defined set of states. Understanding this prevents confusion when a container "disappears."

```
docker pull   →  Image downloaded to local cache
docker run    →  Container created + started
docker stop   →  Container stopped (state preserved)
docker start  →  Container restarted from stopped state
docker rm     →  Container deleted (state gone permanently)
docker rmi    →  Image deleted from local cache
```

```bash
# Stop the nginx container
docker stop my-nginx

# It still exists — just stopped
docker ps -a

# Start it again
docker start my-nginx

# Now remove it (must be stopped first, or use -f to force)
docker stop my-nginx
docker rm my-nginx

# Check it is gone
docker ps -a
```

```bash
# List all images on your machine
docker images

# Remove an image you no longer need
docker rmi hello-world
```

> **Disk hygiene tip:** Containers and images accumulate fast. `docker system prune` removes all stopped containers, dangling images, and unused networks in one command. Run it regularly on development machines.

---

## Step 7 — Your Docker Cheat Sheet

Create a reference file you can return to throughout this course.

```bash
mkdir docker-course && cd docker-course
cat > cheatsheet.md << 'EOF'
# Docker Cheat Sheet

## Images
docker pull <image>            # Download an image
docker images                  # List local images
docker rmi <image>             # Remove an image
docker build -t <name> .       # Build image from Dockerfile in current dir

## Containers
docker run <image>             # Create and start a container
docker run -it <image> bash    # Interactive shell
docker run -d -p 8080:80 <img> # Detached, port-mapped
docker ps                      # List running containers
docker ps -a                   # List all containers (including stopped)
docker stop <name/id>          # Stop a container
docker start <name/id>         # Start a stopped container
docker rm <name/id>            # Remove a stopped container
docker logs <name/id>          # View container output
docker exec -it <name> bash    # Open shell in running container
docker inspect <name/id>       # Full container metadata

## Cleanup
docker system prune            # Remove stopped containers + dangling images
docker system prune -a         # Remove everything unused (including images)
EOF
```

In the next lab you will write your own Dockerfile to package a real application — the skill that transforms Docker from a tool you consume into one you create with.
