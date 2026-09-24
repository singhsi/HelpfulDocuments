# Build Tools — Best Practices Guide

A comprehensive, developer-friendly guide covering modern build tools (**Maven**, **Gradle**, and legacy/enterprise **Ant**) for architecting, building, and maintaining multi-module enterprise projects with shared common libraries and frameworks.

---

## Contents

### [`build/maven/`](maven/)
Apache Maven — multi-module BOM patterns, parent POM inheritance, dependency management, and reproducible CI/CD builds.

| File | Description |
|---|---|
| [README.md](maven/README.md) | Multi-module Maven architecture, parent vs BOM vs aggregator, plugin management, and enterprise best practices |
| [commands.md](maven/commands.md) | Quick command reference for building specific modules, dependency debugging, and release workflows |

---

### [`build/gradle/`](gradle/)
Gradle — multi-project builds, `settings.gradle.kts`, convention plugins (`buildSrc` / composite builds), version catalogs (`libs.versions.toml`), and cache tuning.

| File | Description |
|---|---|
| [README.md](gradle/README.md) | Multi-project Gradle architecture with Kotlin DSL, version catalogs, convention plugins, and performance optimization |
| [commands.md](gradle/commands.md) | Quick command reference for Gradle tasks, daemon management, build scans, and dependency insight |

---

### [`build/ant/`](ant/)
Apache Ant + Ivy — modular legacy enterprise builds, target orchestration, and migrating towards modern build tools.

| File | Description |
|---|---|
| [README.md](ant/README.md) | Modular Ant builds with `ant-contrib` and Apache Ivy dependency management, target dependencies, and modernization patterns |
| [commands.md](ant/commands.md) | Quick command reference for Ant properties, targets, and Ivy resolutions |

---

## Multi-Module Project Pattern Overview

When building applications with a shared common core/framework, the standard multi-module directory structure is:

```
my-enterprise-platform/
├── common-core/              # Domain models, utilities, exceptions, custom annotations
├── common-security/          # JWT, OAuth2, LDAP, RBAC filters & interceptors
├── common-persistence/       # Database entities, JPA repos, connection pooling
├── service-api-orders/       # Web service / REST API for Orders
├── service-api-billing/      # Web service / REST API for Billing
└── app-monolith-ear/         # Enterprise EAR combining multiple modules (if applicable)
```

Each build tool handles this structure using specific idioms:
- **Maven**: `<modules>`, `<dependencyManagement>`, and BOM (Bill of Materials) imports.
- **Gradle**: `include("...")` in `settings.gradle.kts`, version catalogs (`libs.versions.toml`), and convention plugins.
- **Ant + Ivy**: Sub-builds (`<ant dir="..." target="..."/>`), `ivy.xml` module resolvers, and shared common property files.
