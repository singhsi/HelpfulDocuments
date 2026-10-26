# Docker — New Developer Guide

A beginner-friendly guide to Docker and container fundamentals based on the Docker cheat sheet.

> **See also:** [commands.md](commands.md)

---

## Table of Contents

- [What is Docker?](#what-is-docker)
- [Key Concepts](#key-concepts)
  - [Images](#images)
  - [Containers](#containers)
  - [Docker Hub](#docker-hub)
- [How Docker Works](#how-docker-works)
- [Quick Start Workflow](#quick-start-workflow)
- [Help & Resources](#help--resources)

---

## What is Docker?

Docker provides the ability to package and run an application in a loosely isolated environment called a container.

The isolation and security allows you to run many containers simultaneously on a given host. Containers are lightweight and contain everything needed to run the application, so you do not need to rely on what is currently installed on the host. You can easily share containers while you work, and be sure that everyone you share with gets the same container that works in the same way.

---

## Key Concepts

### Images
Docker images are a lightweight, standalone, executable package of software that includes everything needed to run an application: code, runtime, system tools, system libraries, and settings.

### Containers
A container is a runtime instance of a Docker image. A container will always run the same, regardless of the infrastructure. Containers isolate software from its environment and ensure that it works uniformly despite differences between environments (for instance, between development and staging).

### Docker Hub
[Docker Hub](https://hub.docker.com) is a hosted repository service provided by Docker for finding and sharing container images with your team and the broader community.

---

## How Docker Works

```
Dockerfile ──docker build──▶ Image ──docker push──▶ Docker Hub / Registry
                               │
                          docker run
                               ▼
                           Container
```

1. **Build:** Use a `Dockerfile` to create a lightweight image (`docker build`).
2. **Share:** Store and share your image via Docker Hub or private registries (`docker push` / `docker pull`).
3. **Run:** Instantiate and execute containers locally or in production servers (`docker run`).

---

## Quick Start Workflow

### 1. Build an image
```bash
docker build -t my-app:1.0 .
```

### 2. Run the container
```bash
docker run -d --name my-app-instance -p 8080:80 my-app:1.0
```

### 3. Check status & logs
```bash
docker ps
docker logs -f my-app-instance
```

### 4. Stop and clean up
```bash
docker stop my-app-instance
docker rm my-app-instance
```

---

## Help & Resources

- **CLI Help:** Run `docker --help` or `docker <subcommand> --help`
- **Docker Desktop:** [https://docs.docker.com/desktop](https://docs.docker.com/desktop) (available for Mac, Linux, and Windows)
- **Official Docs:** [https://docs.docker.com](https://docs.docker.com)
- **Sample Projects:** [Awesome Compose Examples](https://github.com/docker/awesome-compose)
