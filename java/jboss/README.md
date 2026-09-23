# JBoss EAP & WildFly — Developer Guide

A developer-friendly guide for installing, configuring, and working with JBoss Enterprise Application Platform (EAP) and WildFly Application Server on Linux, setting up popular IDEs (Eclipse, IntelliJ IDEA, VS Code), running multiple monolith instances cleanly, and tuning performance for large enterprise codebases.

> **See also:** [commands.md](commands.md) for quick command-line reference and CLI recipes.

---

## Table of Contents

- [Overview: JBoss EAP vs WildFly](#overview-jboss-eap-vs-wildfly)
- [Linux Installation Guide](#linux-installation-guide)
  - [Prerequisites (Java JDK)](#prerequisites-java-jdk)
  - [Installing WildFly / JBoss EAP](#installing-wildfly--jboss-eap)
  - [Environment Variables Setup](#environment-variables-setup)
  - [Configuring Admin & Application Users](#configuring-admin--application-users)
  - [Running as a systemd Service (Production)](#running-as-a-systemd-service-production)
- [IDE Setup & Integration](#ide-setup--integration)
  - [Eclipse IDE](#eclipse-ide)
  - [IntelliJ IDEA (Ultimate & Community)](#intellij-idea-ultimate--community)
  - [Visual Studio Code (VS Code)](#visual-studio-code-vs-code)
- [Architectural Decision: Running Multiple Monoliths on One Server](#architectural-decision-running-multiple-monoliths-on-one-server)
  - [Single Runtime with Multiple Deployments (Option A)](#option-a-single-standalone-instance-deploy-multiple-wars)
  - [Multiple Custom Standalone Directories (Option B — Recommended)](#option-b-multiple-standalone-base-directories-recommended)
  - [Step-by-step Setup for Multiple Standalone Directories](#step-by-step-setup-for-multiple-standalone-directories)
  - [Domain Mode (Option C)](#option-c-managed-domain-mode)
  - [Comparison Matrix](#comparison-matrix)
- [Performance Tuning for Large Monoliths](#performance-tuning-for-large-monoliths)
  - [JVM & Garbage Collection Tuning](#jvm--garbage-collection-tuning)
  - [Database Connection Pool Tuning](#database-connection-pool-tuning)
  - [Undertow Web Server & Worker Threads](#undertow-web-server--worker-threads)
  - [Deployment Optimization & Scanning](#deployment-optimization--scanning)
  - [Classloading & Module Isolation (`jboss-deployment-structure.xml`)](#classloading--module-isolation)
- [Troubleshooting & Best Practices](#troubleshooting--best-practices)

---

## Overview: JBoss EAP vs WildFly

WildFly is the upstream open-source community project sponsored by Red Hat. JBoss Enterprise Application Platform (EAP) is the hardened, commercially supported enterprise build derived from WildFly.

| Feature / Aspect | WildFly | JBoss EAP |
|---|---|---|
| **License** | LGPL v2.1 (Free, Open Source) | Subscription-based (Enterprise Support) |
| **Release Cadence** | Fast (~quarterly feature releases) | Stable, multi-year maintenance |
| **Java EE / Jakarta EE** | Jakarta EE 8 / 9.1 / 10 (depending on version) | Jakarta EE 8 / 10 |
| **Default Web Engine** | Undertow | Undertow |
| **Management** | JBoss CLI (`jboss-cli.sh`), Web Console (`:9990`) | JBoss CLI (`jboss-cli.sh`), Web Console (`:9990`) |
| **Configuration Model** | XML-based (`standalone.xml` / `domain.xml`) + CLI | XML-based (`standalone.xml` / `domain.xml`) + CLI |

---

## Linux Installation Guide

### Prerequisites (Java JDK)

JBoss EAP 7.4 / WildFly 26+ generally requires **Java 11 or Java 17 (LTS)**.

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install -y openjdk-17-jdk

# RHEL / CentOS / Rocky Linux / Fedora
sudo dnf install -y java-17-openjdk java-17-openjdk-devel

# Verify Java installation
java -version
```

### Installing WildFly / JBoss EAP

1. **Create a dedicated system user (recommended for security):**
   ```bash
   sudo useradd -r -m -d /opt/wildfly -s /bin/bash wildfly
   ```

2. **Download and extract:**
   ```bash
   cd /opt
   # For WildFly (replace version with latest stable, e.g. 31.0.1.Final):
   sudo wget https://github.com/wildfly/wildfly/releases/download/31.0.1.Final/wildfly-31.0.1.Final.tar.gz
   sudo tar -xvf wildfly-31.0.1.Final.tar.gz
   sudo ln -s /opt/wildfly-31.0.1.Final /opt/wildfly
   sudo chown -R wildfly:wildfly /opt/wildfly-31.0.1.Final /opt/wildfly
   ```

   *(For JBoss EAP: Download the `jboss-eap-7.x.x.zip` from the Red Hat Customer Portal and extract into `/opt/jboss-eap`)*

### Environment Variables Setup

Add the following to `/etc/profile.d/wildfly.sh` or `~/.bashrc`:

```bash
export JBOSS_HOME=/opt/wildfly
export PATH=$PATH:$JBOSS_HOME/bin
```

Reload environment:
```bash
source /etc/profile.d/wildfly.sh
```

### Configuring Admin & Application Users

By default, management console access is disabled until an admin user is created:

```bash
# Create an Administrator for the Management Console (:9990)
$JBOSS_HOME/bin/add-user.sh

# Select:
# a) Management User
# Username: admin
# Password: StrongPassword123!
# Groups: (leave blank or press enter)
# Add for remote connections?: yes
```

### Running as a systemd Service (Production)

WildFly comes with pre-configured systemd templates in `$JBOSS_HOME/docs/contrib/scripts/systemd/`:

```bash
# 1. Create configuration directory
sudo mkdir -p /etc/wildfly
sudo cp /opt/wildfly/docs/contrib/scripts/systemd/wildfly.conf /etc/wildfly/

# 2. Copy launch script
sudo cp /opt/wildfly/docs/contrib/scripts/systemd/launch.sh /opt/wildfly/bin/
sudo chmod +x /opt/wildfly/bin/launch.sh

# 3. Copy systemd service file
sudo cp /opt/wildfly/docs/contrib/scripts/systemd/wildfly.service /etc/systemd/system/

# 4. Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable --now wildfly
sudo systemctl status wildfly
```

Access the Web Console at `http://<server-ip>:9990` and the application port at `http://<server-ip>:8080`.

---

## IDE Setup & Integration

### Eclipse IDE

1. **Install JBoss Tools / Red Hat Server Connector:**
   - Go to **Help** -> **Eclipse Marketplace...**
   - Search for `JBoss Tools` or `Red Hat Server Adapter`.
   - Install **JBoss AS, WildFly & EAP Server Tools**.
2. **Add Server Runtime:**
   - Open the **Servers** view (**Window** -> **Show View** -> **Servers**).
   - Right-click in the window -> **New** -> **Server**.
   - Expand **Red Hat JBoss Middleware** or **WildFly** and select your WildFly / EAP version.
   - Set the **Home Directory** to `/opt/wildfly` (or your local installation path).
   - Specify the JDK (Java 11 or 17).
3. **Deploying Applications:**
   - Right click the server in the Servers view -> **Add and Remove...** -> select your project -> **Finish**.
   - Click the green **Debug** or **Run** button to launch.

---

### IntelliJ IDEA (Ultimate & Community)

#### IntelliJ IDEA Ultimate (Native Support)
1. Open **Run/Debug Configurations** -> click `+` -> Select **JBoss / WildFly Server** -> **Local**.
2. Click **Configure...** next to "Application Server" and point to your `$JBOSS_HOME`.
3. In the **Deployment** tab, click `+` -> **Artifact** -> choose your `ear:exploded` or `ear` artifact (or `.war`).
4. Set "On 'Update' action" to **Update classes and resources** for fast hot reloading.
5. In the **Server** tab, configure JVM options and port settings if necessary.

#### IntelliJ IDEA Community Edition
Community Edition does not include Java EE application server integration plugins. Use one of these options:
- **Option 1: Remote Debugging:**
  Start WildFly in debug mode:
  ```bash
  $JBOSS_HOME/bin/standalone.sh --debug
  ```
  In IntelliJ, create a **Remote JVM Debug** configuration (Port: `8787`).
- **Option 2: Maven / Gradle Plugin:**
  Use `wildfly-maven-plugin` directly from the Maven tool window:
  ```bash
  mvn wildfly:deploy
  ```

---

### Visual Studio Code (VS Code)

1. Install the official Red Hat extensions:
   - **Extension Pack for Java** (`vscjava.vscode-java-pack`)
   - **Server Connector** (`redhat.vscode-server-connector`)
2. In VS Code Explorer, open the **Servers** pane in the sidebar.
3. Click `+` (Create New Server) -> Select **Red Hat JBoss Enterprise Application Platform (EAP)** or **WildFly**.
4. Point to the local directory where WildFly is extracted.
5. Right-click the server -> **Start Server** or **Debug Server**.
6. Right-click the server -> **Add Deployment** -> pick your project `.ear` file (or exploded folder / `.war`).

---

## Architectural Decision: Running Multiple Monoliths on One Server (Multiple EARs)

Enterprise monolithic applications are typically packaged as **Enterprise Archive (`.ear`)** files containing multiple Web Modules (`.war`), EJB Modules (`.jar`), shared utility libraries (`/lib`), and configuration descriptors (`META-INF/application.xml`, `META-INF/jboss-app.xml`).

When running multiple large monolith EARs on a single machine or VM (e.g., `app1.ear` and `app2.ear`), you have three primary architectural choices:

### Option A: Single Standalone Instance, Deploy Multiple EARs
*Deploy `app1.ear` and `app2.ear` into the same `/opt/wildfly/standalone/deployments` directory.*
- **Pros:** Single JVM process to start and monitor.
- **Cons — Critical issues for large monoliths:**
  - **Single Point of Failure:** If `app1.ear` throws an `OutOfMemoryError` or triggers high GC pauses, `app2.ear` freezes or crashes.
  - **Shared Metaspace & Thread Pools:** Loading hundreds of EJBs, servlets, and JAX-RS/JAX-WS endpoints from multiple EARs exhausts Metaspace (`java.lang.OutOfMemoryError: Metaspace`) and saturates worker threads.
  - **Lifecycle Coupling:** Restarting or patching `app1.ear` requires restarting the entire server process, disrupting `app2.ear`.
  - **Global Configuration Clashes:** Security domains, mail sessions, or JNDI bindings defined at the server level may clash between the two monoliths.

---

### Option B: Multiple Standalone Base Directories (RECOMMENDED)
*Use a single shared WildFly binary installation, but create isolated standalone directories such as `/opt/wildfly/standalone_app1` and `/opt/wildfly/standalone_app2`.*

Each monolith runs in its **own separate JVM process** with its own:
- Dedicated ports (using port offsets)
- Independent memory & garbage collection settings
- Dedicated log files (`server.log`)
- Dedicated deployment directory and configuration file

#### Why this is the best pattern for multiple large EAR monoliths:
1. **Total Process & Memory Isolation:** A crash, memory leak, or thread starvation in `app1.ear` never impacts `app2.ear`.
2. **Zero Binary Duplication:** Shared `$JBOSS_HOME/bin`, `$JBOSS_HOME/modules`, and server core files save disk space and simplify applying server patches.
3. **Independent Lifecycles:** Deploy, restart, rebuild, or debug `app1.ear` without disturbing developers or services on `app2.ear`.
4. **Dedicated JVM & Resource Profiles:** Allocate 12GB heap + 2GB Metaspace to a massive legacy EAR (`standalone_app1`), and 4GB heap to a smaller EAR (`standalone_app2`).
5. **Clean EAR Deployments:** Place `app1.ear` into `standalone_app1/deployments/` and `app2.ear` into `standalone_app2/deployments/`.

---

### Step-by-step Setup for Multiple Standalone Directories

Here is how to set up `standalone_app1` (for `app1.ear`) and `standalone_app2` (for `app2.ear`):

#### 1. Copy the standalone template directory
```bash
cd $JBOSS_HOME

# Create dedicated directory for App 1 EAR
cp -r standalone standalone_app1

# Create dedicated directory for App 2 EAR
cp -r standalone standalone_app2

# Clean default history and old deployments
rm -rf standalone_app1/data standalone_app1/log standalone_app1/tmp standalone_app1/deployments/*
rm -rf standalone_app2/data standalone_app2/log standalone_app2/tmp standalone_app2/deployments/*

# Place EAR files into their respective deployment folders
cp /path/to/app1.ear standalone_app1/deployments/
cp /path/to/app2.ear standalone_app2/deployments/
```

#### 2. Create custom startup scripts for each app

Create `$JBOSS_HOME/bin/start-app1.sh`:
```bash
#!/usr/bin/env bash
export JBOSS_HOME=/opt/wildfly
export JAVA_OPTS="-Xms4096m -Xmx8192m -XX:+UseG1GC -Djava.net.preferIPv4Stack=true"

exec $JBOSS_HOME/bin/standalone.sh \
    -Djboss.server.base.dir=$JBOSS_HOME/standalone_app1 \
    -Djboss.socket.binding.port-offset=0 \
    -Djboss.bind.address=0.0.0.0 \
    -Djboss.bind.address.management=0.0.0.0
```

Create `$JBOSS_HOME/bin/start-app2.sh` (using `port-offset=100`):
```bash
#!/usr/bin/env bash
export JBOSS_HOME=/opt/wildfly
export JAVA_OPTS="-Xms2048m -Xmx4096m -XX:+UseG1GC -Djava.net.preferIPv4Stack=true"

exec $JBOSS_HOME/bin/standalone.sh \
    -Djboss.server.base.dir=$JBOSS_HOME/standalone_app2 \
    -Djboss.socket.binding.port-offset=100 \
    -Djboss.bind.address=0.0.0.0 \
    -Djboss.bind.address.management=0.0.0.0
```

Make them executable:
```bash
chmod +x $JBOSS_HOME/bin/start-app1.sh $JBOSS_HOME/bin/start-app2.sh
```

#### 3. Understanding Port Offsets
When `port-offset=100` is applied:
| Service | Default Port (App 1, Offset 0) | Offset 100 Port (App 2) |
|---|---|---|
| HTTP Web Traffic | `8080` | `8180` |
| HTTPS Secure | `8443` | `8543` |
| Management Console / CLI | `9990` | `10090` |
| Remote Debugging (if enabled) | `8787` | `8887` |

#### 4. Managing with JBoss CLI for specific instances
Connect to the desired instance via CLI by providing the management port:
```bash
# Connect to App 1 CLI
$JBOSS_HOME/bin/jboss-cli.sh --connect --controller=localhost:9990

# Connect to App 2 CLI
$JBOSS_HOME/bin/jboss-cli.sh --connect --controller=localhost:10090
```

---

### Option C: Managed Domain Mode
*Run a Domain Controller (`domain.sh`) managing multiple Host Controllers and Server Groups on one machine.*
- **Pros:** Centralized XML management for hundreds of server instances.
- **Cons:** Significantly steeper learning curve, complex clustering/topology definitions, rarely needed for single-host development or moderate monolith instances.

### Comparison Matrix

| Criteria | Single Standalone (Option A) | Multiple Standalone Folders (Option B) | Domain Mode (Option C) |
|---|---|---|---|
| **JVM Process Isolation** | ❌ None (Shared) | ✅ Complete | ✅ Complete |
| **Independent Restarts** | ❌ No | ✅ Yes | ✅ Yes |
| **Complexity** | 🟢 Very Low | 🟢 Low | 🔴 High |
| **Memory Tuning Flexibility**| ❌ Global only | ✅ Per-Monolith | ✅ Per Server Group |
| **Recommended For** | Small microservices / small WARs | **Large Monoliths (EARs)** | Multi-node enterprise clusters |

---

## Performance Tuning for Large Monoliths (EARs)

Large monolithic EAR codebases (often containing tens of JARs, EJBs, and multiple WAR sub-modules) suffer from long startup times, high heap and Metaspace consumption, database connection starvation, and classloading overhead. Apply the following configurations in your `standalone.conf` or startup script:

### JVM & Garbage Collection Tuning

For monoliths with 8GB–32GB heap requirements, use the **G1 Garbage Collector** (default in modern JDKs) with tuned pause time targets:

Add to `bin/standalone.conf` (or instance startup script):
```bash
# Minimum and Maximum Heap (equal size prevents dynamic resizing pauses)
JAVA_OPTS="-Xms8192m -Xmx8192m"

# Metaspace for large number of loaded classes
JAVA_OPTS="$JAVA_OPTS -XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=1024m"

# G1 Garbage Collector Settings
JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC"
JAVA_OPTS="$JAVA_OPTS -XX:MaxGCPauseMillis=200"
JAVA_OPTS="$JAVA_OPTS -XX:InitiatingHeapOccupancyPercent=45"
JAVA_OPTS="$JAVA_OPTS -XX:G1ReservePercent=15"

# Thread Stack Size (prevent StackOverflow in deep frameworks like Spring/Hibernate)
JAVA_OPTS="$JAVA_OPTS -Xss1024k"

# Networking & Headless
JAVA_OPTS="$JAVA_OPTS -Djava.net.preferIPv4Stack=true -Djava.awt.headless=true"
```

---

### Database Connection Pool Tuning

Default connection pools (typically min=10, max=20) are too small for heavy monoliths.

Execute in `jboss-cli.sh`:
```bash
/subsystem=datasources/data-source=MyDataSource:write-attribute(name=min-pool-size,value=20)
/subsystem=datasources/data-source=MyDataSource:write-attribute(name=max-pool-size,value=150)
/subsystem=datasources/data-source=MyDataSource:write-attribute(name=pool-prefill,value=true)
/subsystem=datasources/data-source=MyDataSource:write-attribute(name=idle-timeout-minutes,value=15)
/subsystem=datasources/data-source=MyDataSource:write-attribute(name=blocking-timeout-wait-millis,value=10000)
/subsystem=datasources/data-source=MyDataSource:write-attribute(name=validate-on-match,value=false)
/subsystem=datasources/data-source=MyDataSource:write-attribute(name=check-valid-connection-sql,value="SELECT 1")
```

---

### Undertow Web Server & Worker Threads

Configure the Undertow IO subsystem and default worker to handle concurrent requests:

In `standalone.xml` (under `<subsystem xmlns="urn:jboss:domain:io:...">`):
```xml
<worker name="default"
        io-threads="16"
        task-core-threads="64"
        task-max-threads="512"
        task-keepalive="60000" />
```

Via CLI:
```bash
/subsystem=io/worker=default:write-attribute(name=task-max-threads,value=512)
/subsystem=io/worker=default:write-attribute(name=io-threads,value=16)
```
*(Rule of thumb: `io-threads` = CPU core count; `task-max-threads` = 16 to 32 × CPU core count).*

---

### Deployment Optimization & Scanning

#### Disable Dynamic Deployment Scanner in Production
The deployment scanner constantly polls the disk for file modifications. For high performance, disable it or use CLI deployments:
```bash
/subsystem=deployment-scanner/scanner=default:write-attribute(name=scan-interval,value=0)
```

#### Increase Deployment Timeout
Large monoliths can take several minutes to deploy on first boot. Prevent timeout failures:
```bash
/subsystem=deployment-scanner/scanner=default:write-attribute(name=deployment-timeout,value=1200)
```

---

### Classloading & Module Isolation

Large monolithic EARs bundle multiple sub-deployments (WARs and EJB JARs) along with third-party libraries in the EAR's `/lib` folder. In JBoss/WildFly, each sub-deployment has its own classloader. Use `META-INF/jboss-deployment-structure.xml` at the root of the EAR to control class visibility across sub-modules and exclude server runtime conflicts:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jboss-deployment-structure xmlns="urn:jboss:deployment-structure:1.3">
    <!-- EAR-level root deployment configuration -->
    <deployment>
        <!-- Exclude server-bundled modules that conflict with your EAR libraries -->
        <exclusions>
            <module name="org.slf4j" />
            <module name="org.slf4j.impl" />
            <module name="org.apache.commons.logging" />
            <module name="org.apache.log4j" />
        </exclusions>
        
        <!-- Explicitly import specific JBoss modules the EAR requires -->
        <dependencies>
            <module name="org.infinispan" export="true" />
            <module name="javax.annotation.api" />
        </dependencies>
    </deployment>

    <!-- Sub-deployment configuration for a specific WAR inside the EAR -->
    <sub-deployment name="webapp.war">
        <exclusions>
            <module name="org.slf4j" />
        </exclusions>
        <dependencies>
            <!-- Allow WAR to access classes from an EJB sub-module -->
            <module name="deployment.app1.ear.core-ejb.jar" />
        </dependencies>
    </sub-deployment>
</jboss-deployment-structure>
```

---

## Troubleshooting & Best Practices

1. **Port Conflicts on Startup:**
   - Error: `Address already in use: bind`
   - Fix: Check active ports with `sudo ss -tulpn | grep -E '8080|9990|8180|10090'` or ensure each instance uses a distinct `port-offset`.
2. **OutOfMemoryError / Metaspace:**
   - Increase `-XX:MaxMetaspaceSize=1024m` and `-Xmx`.
3. **Slow Server Startup:**
   - Exclude unnecessary scan annotations in your beans.xml / web.xml (`metadata-complete="true"`).
   - Tune database pool initialization with `pool-prefill=true`.
4. **Log Rotation:**
   - Ensure `periodic-rotating-file-handler` is active in `standalone.xml` logging subsystem to prevent disk exhaustion.
