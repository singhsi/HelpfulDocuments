# Java Application Servers & Middleware

Guides and references for enterprise Java runtimes and application servers.

---

## Contents

### [`java/jboss/`](jboss/)
JBoss Enterprise Application Platform (EAP) and WildFly Application Server.

| File | Description |
|---|---|
| [README.md](jboss/README.md) | Complete guide covering Linux installation, IDE integration (Eclipse, IntelliJ, VS Code), running multiple monolith instances with custom standalone directories, and high-performance tuning for large codebases |
| [commands.md](jboss/commands.md) | Quick-lookup CLI reference for JBoss CLI (`jboss-cli.sh`), instance management, datasources, and deployments |

---

### [`java/tomcat/`](tomcat/)
Apache Tomcat servlet container.

| File | Description |
|---|---|
| [README.md](tomcat/README.md) | Comprehensive guide: Servlet Container vs Full App Server, Dev/Test/Prod configuration differences, capabilities vs limits, and production hardening |
| [installation.md](tomcat/installation.md) | Step-by-step Linux installation, user permissions, `setenv.sh`, and systemd service configuration |
| [commands.md](tomcat/commands.md) | Quick-lookup CLI reference for systemd service management, lifecycle scripts, remote JPDA debugging, and deployments |
| [Apache-Tomcat-Installation.docx](tomcat/Apache-Tomcat-Installation.docx) | Step-by-step installation instructions for Apache Tomcat and Amazon Corretto JDK on Windows environments |
