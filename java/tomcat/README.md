# Apache Tomcat — Developer Guide

A comprehensive, beginner-to-senior guide covering what Apache Tomcat is, whether it qualifies as an application server, environment-specific deployment strategies (Dev vs Test vs Prod), and a clear capability comparison between Tomcat and full Java EE / Jakarta EE application servers (like WildFly/JBoss EAP or WebSphere Liberty).

> **Related Guides:**
> - [installation.md](installation.md) — Step-by-step Linux installation and systemd service setup
> - [commands.md](commands.md) — Quick-lookup CLI commands, lifecycle scripts, and configuration recipes
> - [Apache-Tomcat-Installation.docx](Apache-Tomcat-Installation.docx) — Windows & Corretto JDK setup guide

---

## Table of Contents

- [Is Tomcat an Application Server?](#is-tomcat-an-application-server)
  - [The Short Answer](#the-short-answer)
  - [Servlet Container vs Full Application Server](#servlet-container-vs-full-application-server)
  - [Visual Architecture](#visual-architecture)
- [Tomcat vs Full App Servers: What You Can & Cannot Do](#tomcat-vs-full-app-servers-what-you-can--cannot-do)
  - [What CAN be done on Tomcat](#what-can-be-done-on-tomcat)
  - [What CANNOT be done on Tomcat out of the box](#what-cannot-be-done-on-tomcat-out-of-the-box)
  - [Feature Comparison Matrix](#feature-comparison-matrix)
  - [When to choose Tomcat vs JBoss / WildFly / Liberty](#when-to-choose-tomcat-vs-jboss--wildfly--liberty)
- [Installation Reference](#installation-reference)
- [Environment Setup: Dev vs Test vs Production](#environment-setup-dev-vs-test-vs-production)
  - [Development Environment](#1-development-environment)
  - [Test / Staging / QA Environment](#2-test--staging--qa-environment)
  - [Production Environment](#3-production-environment)
  - [Environment Differences at a Glance](#environment-differences-at-a-glance)
- [Hardening & Production Best Practices](#hardening--production-best-practices)
  - [JVM Tuning (`setenv.sh`)](#jvm-tuning-setenvsh)
  - [Disable / Remove Default Apps](#disable--remove-default-apps)
  - [Thread Pool & Connector Optimization (`server.xml`)](#thread-pool--connector-optimization-serverxml)
  - [Securing Manager App & Ports](#securing-manager-app--ports)
- [Common Troubleshooting](#common-troubleshooting)

---

## Is Tomcat an Application Server?

### The Short Answer
Technically, **no**. Apache Tomcat is a **Servlet Container** (also called a **Web Container**), not a full Java EE / Jakarta EE Application Server.

However, in everyday developer conversation, people often casually refer to Tomcat as an "app server" because it runs Java web applications (like Spring Boot, Spring MVC, REST APIs, and Servlets).

### Servlet Container vs Full Application Server

```
┌────────────────────────────────────────────────────────────────────────┐
│               Full Jakarta EE / Java EE Application Server             │
│            (e.g., JBoss EAP, WildFly, WebSphere Liberty, WebLogic)     │
│                                                                        │
│ ┌───────────────────────────┐  ┌─────────────────────────────────────┐ │
│ │  Apache Tomcat / Web Tier │  │    Enterprise Java Services Tier    │ │
│ │  ───────────────────────  │  │    ─────────────────────────────    │ │
│ │  • Servlet Specification  │  │  • Enterprise JavaBeans (EJB)       │ │
│ │  • Jakarta Server Pages   │  │  • Java Message Service (JMS Broker)│ │
│ │    (JSP)                  │  │  • JTA (Distributed 2-Phase XA Tx)  │ │
│ │  • Expression Language    │  │  • Full CDI & Contexts              │ │
│ │    (EL)                   │  │  • JCA (Enterprise Connectors)      │ │
│ │  • WebSockets             │  │  • Remote EJB / IIOP / CORBA        │ │
│ │  • Basic JNDI Datasources │  │  • Batch Processing & Concurrency   │ │
│ │  • WAR Deployment Only    │  │  • EAR Multi-Module Packaging       │ │
│ └───────────────────────────┘  └─────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

- **Apache Tomcat (Servlet Container)** implements only the web-tier specifications of Java EE / Jakarta EE:
  - **Jakarta Servlet**
  - **Jakarta Server Pages (JSP)**
  - **Jakarta Expression Language (EL)**
  - **Jakarta WebSocket**
  - Basic JNDI connection pooling (`org.apache.tomcat.dbcp`)
- **Full Application Servers (WildFly, JBoss EAP, WebSphere Liberty, GlassFish, WebLogic)** implement the **entire Jakarta EE Platform Specification** (30+ enterprise APIs) including EJBs, built-in JMS brokers, two-phase distributed transactions (JTA/XA), enterprise connector architecture (JCA), and `.ear` multi-module deployment support.

---

## Tomcat vs Full App Servers: What You Can & Cannot Do

### What CAN be done on Tomcat

1. **Spring & Modern Frameworks:** Run full Spring Boot, Spring MVC, Spring Data, and Quarkus/Micronaut applications (packaged as WARs or running embedded). Modern frameworks bundle their own dependencies, making a full app server unnecessary for 90% of use cases.
2. **RESTful APIs & Web Applications:** Host high-throughput microservices, REST APIs (Jersey, RESTEasy, Spring Web), and traditional JSP/HTML web apps.
3. **Database Connection Pooling:** Configure robust database pools via JNDI in `context.xml` / `server.xml` using Tomcat DBCP or HikariCP.
4. **Clustering & Session Replication:** Group multiple Tomcat nodes with multicast/unicast session replication and sticky sessions via reverse proxies (Nginx, HAProxy, Apache HTTPD).
5. **Security & Authentication:** Integrate with LDAP, Active Directory, OAuth/SAML (via filter extensions or Keycloak adapters), and SSL/TLS HTTPS termination.
6. **WebSockets & HTTP/2:** Full support for real-time bidirectional WebSocket communication and HTTP/2 protocol.

---

### What CANNOT be done on Tomcat out of the box

1. **No `.ear` Deployment Support:**
   - Tomcat **only** understands `.war` (Web Archive) and exploded directories.
   - It **cannot** deploy Enterprise Archives (`.ear`) containing mixed EJB JARs and WARs.
2. **No Built-in EJB Container:**
   - You cannot use `@Stateless`, `@Stateful`, `@Singleton` session beans, `@MessageDriven` beans, or `@Asynchronous` EJB methods natively without adding heavyweight third-party libraries (like Apache OpenEJB / TomEE).
3. **No Native JMS Message Broker:**
   - Tomcat has no built-in message broker (unlike ActiveMQ Artemis inside WildFly). To use JMS, you must connect to an external broker (e.g. RabbitMQ, ActiveMQ, Kafka).
4. **No Two-Phase Commit Distributed Transactions (JTA/XA):**
   - Tomcat supports single-resource database transactions. It cannot natively coordinate atomic transactions across multiple different databases or a database + message queue in a single transaction without integrating an external transaction manager (like Atomikos or Bitronix).
5. **No JCA (Java EE Connector Architecture):**
   - Cannot natively connect to mainframe legacy systems, CICS, IMS, or SAP resource adapters via JCA.
6. **No Remote EJB / IIOP / Corba:**
   - Cannot expose or invoke remote EJB interfaces over RMI/IIOP.

---

### Feature Comparison Matrix

| Feature | Apache Tomcat | WildFly / JBoss EAP | WebSphere Liberty |
|---|---|---|---|
| **Category** | Servlet / Web Container | Full Jakarta EE Application Server | Modular Cloud / Jakarta EE Server |
| **Package Format** | `.war` only | `.ear`, `.war`, `.jar` | `.ear`, `.war`, `.jar` |
| **Memory Footprint (Idle)** | ~50 MB – 150 MB | ~500 MB – 1.5 GB | ~80 MB – 250 MB |
| **Startup Time** | 1 – 3 seconds | 10 – 30+ seconds | 3 – 8 seconds |
| **Servlet / JSP / WebSockets**| ✅ Yes | ✅ Yes | ✅ Yes |
| **Spring Boot / Spring MVC** | ✅ Excellent | ✅ Yes | ✅ Yes |
| **EJB (`@Stateless`, `@MessageDriven`)**| ❌ No | ✅ Native | ✅ Native (with EJB feature) |
| **Built-in JMS Broker** | ❌ No (Requires external) | ✅ ActiveMQ Artemis bundled | ✅ Bundled / External |
| **Distributed XA Transactions** | ❌ Requires 3rd-party (Atomikos) | ✅ Native Narayana engine | ✅ Native Liberty Transaction Mgr |
| **JCA Adapters (ERP/Mainframe)** | ❌ No | ✅ Native | ✅ Native |
| **Admin Web Console** | ⚠️ Basic Manager App | ✅ Rich Enterprise Console (`:9990`) | ✅ Admin Center UI |

---

### When to choose Tomcat vs JBoss / WildFly / Liberty

- **Choose Tomcat if:**
  - You are building modern Spring Boot / Spring MVC web apps or REST APIs.
  - Your application is packaged as a `.war` file.
  - You want lightweight, fast-booting containers (Docker/Podman) with minimal resource usage.
  - You manage messaging via external brokers (Kafka, RabbitMQ) and persistence via Spring/Hibernate.
- **Choose a Full App Server (WildFly / JBoss / Liberty) if:**
  - You have large legacy **`.ear` monoliths** with EJBs, SOAP services, and multiple sub-modules.
  - You require two-phase commit transactions (XA) spanning multiple databases.
  - You need enterprise JCA connectors to mainframe or ERP backends.
  - You rely on full Jakarta EE standard APIs managed by the container.

---

## Installation Reference

For complete, step-by-step setup guides, refer to:
- **Linux Installation & systemd Service:** See [installation.md](installation.md) for JDK prerequisites, non-root user setup, permissions, `setenv.sh` tuning, systemd unit files, and verification commands.
- **Windows Installation:** See [Apache-Tomcat-Installation.docx](Apache-Tomcat-Installation.docx) for Amazon Corretto JDK and installer wizard steps.

---

## Environment Setup: Dev vs Test vs Production

Deploying Tomcat differs dramatically depending on the environment. Here is how to configure each environment properly:

```
┌────────────────────────────────────────────────────────────────────────┐
│               Environment Configuration Strategy                       │
├─────────────────┬──────────────────────────┬───────────────────────────┤
│   DEVELOPMENT   │       TEST / QA / UAT    │        PRODUCTION         │
│  ─────────────  │       ───────────────    │        ──────────         │
│ • IDE integrated│ • Standalone Linux node  │ • Hardened cluster/VM     │
│ • Auto-reload ON│ • Auto-reload OFF        │ • Root access blocked     │
│ • Full debug log│ • INFO logging level     │ • Reverse proxy (Nginx)   │
│ • Manager UI ON │ • Automated CI/CD deploy │ • Default apps removed    │
│ • Small heap    │ • Production-like heap   │ • Tuned G1GC + JMX metrics│
└─────────────────┴──────────────────────────┴───────────────────────────┘
```

---

### 1. Development Environment

**Goal:** Fast feedback loop, hot class reloading, easy debugging, and local IDE control.

- **Installation Method:** Embedded via IDE (Eclipse / IntelliJ IDEA / VS Code) or extracted locally into a developer home folder (`~/dev/apache-tomcat`).
- **JVM & Memory:** Small heap (`-Xms256m -Xmx1024m`) to save developer laptop RAM.
- **Auto-Reload:** Enabled (`<Context reloadable="true">`) so code changes update without full restarts.
- **Debugging:** Remote debug enabled on port `8000` or `8787`:
  ```bash
  catalina.sh jpda start
  # or in setenv.sh:
  JPDA_ADDRESS="*:8000"
  ```
- **Management Apps:** `manager-gui` and `host-manager` enabled in `tomcat-users.xml` for quick web deployments.
- **Logging:** Verbose logging (`DEBUG` or `FINE`) in `conf/logging.properties` for tracking SQL queries and request traces.

---

### 2. Test / Staging / QA Environment

**Goal:** Mirror production runtime configuration, test automated deployments, and validate under realistic data loads.

- **Installation Method:** Managed Linux server/VM or container image built via CI/CD pipeline.
- **JVM & Memory:** Scaled close to production (e.g. `-Xms2048m -Xmx4096m`) to catch memory leaks, heap saturation, or GC stalls before prod release.
- **Auto-Reload:** **Disabled** (`reloadable="false"`, `autoDeploy="false"`). Deployments should be deterministic via CI/CD pipelines (Jenkins, GitHub Actions, GitLab CI).
- **Datasources:** JNDI pools configured to match production connection pool limits to test connection starvation.
- **Management Apps:** Manager app IP-restricted only to the CI/CD deployment runner subnet.
- **Logging:** `INFO` level; rotating file logs enabled to test log aggregators (ELK, Splunk).

---

### 3. Production Environment

**Goal:** High availability, rock-solid security, maximum throughput, zero unauthenticated endpoints.

- **Installation Method:** Dedicated, non-root system user (`tomcat`) running behind a reverse proxy / load balancer (Nginx, HAProxy, AWS ALB, Cloudflare).
- **JVM & Memory:**
  - Fixed heap (`-Xms4096m -Xmx4096m` or higher) to avoid JVM resizing pauses.
  - Tuned **G1GC** (`-XX:+UseG1GC -XX:MaxGCPauseMillis=200`).
  - Heap dump on out-of-memory: `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/tomcat/dumps/`.
- **Security Hardening:**
  - **Remove all default apps:** Delete `/opt/tomcat/webapps/ROOT`, `/docs`, `/examples`, `/manager`, `/host-manager`.
  - **Mask Server Info:** Hide Tomcat version banners in error pages to prevent attacker fingerprinting.
  - **Disable Shutdown Port:** Set `<Server port="-1" shutdown="SHUTDOWN">` in `server.xml` to prevent local shutdown exploits.
- **Network & Connectors:**
  - HTTP/HTTPS terminated at Nginx/Load Balancer; Tomcat receives forwarded requests over high-performance NIO/NIO2 connector.
  - `access_log` enabled with client IP, response time (`%D`), and status code.

---

### Environment Differences at a Glance

| Setting / Property | Development | Test / Staging | Production |
|---|---|---|---|
| **Run As User** | Local developer user | `tomcat` system user | `tomcat` (locked down, `/bin/false`) |
| **JVM Heap Sizing** | `-Xms256m -Xmx1024m` | `-Xms2g -Xmx4g` | `-Xms8g -Xmx8g` (Equal min/max) |
| **GC Policy** | Default | G1GC | Tuned G1GC + OOM Heap Dump |
| **Auto-Deploy / Reload**| `true` | `false` | `false` |
| **Remote Debugging** | Enabled (port 8000) | On-demand only | ❌ Strictly Disabled |
| **Default Webapps** | Retained for testing | Minimal | ❌ Completely Deleted |
| **Manager Web UI** | Enabled (`admin`/`pass`) | Restricted to CI runner | ❌ Disabled / Deleted |
| **Reverse Proxy** | None (Direct `8080`) | Optional / Recommended | ✅ Mandatory (Nginx / ALB) |
| **Shutdown Port** | `8005` | `8005` (localhost only) | `-1` (Disabled via TCP) |

---

## Hardening & Production Best Practices

### JVM Tuning (`setenv.sh`)

Create `/opt/tomcat/bin/setenv.sh` (Tomcat automatically executes this file on startup if present):

```bash
#!/usr/bin/env bash

# Fixed Heap Allocation (Prevents dynamic allocation pauses)
JAVA_OPTS="-Xms4096m -Xmx4096m"

# Metaspace Tuning
JAVA_OPTS="$JAVA_OPTS -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m"

# Modern G1 Garbage Collector
JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:InitiatingHeapOccupancyPercent=45"

# Diagnostics & Crash Dumps
JAVA_OPTS="$JAVA_OPTS -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/opt/tomcat/logs/oom-dump.hprof"

# Headless mode & Network
JAVA_OPTS="$JAVA_OPTS -Djava.awt.headless=true -Djava.net.preferIPv4Stack=true"

# Hide Tomcat Server Version in Response Headers
JAVA_OPTS="$JAVA_OPTS -Dorg.apache.catalina.STRICT_SERVLET_COMPLIANCE=true"

export JAVA_OPTS
```

Make it executable:
```bash
chmod +x /opt/tomcat/bin/setenv.sh
```

---

### Disable / Remove Default Apps

In production, wipe all unneeded sample applications:

```bash
cd /opt/tomcat/webapps
rm -rf docs examples ROOT host-manager manager
```

---

### Thread Pool & Connector Optimization (`server.xml`)

In `/opt/tomcat/conf/server.xml`, optimize the HTTP Connector for high concurrent traffic:

```xml
<Connector port="8080" 
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="300" 
           minSpareThreads="50" 
           maxConnections="10000"
           acceptCount="200"
           connectionTimeout="20000"
           enableLookups="false"
           compression="on"
           compressionMinSize="1024"
           compressableMimeType="text/html,text/xml,text/plain,text/css,application/json,application/javascript"
           server="Web Server"
           redirectPort="8443" />
```

- `maxThreads="300"`: Maximum simultaneous worker request threads.
- `minSpareThreads="50"`: Always keeps 50 threads idle and warm for incoming traffic bursts.
- `acceptCount="200"`: Maximum queue length for incoming connection requests when all threads are busy.
- `server="Web Server"`: Overwrites the HTTP response header `Server: Apache-Coyote/1.1` to prevent version leaking.

---

### Securing Manager App & Ports

If you need the Manager App in QA/Staging, restrict access strictly to trusted IPs in `/opt/tomcat/webapps/manager/META-INF/context.xml`:

```xml
<Context antiResourceLocking="false" privileged="true" >
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|192\.168\.1\.\d+" />
</Context>
```

Disable the TCP shutdown port in `/opt/tomcat/conf/server.xml`:
```xml
<!-- Set port to -1 to disable shutdown listening socket -->
<Server port="-1" shutdown="SHUTDOWN">
```

---

## Common Troubleshooting

1. **Port 8080 Already in Use:**
   - Diagnose: `sudo ss -tulpn | grep 8080` or `sudo lsof -i :8080`
   - Fix: Kill conflicting process or change `<Connector port="8080" ...>` in `server.xml`.
2. **OutOfMemoryError: Java heap space / Metaspace:**
   - Fix: Increase `-Xmx` and `-XX:MaxMetaspaceSize` in `bin/setenv.sh`.
3. **Class Scanning StackOverflow / Slow Startup (e.g. BouncyCastle):**
   - Symptom: `Unable to complete the scan for annotations for web application due to a StackOverflowError`.
   - Fix: In `conf/catalina.properties`, add the jar pattern (e.g., `bcprov*.jar`) to `tomcat.util.scan.StandardJarScanFilter.jarsToSkip=`.
4. **Tomcat Fails to Stop (`shutdown.sh` hangs):**
   - Cause: Non-daemon background threads spawned by application code.
   - Fix: Use `catalina.sh stop -force` or define `CATALINA_PID` in `setenv.sh`.
