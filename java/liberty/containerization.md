# WebSphere Liberty & Open Liberty — Production-Grade Containerization Guide

A comprehensive, production-ready guide to building, tuning, securing, and deploying WebSphere Liberty and Open Liberty container images with Docker / Podman and Kubernetes / Red Hat OpenShift.

> **See also:**
> - [README.md](README.md) — Liberty core concepts, `server.xml`, and development guide
> - [commands.md](commands.md) — Quick command-line reference

---

## Table of Contents

- [1. Overview & Architecture](#1-overview--architecture)
- [2. Choosing the Right Base Image](#2-choosing-the-right-base-image)
- [3. Multi-Stage Dockerfile (Production-Grade)](#3-multi-stage-dockerfile-production-grade)
- [4. The Liberty `configure.sh` Build Tool](#4-the-liberty-configuresh-build-tool)
- [5. Production `server.xml` Configuration](#5-production-serverxml-configuration)
- [6. Container Security & Hardening](#6-container-security--hardening)
- [7. Memory, JVM & Performance Tuning](#7-memory-jvm--performance-tuning)
- [8. Health Checks, Metrics & Observability](#8-health-checks-metrics--observability)
- [9. Production Deployment on Kubernetes / OpenShift](#9-production-deployment-on-kubernetes--openshift)
- [10. Container Verification & CI/CD Checklist](#10-container-verification--cicd-checklist)

---

## 1. Overview & Architecture

Liberty was engineered from the ground up for containerized workloads. Unlike traditional application servers, Liberty containers achieve:
- **Instant startup (1-3s)** with IBM Semeru / OpenJ9 Shared Class Cache (SCC)
- **Sub-100MB idle memory footprint**
- **Dynamic configuration injection** via MicroProfile Config, environment variables, and Kubernetes ConfigMaps/Secrets
- **Zero-root security** complying with OpenShift `restricted` SCC and CIS Docker benchmarks

```
┌────────────────────────────────────────────────────────────────────────┐
│                   Liberty Container Layer Architecture                 │
├────────────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────────────────────────────────────┐ │
│ │ Layer 4: Application Layer (/config/apps/myapp.war or .ear)         │ │
│ ├────────────────────────────────────────────────────────────────────┤ │
│ │ Layer 3: Server Configuration (/config/server.xml, configDropins/)  │ │
│ ├────────────────────────────────────────────────────────────────────┤ │
│ │ Layer 2: Liberty Runtime + Features (pre-warmed via configure.sh)  │ │
│ ├────────────────────────────────────────────────────────────────────┤ │
│ │ Layer 1: Base OS (Red Hat UBI) + IBM Semeru Runtimes (Java 17/21)  │ │
│ └────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Choosing the Right Base Image

Liberty provides official images on **IBM Container Registry (`icr.io`)** and **Docker Hub**.

### Official Image Registries:
- **WebSphere Liberty (Enterprise with IBM entitlement):** `icr.io/appcafe/websphere-liberty`
- **Open Liberty (Open Source):** `icr.io/appcafe/open-liberty`

### Recommended Production Image Tags:

| Image Tag | JVM / Base OS | Best For |
|---|---|---|
| `kernel-slim-java17-openj9-ubi` | IBM Semeru Java 17 + Red Hat UBI | **Production Standard** (Smallest size, fastest boot via `configure.sh`) |
| `kernel-slim-java21-openj9-ubi` | IBM Semeru Java 21 + Red Hat UBI | **Modern Java 21 LTS** |
| `full-java17-openj9-ubi` | IBM Semeru Java 17 + All Features | Development / Quick prototyping |

> **Production Recommendation:** Always use the `kernel-slim-java*-openj9-ubi` tags. When `RUN configure.sh` executes during the image build, it downloads *only* the specific features listed in your `server.xml`, keeping image size to a bare minimum (often < 350MB).

---

## 3. Multi-Stage Dockerfile (Production-Grade)

Below is an enterprise-grade multi-stage `Dockerfile` that builds the application from source with Maven, builds the optimized Liberty container, and applies production hardening.

```dockerfile
# ==========================================
# Stage 1: Build Application Artifact (WAR/EAR)
# ==========================================
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /build

# Cache maven dependencies
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Compile and package application
COPY src ./src
RUN mvn clean package -DskipTests -B

# ==========================================
# Stage 2: Production Liberty Runtime
# ==========================================
FROM icr.io/appcafe/websphere-liberty:kernel-slim-java17-openj9-ubi AS production

LABEL maintainer="Enterprise Platform Team" \
      vendor="IBM WebSphere Liberty" \
      version="1.0.0"

# Set environment variables for production behavior
ENV OPENJ9_RESTORE_JAVA_DUMP=true \
    WLP_LOGGING_MESSAGE_FORMAT=JSON \
    WLP_LOGGING_MESSAGE_SOURCE=message,trace,accessLog,ffdc \
    WLP_LOGGING_CONSOLE_LOGLEVEL=INFO

# Copy server configuration and database drivers
COPY --chown=1001:0 src/main/liberty/config/server.xml /config/server.xml
COPY --chown=1001:0 src/main/liberty/config/jvm.options /config/jvm.options
COPY --chown=1001:0 src/main/liberty/config/bootstrap.properties /config/bootstrap.properties

# Copy application binary from builder stage
COPY --chown=1001:0 --from=builder /build/target/*.war /config/apps/

# Optional: Copy JDBC drivers if not bundled in app
# COPY --chown=1001:0 src/main/liberty/lib/ /opt/ibm/wlp/usr/shared/resources/

# Run Liberty configure script to download required features and prime Shared Class Cache (SCC)
RUN configure.sh

# Expose HTTP, HTTPS, and Metrics/Health endpoints
EXPOSE 9080 9443

# Liberty non-root user (1001)
USER 1001

# Start Liberty server
CMD ["/opt/ibm/wlp/bin/server", "run", "defaultServer"]
```

---

## 4. The Liberty `configure.sh` Build Tool

The `RUN configure.sh` command is the single most critical step in creating a production Liberty container.

### What `configure.sh` does during `docker build`:
1. **Feature Provisioning:** Reads `/config/server.xml` and downloads only the features declared in `<featureManager>`.
2. **Intermediate Class Pre-compilation:** Pre-compiles and warms the **Shared Class Cache (SCC)**.
3. **AOT (Ahead-of-Time) Compilation:** Triggers OpenJ9 AOT compilation for runtime classes.
4. **Prunes Temp Files:** Strips caches and installer metadata to reduce container layer size.

### Result:
- Startup time drops from **8-12 seconds down to 1.2-2.5 seconds**.
- Image size stays minimal.

---

## 5. Production `server.xml` Configuration

Create `src/main/liberty/config/server.xml` with production patterns:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<server description="Production Liberty Application">

    <!-- 1. Minimal Feature Set: Load only what your app uses -->
    <featureManager>
        <feature>servlet-6.0</feature>
        <feature>restfulWS-3.1</feature>
        <feature>jsonb-3.0</feature>
        <feature>cdi-4.0</feature>
        <!-- Observability & Cloud Native features -->
        <feature>mpHealth-4.0</feature>
        <feature>mpMetrics-5.0</feature>
        <feature>mpConfig-3.1</feature>
    </featureManager>

    <!-- 2. HTTP/HTTPS Endpoint Configuration -->
    <httpEndpoint id="defaultHttpEndpoint"
                  host="*"
                  httpPort="9080"
                  httpsPort="9443">
        <!-- High throughput connection tuning -->
        <tcpOptions maxOpenConnections="10000"
                    soReuseAddr="true" />
    </httpEndpoint>

    <!-- 3. Web Application Deployment (Disable dynamic reload in production) -->
    <webApplication id="productionApp"
                    location="my-app.war"
                    contextRoot="/" />

    <!-- Disables file monitoring for file changes to save CPU -->
    <applicationMonitor updateTrigger="disabled" />
    <config updateTrigger="disabled" />

    <!-- 4. Production Datasource with Environment Variable Injection -->
    <dataSource id="primaryDS" jndiName="jdbc/PrimaryDS" type="javax.sql.ConnectionPoolDataSource">
        <jdbcDriver libraryRef="dbDriverLib" />
        <properties.postgresql serverName="${env.DB_HOST:-postgres}"
                               portNumber="${env.DB_PORT:-5432}"
                               databaseName="${env.DB_NAME:-appdb}"
                               user="${env.DB_USER}"
                               password="${env.DB_PASSWORD}" />
        <connectionManager minPoolSize="10"
                           maxPoolSize="100"
                           maxIdleTime="30m"
                           agedTimeout="2h"
                           reapTime="3m" />
    </dataSource>

    <library id="dbDriverLib">
        <fileset dir="${shared.resource.dir}/jdbc" includes="*.jar" />
    </library>

    <!-- 5. Structured JSON Logging for Log Forwarders (Elasticsearch, FluentBit, Splunk) -->
    <logging messageFormat="json"
             messageSource="message,trace,accessLog,ffdc"
             consoleLogLevel="INFO" />

</server>
```

---

## 6. Container Security & Hardening

1. **Non-Root User (`1001:0`):**
   - The base image runs as user `1001` (group `0` / root group for OpenShift compatibility).
   - All copied files must use `--chown=1001:0`.
   - Never run container processes with `USER root`.

2. **Immutable Server Configuration:**
   - In production, set `<applicationMonitor updateTrigger="disabled" />` and `<config updateTrigger="disabled" />`. This prevents unexpected runtime reloads and eliminates CPU polling overhead.

3. **Externalized Secrets (Zero Plaintext Passwords):**
   - Use `${env.VARIABLE_NAME}` syntax in `server.xml`.
   - Never commit database passwords or keystore passwords in Dockerfiles or Git repositories.

4. **HTTPS / TLS Configuration:**
   - In container platforms (Kubernetes / OpenShift), TLS termination is typically handled at the Ingress / Route.
   - If end-to-end TLS is required inside the cluster:
     ```xml
     <keyStore id="defaultKeyStore" 
               location="/etc/liberty-tls/keystore.p12" 
               password="${env.KEYSTORE_PASSWORD}" 
               type="PKCS12" />
     ```

---

## 7. Memory, JVM & Performance Tuning

Configure `src/main/liberty/config/jvm.options`:

```properties
# ==========================================
# Production OpenJ9 / Semeru JVM Options
# ==========================================

# Enable container memory awareness
-XX:+UseContainerSupport

# Set max heap dynamically based on container memory limit
-XX:MaxRAMPercentage=75.0
-XX:InitialRAMPercentage=50.0

# Optimize Garbage Collection for throughput and low memory footprint
-Xgcpolicy:gencon

# Shared Class Cache (SCC) tuning for rapid container restart
-Xshareclasses:cacheDir=/output/.classCache,nonfatal

# Thread stack size
-Xss512k

# Headless environment
-Djava.awt.headless=true

# Fast DNS caching for microservice service discovery
-Dnetworkaddress.cache.ttl=30
-Dnetworkaddress.cache.negative.ttl=5
```

---

## 8. Health Checks, Metrics & Observability

With `mpHealth-4.0` and `mpMetrics-5.0` enabled, Liberty automatically exposes Kubernetes-standard health and monitoring endpoints:

| Endpoint | Purpose | Kubernetes Probe |
|---|---|---|
| `http://<pod-ip>:9080/health/live` | Process Liveness | `livenessProbe` |
| `http://<pod-ip>:9080/health/ready` | Traffic Readiness | `readinessProbe` |
| `http://<pod-ip>:9080/health/started` | Startup Verification | `startupProbe` |
| `http://<pod-ip>:9080/metrics` | Prometheus Metrics | Prometheus Scraping |

---

## 9. Production Deployment on Kubernetes / OpenShift

Here is a complete, production-ready Kubernetes deployment manifest (`deployment.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: liberty-app
  labels:
    app: liberty-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: liberty-app
  template:
    metadata:
      labels:
        app: liberty-app
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 0
        fsGroup: 0
      containers:
        - name: liberty-app
          image: myregistry.example.com/apps/liberty-app:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 9080
            - name: https
              containerPort: 9443
          env:
            - name: DB_HOST
              value: "postgres-service.production.svc.cluster.local"
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: "customerdb"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1024Mi"
          startupProbe:
            httpGet:
              path: /health/started
              port: 9080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 20
          livenessProbe:
            httpGet:
              path: /health/live
              port: 9080
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 9080
            periodSeconds: 5
            failureThreshold: 2
---
apiVersion: v1
kind: Service
metadata:
  name: liberty-app-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 9080
      targetPort: 9080
  selector:
    app: liberty-app
```

---

## 10. Container Verification & CI/CD Checklist

Before deploying the container image to production, verify the build locally:

```bash
# 1. Build the image with Podman or Docker
podman build -t liberty-prod:1.0.0 -f Dockerfile .

# 2. Run the container locally with resource limits
podman run -d --name test-liberty \
  -p 9080:9080 \
  -m 512m --cpus=1.0 \
  -e DB_USER=test -e DB_PASSWORD=secret \
  liberty-prod:1.0.0

# 3. Check startup time (should be < 3 seconds)
podman logs test-liberty | grep "ready to run a smarter planet"

# 4. Verify MicroProfile health endpoints
curl -I http://localhost:9080/health/ready
curl -I http://localhost:9080/health/live

# 5. Verify metrics scrape
curl http://localhost:9080/metrics

# 6. Verify non-root execution
podman exec test-liberty id
# Output should show: uid=1001(default) gid=0(root)

# 7. Clean up
podman rm -f test-liberty
```

### Pre-Deployment Checklist

- [ ] Used `kernel-slim-java*-openj9-ubi` base image.
- [ ] Executed `RUN configure.sh` during the container build.
- [ ] Added `--chown=1001:0` to all `COPY` commands.
- [ ] Disabled runtime file monitoring (`updateTrigger="disabled"`).
- [ ] Configured JSON structured logging (`WLP_LOGGING_MESSAGE_FORMAT=JSON`).
- [ ] Injected all credentials via environment variables (`${env.VAR}`).
- [ ] Configured Kubernetes `startupProbe`, `livenessProbe`, and `readinessProbe` with `/health/*`.
- [ ] Set JVM container memory parameters (`-XX:MaxRAMPercentage=75.0`).
