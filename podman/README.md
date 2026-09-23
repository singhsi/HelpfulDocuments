# Podman & Containers — New Developer Guide

A beginner-friendly introduction to containers and Podman. No prior experience needed.

> **See also:** [commands.md](commands.md)

---

## Table of Contents

- [What is a Container?](#what-is-a-container)
- [Podman vs Docker](#podman-vs-docker)
- [Key Concepts](#key-concepts)
- [Your First Container](#your-first-container)
- [Writing a Dockerfile](#writing-a-dockerfile)
- [Best Practices](#best-practices)
- [Common Mistakes to Avoid](#common-mistakes-to-avoid)

---

## What is a Container?

A container is a lightweight, self-contained package that includes everything needed to run a piece of software — the code, runtime, libraries, and settings. Think of it like a shipping container: it works the same no matter which ship (machine) it's loaded onto.

**Why containers?**
- "It works on my machine" is no longer a problem — containers behave the same everywhere
- Faster to start than a virtual machine (seconds, not minutes)
- Isolated from the host system — containers can't interfere with each other
- Easy to share and deploy

**Container vs Virtual Machine:**

| | Container | Virtual Machine |
|---|---|---|
| Startup time | Seconds | Minutes |
| Size | Megabytes | Gigabytes |
| Shares host OS | Yes | No (has its own OS) |
| Isolation | Process-level | Full hardware-level |

---

## Podman vs Docker

Podman and Docker serve the same purpose — building and running containers — but Podman has some key differences:

| | Podman | Docker |
|---|---|---|
| Daemon (background service) | No daemon — each container is its own process | Requires a running Docker daemon |
| Root access | Runs rootless by default (more secure) | Traditionally requires root |
| Commands | Almost identical to Docker | — |
| Compose support | `podman-compose` | `docker-compose` |

> **Tip:** Almost every `docker` command you find online will work with Podman — just replace `docker` with `podman`.

---

## Key Concepts

**Image**
A read-only template used to create containers. Think of it as a snapshot of a filesystem with your app and all its dependencies baked in. You build images from a `Dockerfile`.

**Container**
A running instance of an image. You can run many containers from the same image, just like you can open many windows from the same app.

**Dockerfile**
A plain text file with step-by-step instructions for building an image. Each instruction adds a layer to the image.

**Registry**
A place to store and share images. The most common public registry is [Docker Hub](https://hub.docker.com). IBM uses `icr.io` (IBM Container Registry).

**Port mapping**
Containers are isolated, so their ports aren't accessible from your machine by default. Port mapping connects a port on your machine to a port inside the container.
```
podman run -p 8080:3000 my-app
#              ^     ^
#        host port   container port
```
Visiting `localhost:8080` on your machine will reach port `3000` inside the container.

**Volume**
A way to persist data outside the container. Containers are ephemeral — when a container is removed, its internal data is gone. Volumes survive container restarts and removals.
```
podman run -v /my/host/folder:/app/data my-app
```

---

## Your First Container

Try running an existing image from a registry — no Dockerfile needed:

```
# pull and run an nginx web server
podman run -d -p 8080:80 --name my-nginx nginx

# open http://localhost:8080 in your browser — you should see the nginx welcome page

# stop it when done
podman stop my-nginx

# remove the container
podman rm my-nginx
```

Flags used above:
- `-d` — run in the background (detached mode)
- `-p 8080:80` — map port 8080 on your machine to port 80 in the container
- `--name my-nginx` — give the container a friendly name instead of a random one

---

## Writing a Dockerfile

A `Dockerfile` lives in the root of your project. Here's a simple example for a Node.js app:

```dockerfile
# 1. start from an official base image
FROM node:18-alpine

# 2. set the working directory inside the container
WORKDIR /app

# 3. copy dependency files first (so Docker can cache this layer)
COPY package*.json ./

# 4. install dependencies
RUN npm install

# 5. copy the rest of your source code
COPY . .

# 6. tell the container which port your app listens on
EXPOSE 3000

# 7. the command to start your app
CMD ["node", "server.js"]
```

Build and run it:
```
podman build -t my-node-app .
podman run -p 3000:3000 my-node-app
```

---

## Best Practices

#### use specific base image versions, not `latest`
`latest` changes over time and can break your build unexpectedly.
```dockerfile
# avoid
FROM node:latest

# prefer
FROM node:18-alpine
```

#### prefer smaller base images
`alpine` variants (e.g. `node:18-alpine`, `python:3.11-slim`) are much smaller than full OS images — faster to build, pull, and deploy.

#### copy `package.json` before source code
Docker/Podman caches each layer. If you copy your source code before installing dependencies, the dependency install re-runs every time any source file changes. Copying `package.json` first means the install layer is only invalidated when dependencies actually change.

#### never store secrets in a Dockerfile
Don't hardcode passwords, API keys, or tokens. Use environment variables passed at runtime instead:
```
podman run -e DB_PASSWORD=secret my-app
```
Or use a `.env` file (and add it to `.gitignore`):
```
podman run --env-file .env my-app
```

#### use `.dockerignore` to keep images small
Just like `.gitignore`, a `.dockerignore` file prevents unnecessary files from being copied into the image:
```
node_modules
.env
.git
*.log
dist
```

#### one process per container
Each container should do one thing. Don't run your database and your web server in the same container. Keep them separate and connect them via networking.

#### always name your containers
Random container names are hard to work with. Use `--name` so you can reference them easily:
```
podman run --name my-api my-api-image
```

#### clean up regularly
Unused images and containers consume disk space.
```
podman system prune          # remove stopped containers and unused images
podman image prune -a        # remove all unused images
```

---

## Common Mistakes to Avoid

| Mistake | Why it's a problem | Fix |
|---|---|---|
| Using `FROM ... :latest` | Build breaks when the image updates | Pin to a specific version |
| Committing `.env` to git | Exposes secrets | Add `.env` to `.gitignore` |
| Installing everything as root | Security risk | Add a non-root user in your Dockerfile |
| Forgetting `.dockerignore` | Bloated images, slow builds | Always add one |
| Running `npm install` after `COPY . .` | Cache busted on every code change | Copy `package.json` first |
| Storing data inside a container | Data lost when container is removed | Use volumes for persistent data |
