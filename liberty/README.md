# WebSphere Liberty & Open Liberty — New Developer Guide

A beginner-friendly introduction to Liberty application servers. Covers what Liberty is, how it compares to traditional WebSphere, and best practices for development and containerisation.

> **See also:**
> - [commands.md](commands.md) — Quick command-line reference
> - [containerization.md](containerization.md) — Production-grade Docker/Podman containerization & Kubernetes deployment

---

## Table of Contents

- [What is Liberty?](#what-is-liberty)
- [WebSphere Liberty vs Open Liberty](#websphere-liberty-vs-open-liberty)
- [Liberty vs Traditional WebSphere (WAS)](#liberty-vs-traditional-websphere-was)
- [Key Concepts](#key-concepts)
- [Project Structure](#project-structure)
- [server.xml — The Heart of Liberty](#serverxml--the-heart-of-liberty)
- [Running Liberty in a Container](#running-liberty-in-a-container)
- [Best Practices](#best-practices)
- [Common Mistakes to Avoid](#common-mistakes-to-avoid)
- [Migrating from Traditional WebSphere](#migrating-from-traditional-websphere)

---

## What is Liberty?

Liberty is a lightweight, fast Java application server from IBM. It is designed to start in seconds, use minimal memory, and support modern cloud-native deployment patterns like containers.

It supports standard Java EE / Jakarta EE APIs (Servlets, JPA, JAX-RS, CDI, etc.) as well as MicroProfile — a set of APIs tailored for building microservices.

**Key characteristics:**
- Starts in 2–5 seconds (compared to minutes for traditional WAS)
- Modular — you only enable the features your app actually needs
- Configuration is a single XML file (`server.xml`) — no clicking through admin consoles
- Designed to run well in Docker/Podman containers
- Supports zero-migration between Liberty versions (IBM's compatibility promise)

---

## WebSphere Liberty vs Open Liberty

These two are closely related and share nearly all the same code.

| | Open Liberty | WebSphere Liberty |
|---|---|---|
| License | Open source (Apache 2.0) | IBM commercial product |
| Cost | Free | Requires IBM entitlement |
| Support | Community | IBM production support |
| Features | All Open Liberty features | Open Liberty features + IBM-only features |
| Release cadence | 4 weeks | Quarterly (based on Open Liberty) |
| Use case | Development, community projects | Enterprise production deployments |

> **Think of it this way:** Open Liberty is the upstream open-source project. WebSphere Liberty is the IBM-supported, enterprise-hardened version built on top of it. Most code written for one runs on the other without changes.

For local development, Open Liberty works great. For production deployments at an enterprise, WebSphere Liberty with an IBM entitlement is the typical choice.

---

## Liberty vs Traditional WebSphere (WAS)

Many enterprise teams are migrating from traditional WebSphere Application Server (WAS) to Liberty. Here's why:

| | Traditional WAS | Liberty |
|---|---|---|
| Startup time | Minutes | Seconds |
| Memory footprint | High | Low |
| Configuration | Admin console (GUI) | `server.xml` (text file) |
| Deployment | EAR/WAR via console | Drop file into `dropins/` or config |
| Container-friendly | Difficult | Designed for it |
| Feature model | Monolithic — everything included | Modular — enable only what you need |
| Java EE support | Full | Full (Jakarta EE 10 on latest) |

---

## Key Concepts

**server.xml**
The single configuration file for a Liberty server. All features, ports, datasources, security, and app config live here. No clicking through admin consoles.

**Features**
Liberty is modular. You declare exactly which Java EE/MicroProfile capabilities your app needs. For example, `servlet-6.0` gives you Servlet support; `jpa-3.0` gives you JPA. The server only loads what's listed — this keeps memory usage low and startup fast.

**dropins**
A folder where you can drop a WAR or EAR file to have Liberty automatically deploy it — no restart needed. Useful during development.

**WLP (WebSphere Liberty Profile)**
You'll see `wlp` in directory names and commands. It stands for WebSphere Liberty Profile — the install directory of a Liberty runtime.

**defaultServer**
When you install Liberty, a default server called `defaultServer` is created automatically. It's the one used when no server name is specified.

**MicroProfile**
An open standard (backed by IBM, Red Hat, Microsoft and others) that extends Jakarta EE with APIs specifically for microservices: health checks, metrics, fault tolerance, JWT auth, config injection, and more. Liberty has full MicroProfile support.

---

## Project Structure

A typical Liberty server directory looks like this:

```
wlp/
└── usr/
    └── servers/
        └── my-server/
            ├── server.xml          # main configuration
            ├── bootstrap.properties  # variables used in server.xml
            ├── apps/               # deployed applications (.war, .ear)
            ├── dropins/            # auto-deploy folder — drop apps here
            └── logs/
                ├── messages.log    # main log
                ├── console.log     # JVM output
                └── ffdc/           # crash dumps
```

---

## server.xml — The Heart of Liberty

This is the file you'll spend most of your time in. Here's an annotated example:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<server description="My App Server">

    <!-- features: only load what you need -->
    <featureManager>
        <feature>servlet-6.0</feature>
        <feature>jpa-3.0</feature>
        <feature>jaxrs-3.0</feature>
        <feature>cdi-4.0</feature>
        <feature>mpHealth-4.0</feature>   <!-- MicroProfile health checks -->
    </featureManager>

    <!-- HTTP and HTTPS ports -->
    <httpEndpoint id="defaultHttpEndpoint"
                  host="*"
                  httpPort="9080"
                  httpsPort="9443" />

    <!-- deploy an app from the apps/ folder -->
    <webApplication location="my-app.war" contextRoot="/my-app" />

    <!-- a datasource (database connection) -->
    <dataSource id="myDS" jndiName="jdbc/myDB">
        <jdbcDriver libraryRef="db2Lib" />
        <properties.db2.jcc databaseName="MYDB"
                            serverName="localhost"
                            portNumber="50000"
                            user="${db.user}"
                            password="${db.password}" />
    </dataSource>

    <!-- reference secrets from environment variables, not hardcoded -->
    <variable name="db.user"     defaultValue="admin" />
    <variable name="db.password" defaultValue="changeme" />

</server>
```

> **Tip:** Use `${variable-name}` in `server.xml` to inject values from environment variables or `bootstrap.properties`. Never hardcode passwords.

---

## Running Liberty in a Container

Liberty provides official container images on IBM Container Registry (`icr.io`) and Docker Hub. The recommended base image is `icr.io/appcafe/websphere-liberty` (for WebSphere Liberty) or `icr.io/appcafe/open-liberty` (for Open Liberty).

#### example Dockerfile
```dockerfile
# use the WebSphere Liberty base image with Jakarta EE + MicroProfile
FROM icr.io/appcafe/websphere-liberty:full-java17-openj9-ubi

# copy your server configuration
COPY --chown=1001:0 src/main/liberty/config/server.xml /config/server.xml

# copy your built application
COPY --chown=1001:0 target/my-app.war /config/apps/my-app.war

# Liberty's configure script applies feature installs and other setup
RUN configure.sh
```

Key points:
- The `/config/` directory inside the container is where Liberty looks for `server.xml`
- `--chown=1001:0` is required — Liberty containers run as a non-root user (`1001`)
- `RUN configure.sh` pre-warms feature installation so the container starts faster
- Default ports are `9080` (HTTP) and `9443` (HTTPS)

#### default ports
| Port | Protocol |
|---|---|
| 9080 | HTTP |
| 9443 | HTTPS |

---

## Best Practices

#### enable only the features you need
Every feature you add increases memory usage and startup time. Start with the minimum and add more only when required.
```xml
<!-- avoid -->
<feature>javaee-8.0</feature>   <!-- loads everything -->

<!-- prefer -->
<feature>servlet-6.0</feature>
<feature>jaxrs-3.0</feature>
```

#### never hardcode secrets in server.xml
Use Liberty variables backed by environment variables instead:
```xml
<variable name="db.password" defaultValue="" />
```
Then pass the value at runtime:
```
podman run -e db.password=mysecret my-liberty-app
```

#### use `dropins/` only for development
The `dropins/` folder is convenient locally but bypasses explicit configuration. For production, declare apps explicitly in `server.xml` using `<webApplication>`.

#### use `server run` for local development, `server start` for production
- `server run` — runs in the foreground, logs go to the console. Easy to see what's happening.
- `server start` — runs as a background daemon. Use this in production or CI.

#### check `messages.log` first when something goes wrong
`messages.log` is the first place to look for errors. It contains timestamped entries for server events, app deployments, and exceptions. `ffdc/` contains detailed dumps for unexpected failures.

#### use MicroProfile Health for container readiness
Add `mpHealth-4.0` to your features and implement health check endpoints. Container orchestrators (like Kubernetes) use these to know when the app is ready to receive traffic and when to restart it.
```xml
<feature>mpHealth-4.0</feature>
```
Liberty automatically exposes:
- `GET /health` — overall status
- `GET /health/live` — liveness (is the process running?)
- `GET /health/ready` — readiness (is the app ready for traffic?)

#### use MicroProfile Config to externalise configuration
Avoid hardcoding environment-specific values (URLs, ports, feature flags) in your code. MicroProfile Config lets you inject them from environment variables or config files:
```java
@Inject
@ConfigProperty(name = "my.service.url")
private String serviceUrl;
```

---

## Common Mistakes to Avoid

| Mistake | Why it's a problem | Fix |
|---|---|---|
| Loading `javaee-8.0` or `jakartaee-10.0` feature | Loads every feature, wastes memory | Enable only the features your app uses |
| Hardcoding passwords in `server.xml` | Security risk | Use `${variable}` backed by env vars |
| Using `dropins/` in production | No version control, no explicit config | Use `<webApplication>` in `server.xml` |
| Not setting `--chown=1001:0` in Dockerfile | Container fails to start (permission error) | Always chown files to `1001:0` |
| Ignoring `messages.log` when debugging | Harder to find the root cause | Check `messages.log` first, then `ffdc/` |
| Not running `configure.sh` in Dockerfile | Feature installs happen at startup (slow) | Always include `RUN configure.sh` |

---

## Migrating from Traditional WebSphere

If you're moving an app from traditional WAS to Liberty, IBM provides tooling to help:

1. **WebSphere Migration Toolkit** — scans your existing app for compatibility issues and suggests fixes. Available as an Eclipse/VS Code plugin.
   - https://github.com/IBMTechSales/klp-workshop-labs/tree/master/1171-Liberty-Migration-Tools

2. **IBM Transformation Advisor** — analyses your WAS environment and estimates migration effort.

3. **Common migration tasks:**
   - Replace proprietary WAS APIs with standard Jakarta EE equivalents
   - Move from `ibm-web-ext.xml` / `ibm-ejb-jar-ext.xml` bindings to `server.xml` config
   - Replace JNDI lookups with CDI injection where possible
   - Test with `server run` to catch issues early with live console output

> **Tip:** The zero-migration guarantee means your app won't break when upgrading between Liberty versions — IBM doesn't remove or change behaviour of existing features. You can upgrade safely.
