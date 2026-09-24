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

### [`java/`](java/)
Java application servers, servlet containers, and middleware.

| Subfolder | Description |
|---|---|
| [`java/java-se/`](java/java-se/) | Java SE core execution model, CLI binaries (`java`, `javac`, `javaw`, `jar`, `javap`), and classpath management |
| [`java/jboss/`](java/jboss/) | JBoss EAP & WildFly installation, multi-monolith standalone setup, IDE configuration, and performance tuning |
| [`java/liberty/`](java/liberty/) | WebSphere Liberty & Open Liberty guide, production containerization, and commands |
| [`java/tomcat/`](java/tomcat/) | Apache Tomcat developer guide, Linux installation, Dev/Test/Prod environments, and commands |

---

### [`build/`](build/)
Build tools and best practices for multi-module enterprise projects with shared common libraries/frameworks.

| Subfolder | Description |
|---|---|
| [`build/maven/`](build/maven/) | Maven multi-module architecture, parent POM vs BOM, dependency convergence, and reactor commands |
| [`build/gradle/`](build/gradle/) | Gradle multi-project builds with Kotlin DSL, version catalogs (`libs.versions.toml`), and convention plugins |
| [`build/ant/`](build/ant/) | Modular Ant builds with Apache Ivy dependency management, shared macro targets, and modernization paths |
