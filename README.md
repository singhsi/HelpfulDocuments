# HelpfulDocuments

A personal reference repository for tools and technologies used in day-to-day development. Each folder contains a beginner-friendly guide (`README.md`) and a quick-lookup command reference (`commands.md`).

---

## Contents

### [`git/`](git/)
Git version control — concepts, workflows, and commands.

| File | Description |
|---|---|
| [README.md](git/README.md) | Merge vs rebase, commit best practices, branch naming, remotes, log tricks, `.gitignore` |
| [commands.md](git/commands.md) | Quick-lookup reference for common Git commands |
| [stash.md](git/stash.md) | All stash commands, including applying a stash to a new branch |
| [undoing.md](git/undoing.md) | Every way to undo something in Git — decision table, `reset`, `revert`, `reflog`, removing secrets from history |

---

### [`podman/`](podman/)
Container basics with Podman (compatible with Docker).

| File | Description |
|---|---|
| [README.md](podman/README.md) | What containers are, Podman vs Docker, key concepts, writing Dockerfiles, best practices |
| [commands.md](podman/commands.md) | Quick-lookup reference for common Podman commands |

---

### [`liberty/`](liberty/)
WebSphere Liberty and Open Liberty application servers.

| File | Description |
|---|---|
| [README.md](liberty/README.md) | What Liberty is, WAS vs Liberty, `server.xml`, running in containers, best practices, migration guide |
| [containerization.md](liberty/containerization.md) | Production-grade containerization with Docker/Podman, `configure.sh`, JVM tuning, and Kubernetes/OpenShift deployment |
| [commands.md](liberty/commands.md) | Quick-lookup reference for Liberty server commands and useful links |

---

### [`java/`](java/)
Java application servers, servlet containers, and middleware.

| Subfolder | Description |
|---|---|
| [`java/jboss/`](java/jboss/) | JBoss EAP & WildFly installation, multi-monolith standalone setup, IDE configuration, and performance tuning |
| [`java/tomcat/`](java/tomcat/) | Apache Tomcat installation guide and notes |
