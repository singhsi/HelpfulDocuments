# Java Ecosystem, Runtimes & Application Servers

Guides and references for core Java Standard Edition (Java SE), enterprise Java runtimes, servlet containers, and application servers.

---

## Contents

### [`java/java-se/`](java-se/)
Java Standard Edition (Java SE) fundamentals, CLI tools, and runtime execution from first principles without IDEs or build tools.

| File | Description |
|---|---|
| [README.md](java-se/README.md) | Pure Java SE guide: execution model (JDK/JRE/JVM), CLI tools (`java`, `javac`, `javaw`, `jar`, `javap`, `jdb`, `jcmd`), classpath deep dive, and setting classpaths in enterprise software |
| [performance.md](java-se/performance.md) | Java performance tuning, memory architecture (Heap, Metaspace, Stacks), GC algorithms (G1, ZGC, Parallel), production JVM presets, and bottleneck diagnosis |
| [commands.md](java-se/commands.md) | Quick-lookup CLI reference for compilation, execution, JAR creation, bytecode disassembly, and diagnostics |

---

### [`java/jboss/`](jboss/)
JBoss Enterprise Application Platform (EAP) and WildFly Application Server.

| File | Description |
|---|---|
| [README.md](jboss/README.md) | Complete guide covering Linux installation, IDE integration (Eclipse, IntelliJ, VS Code), running multiple monolith instances with custom standalone directories, and high-performance tuning for large codebases |
| [commands.md](jboss/commands.md) | Quick-lookup CLI reference for JBoss CLI (`jboss-cli.sh`), instance management, datasources, and deployments |

---

### [`java/liberty/`](liberty/)
WebSphere Liberty and Open Liberty application servers.

| File | Description |
|---|---|
| [README.md](liberty/README.md) | What Liberty is, WAS vs Liberty, `server.xml`, running in containers, best practices, migration guide |
| [containerization.md](liberty/containerization.md) | Production-grade containerization with Docker/Podman, `configure.sh`, JVM tuning, and Kubernetes/OpenShift deployment |
| [commands.md](liberty/commands.md) | Quick-lookup reference for Liberty server commands and useful links |

---

### [`java/tomcat/`](tomcat/)
Apache Tomcat servlet container.

| File | Description |
|---|---|
| [README.md](tomcat/README.md) | Comprehensive guide: Servlet Container vs Full App Server, Dev/Test/Prod configuration differences, capabilities vs limits, and production hardening |
| [installation.md](tomcat/installation.md) | Step-by-step Linux installation, user permissions, `setenv.sh`, and systemd service configuration |
| [commands.md](tomcat/commands.md) | Quick-lookup CLI reference for systemd service management, lifecycle scripts, remote JPDA debugging, and deployments |
| [Apache-Tomcat-Installation.docx](tomcat/Apache-Tomcat-Installation.docx) | Step-by-step installation instructions for Apache Tomcat and Amazon Corretto JDK on Windows environments |
