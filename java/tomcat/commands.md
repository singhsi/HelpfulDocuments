# Apache Tomcat — CLI & Command Reference

A quick-lookup reference for starting, stopping, configuring, inspecting, and managing Apache Tomcat on Linux.

---

## Table of Contents

- [Service Management (systemd)](#service-management-systemd)
- [Manual Lifecycle Scripts](#manual-lifecycle-scripts)
- [Log Inspection](#log-inspection)
- [Remote Debugging (JPDA)](#remote-debugging-jpda)
- [Managing Deployments](#managing-deployments)
- [Ports & Health Verification](#ports--health-verification)
- [Configuration Files Overview](#configuration-files-overview)

---

## Service Management (systemd)

```bash
# Start Tomcat service
sudo systemctl start tomcat

# Stop Tomcat service
sudo systemctl stop tomcat

# Restart Tomcat
sudo systemctl restart tomcat

# Check service status and latest logs
sudo systemctl status tomcat

# Enable automatic startup on boot
sudo systemctl enable tomcat

# Reload systemd configuration after editing tomcat.service
sudo systemctl daemon-reload
```

---

## Manual Lifecycle Scripts

```bash
# Start Tomcat in background
/opt/tomcat/bin/startup.sh

# Stop Tomcat gracefully
/opt/tomcat/bin/shutdown.sh

# Force shutdown if hanging
/opt/tomcat/bin/shutdown.sh -force

# Start in foreground (interactive terminal / prints stdout directly)
/opt/tomcat/bin/catalina.sh run

# Start with remote JPDA debugging enabled
/opt/tomcat/bin/catalina.sh jpda start

# Check Tomcat version and JVM info
/opt/tomcat/bin/version.sh
```

---

## Log Inspection

```bash
# Follow the main catalina log (server lifecycle, stdout, uncaught errors)
tail -f /opt/tomcat/logs/catalina.out

# Follow access log (HTTP traffic, client IP, latency)
tail -f /opt/tomcat/logs/localhost_access_log.*.txt

# Follow localhost log (application-level servlet exceptions)
tail -f /opt/tomcat/logs/localhost.*.log

# Follow systemd journal logs
sudo journalctl -u tomcat -f -n 100
```

---

## Remote Debugging (JPDA)

To enable remote debugging in development:

```bash
# In bin/setenv.sh:
export JPDA_ADDRESS="*:8000"
export JPDA_TRANSPORT="dt_socket"

# Start in debug mode
/opt/tomcat/bin/catalina.sh jpda start
```

In Eclipse / IntelliJ / VS Code, attach a **Remote JVM Debugger** to port `8000`.

---

## Managing Deployments

```bash
# Deploy a new WAR file (auto-extracted when Tomcat is running)
cp /path/to/myapp.war /opt/tomcat/webapps/

# Deploy to root context (http://localhost:8080/)
cp /path/to/myapp.war /opt/tomcat/webapps/ROOT.war

# Undeploy an application
rm -rf /opt/tomcat/webapps/myapp.war /opt/tomcat/webapps/myapp

# Clean temporary compilation cache and work directory
rm -rf /opt/tomcat/work/Catalina/localhost/* /opt/tomcat/temp/*
```

---

## Ports & Health Verification

```bash
# Check if Tomcat ports (8080, 8005, 8009) are listening
sudo ss -tulpn | grep -E '8080|8005|8009'

# Health check HTTP response code
curl -I http://localhost:8080/

# Test Manager API status (if enabled)
curl -u admin:StrongPassword123! http://localhost:8080/manager/text/list
```

---

## Configuration Files Overview

| File Path | Description |
|---|---|
| `conf/server.xml` | Core server architecture: Port connectors (`8080`), Engine, Host, Valves |
| `conf/context.xml` | Global application context: JNDI Datasource connections, Session managers |
| `conf/web.xml` | Global servlet configurations, default MIME types, session timeout |
| `conf/catalina.properties` | Classloader paths, JAR scanning exclusion filters (`jarsToSkip`) |
| `conf/tomcat-users.xml` | User roles and credentials for Manager and Host-Manager web apps |
| `bin/setenv.sh` | Custom environment variables, JVM heap (`-Xmx`), Metaspace, GC parameters |
