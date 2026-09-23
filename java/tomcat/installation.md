# Apache Tomcat — Linux Installation Guide

Step-by-step instructions for downloading, installing, configuring permissions, and setting up Apache Tomcat as a systemd service on Linux (Ubuntu, Debian, RHEL, CentOS, Rocky Linux).

---

## Table of Contents

- [1. Prerequisites (JDK Installation)](#1-prerequisites-jdk-installation)
- [2. Create Dedicated Tomcat System User](#2-create-dedicated-tomcat-system-user)
- [3. Download & Extract Tomcat](#3-download--extract-tomcat)
- [4. Set File Permissions & Directories](#4-set-file-permissions--directories)
- [5. Configure systemd Service](#5-configure-systemd-service)
- [6. Configure JVM Startup Options (`setenv.sh`)](#6-configure-jvm-startup-options-setenvsh)
- [7. Verify Installation](#7-verify-installation)
- [8. Uninstall / Clean Removal](#8-uninstall--clean-removal)

---

## 1. Prerequisites (JDK Installation)

Tomcat requires a Java Development Kit (JDK) or Java Runtime Environment (JRE):
- **Tomcat 9.x:** Java 8+
- **Tomcat 10.0.x:** Java 11+
- **Tomcat 10.1.x / 11.x:** Java 17+ (LTS recommended)

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install -y openjdk-17-jdk curl wget

# RHEL / CentOS / Rocky Linux / Fedora
sudo dnf install -y java-17-openjdk java-17-openjdk-devel curl wget

# Verify Java installation and location
java -version
readlink -f $(which java)
```

---

## 2. Create Dedicated Tomcat System User

For security, never run Tomcat as the `root` user in any environment.

```bash
# Creates a system group and user 'tomcat' with home directory /opt/tomcat and login shell disabled
sudo useradd -r -m -U -d /opt/tomcat -s /bin/false tomcat
```

---

## 3. Download & Extract Tomcat

Download the latest stable release from the official Apache repository:

```bash
cd /tmp

# Choose the appropriate version (Tomcat 10.1 for Jakarta EE 10, or Tomcat 9 for Java EE 8)
export TOMCAT_VERSION=10.1.20
wget https://archive.apache.org/dist/tomcat/tomcat-10/v${TOMCAT_VERSION}/bin/apache-tomcat-${TOMCAT_VERSION}.tar.gz

# Create destination directory and extract
sudo mkdir -p /opt/tomcat
sudo tar -xf apache-tomcat-${TOMCAT_VERSION}.tar.gz -C /opt/tomcat --strip-components=1
```

---

## 4. Set File Permissions & Directories

Ensure the `tomcat` user owns the entire installation and execution permissions are set on scripts:

```bash
# Assign ownership
sudo chown -R tomcat:tomcat /opt/tomcat

# Grant execute permission on scripts in bin/
sudo chmod -R u+x /opt/tomcat/bin

# Secure configuration directory
sudo chmod -R g+r /opt/tomcat/conf
sudo chmod g+x /opt/tomcat/conf
```

---

## 5. Configure systemd Service

Create the systemd service file at `/etc/systemd/system/tomcat.service`:

```ini
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

# Identify Java home on your system (e.g., /usr/lib/jvm/java-17-openjdk)
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk"
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat"

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

Reload systemd daemon, enable automatic startup on boot, and start Tomcat:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now tomcat
```

---

## 6. Configure JVM Startup Options (`setenv.sh`)

Create `/opt/tomcat/bin/setenv.sh` to configure memory heap and GC options cleanly without editing `catalina.sh`:

```bash
sudo bash -c 'cat << "EOF" > /opt/tomcat/bin/setenv.sh
#!/usr/bin/env bash

# JVM Heap Sizing
JAVA_OPTS="-Xms1024m -Xmx2048m"

# Metaspace Sizing
JAVA_OPTS="$JAVA_OPTS -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m"

# Modern G1 Garbage Collector
JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC -XX:MaxGCPauseMillis=200"

# Crash Diagnostic Dumps
JAVA_OPTS="$JAVA_OPTS -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/opt/tomcat/logs/oom-dump.hprof"

# Headless mode & Network
JAVA_OPTS="$JAVA_OPTS -Djava.awt.headless=true -Djava.net.preferIPv4Stack=true"

export JAVA_OPTS
EOF'

# Ensure permissions
sudo chown tomcat:tomcat /opt/tomcat/bin/setenv.sh
sudo chmod +x /opt/tomcat/bin/setenv.sh
```

Restart Tomcat to apply the settings:
```bash
sudo systemctl restart tomcat
```

---

## 7. Verify Installation

```bash
# Check service status
sudo systemctl status tomcat

# Check listening ports (default: 8080)
sudo ss -tulpn | grep 8080

# Test HTTP endpoint
curl -I http://localhost:8080/

# Tail the startup logs
tail -n 50 /opt/tomcat/logs/catalina.out
```

---

## 8. Uninstall / Clean Removal

If you need to uninstall Tomcat:

```bash
# Stop and disable service
sudo systemctl stop tomcat
sudo systemctl disable tomcat

# Remove service file
sudo rm /etc/systemd/system/tomcat.service
sudo systemctl daemon-reload

# Remove Tomcat installation and user
sudo rm -rf /opt/tomcat
sudo userdel -r tomcat
```
