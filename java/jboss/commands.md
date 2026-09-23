# JBoss EAP & WildFly — CLI & Command Reference

A quick-lookup reference for starting, stopping, configuring, and troubleshooting JBoss EAP and WildFly application servers via the terminal and JBoss CLI (`jboss-cli.sh`).

---

## Table of Contents

- [Server Management](#server-management)
- [Running Custom Instances (Multiple Monoliths)](#running-custom-instances-multiple-monoliths)
- [JBoss CLI Basics](#jboss-cli-basics)
- [Datasource & Database Configuration](#datasource--database-configuration)
- [Application Deployment](#application-deployment)
- [Log Inspection & Debugging](#log-inspection--debugging)
- [System Ports & Network Verification](#system-ports--network-verification)

---

## Server Management

```bash
# Start server with default configuration (foreground)
$JBOSS_HOME/bin/standalone.sh

# Start server bound to all network interfaces (accessible externally)
$JBOSS_HOME/bin/standalone.sh -b 0.0.0.0 -bmanagement 0.0.0.0

# Start server in debug mode (default debug port: 8787)
$JBOSS_HOME/bin/standalone.sh --debug

# Start using a specific XML configuration file
$JBOSS_HOME/bin/standalone.sh -c standalone-full.xml

# Start in background using nohup
nohup $JBOSS_HOME/bin/standalone.sh -b 0.0.0.0 > /dev/null 2>&1 &

# Graceful shutdown via CLI
$JBOSS_HOME/bin/jboss-cli.sh --connect --command=":shutdown"

# Reload server without JVM restart
$JBOSS_HOME/bin/jboss-cli.sh --connect --command=":reload"
```

---

## Running Custom Instances (Multiple Monoliths)

```bash
# Start instance App 1 (Offset 0 -> Port 8080, Mgmt 9990)
$JBOSS_HOME/bin/standalone.sh \
    -Djboss.server.base.dir=$JBOSS_HOME/standalone_app1 \
    -Djboss.socket.binding.port-offset=0 \
    -b 0.0.0.0 -bmanagement 0.0.0.0

# Start instance App 2 (Offset 100 -> Port 8180, Mgmt 10090)
$JBOSS_HOME/bin/standalone.sh \
    -Djboss.server.base.dir=$JBOSS_HOME/standalone_app2 \
    -Djboss.socket.binding.port-offset=100 \
    -b 0.0.0.0 -bmanagement 0.0.0.0

# Shutdown specific instance
$JBOSS_HOME/bin/jboss-cli.sh --connect --controller=localhost:10090 --command=":shutdown"
```

---

## JBoss CLI Basics

```bash
# Start interactive CLI session
$JBOSS_HOME/bin/jboss-cli.sh --connect

# Connect to a remote host or custom management port
$JBOSS_HOME/bin/jboss-cli.sh --connect --controller=192.168.1.50:9990

# Execute a single command non-interactively
$JBOSS_HOME/bin/jboss-cli.sh --connect --command="/subsystem=datasources:read-resource(recursive=true)"

# Run commands from a script file
$JBOSS_HOME/bin/jboss-cli.sh --connect --file=configure-server.cli

# Read server runtime information
$JBOSS_HOME/bin/jboss-cli.sh --connect --command=":read-attribute(name=server-state)"
```

---

## Datasource & Database Configuration

### 1. Add JDBC Driver as a Module
```bash
$JBOSS_HOME/bin/jboss-cli.sh --connect --command="module add --name=com.oracle.ojdbc8 --resources=/tmp/ojdbc8.jar --dependencies=javax.api,javax.transaction.api"
```

### 2. Register JDBC Driver Subsystem
```bash
$JBOSS_HOME/bin/jboss-cli.sh --connect --command="/subsystem=datasources/jdbc-driver=oracle:add(driver-name=oracle,driver-module-name=com.oracle.ojdbc8,driver-class-name=oracle.jdbc.driver.OracleDriver)"
```

### 3. Create Datasource
```bash
$JBOSS_HOME/bin/jboss-cli.sh --connect --command="data-source add \
    --name=OracleDS \
    --jndi-name=java:/jdbc/OracleDS \
    --driver-name=oracle \
    --connection-url=jdbc:oracle:thin:@localhost:1521:XE \
    --user-name=dbuser \
    --password=dbpassword \
    --min-pool-size=20 \
    --max-pool-size=100 \
    --pool-prefill=true \
    --validate-on-match=false \
    --check-valid-connection-sql=\"SELECT 1 FROM DUAL\" \
    --enabled=true"
```

### 4. Test Datasource Connection
```bash
$JBOSS_HOME/bin/jboss-cli.sh --connect --command="/subsystem=datasources/data-source=OracleDS:test-connection-in-pool"
```

---

## Application Deployment

```bash
# Deploy an EAR archive via CLI to default instance (:9990)
$JBOSS_HOME/bin/jboss-cli.sh --connect --command="deploy /opt/builds/monolith-app1.ear"

# Deploy an EAR to a specific instance (e.g. App 2 on port 10090)
$JBOSS_HOME/bin/jboss-cli.sh --connect --controller=localhost:10090 --command="deploy /opt/builds/monolith-app2.ear"

# Deploy with custom runtime name
$JBOSS_HOME/bin/jboss-cli.sh --connect --command="deploy /opt/builds/monolith-app-v2.ear --runtime-name=monolith.ear"

# List deployed applications on an instance
$JBOSS_HOME/bin/jboss-cli.sh --connect --command="deployment-info"

# Undeploy an EAR archive
$JBOSS_HOME/bin/jboss-cli.sh --connect --command="undeploy monolith.ear"

# Force redeployment
$JBOSS_HOME/bin/jboss-cli.sh --connect --command="deploy /opt/builds/monolith-app1.ear --force"
```

---

## Log Inspection & Debugging

```bash
# Tail the primary server log
tail -f $JBOSS_HOME/standalone/log/server.log

# Tail instance-specific logs
tail -f $JBOSS_HOME/standalone_app1/log/server.log
tail -f $JBOSS_HOME/standalone_app2/log/server.log

# Change root logger level dynamically via CLI without restart
$JBOSS_HOME/bin/jboss-cli.sh --connect --command="/subsystem=logging/root-logger=ROOT:write-attribute(name=level,value=DEBUG)"

# Revert root logger to INFO
$JBOSS_HOME/bin/jboss-cli.sh --connect --command="/subsystem=logging/root-logger=ROOT:write-attribute(name=level,value=INFO)"
```

---

## System Ports & Network Verification

```bash
# Check if ports are listening (Linux)
sudo ss -tulpn | grep -E '8080|8180|9990|10090|8787'

# Curl health check / web endpoint
curl -I http://localhost:8080/
curl -I http://localhost:8180/
```
