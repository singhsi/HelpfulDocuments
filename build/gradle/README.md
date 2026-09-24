# Gradle Multi-Project Architecture — Best Practices Guide

A production-ready guide for building scalable multi-project enterprise systems using modern Gradle (Kotlin DSL `build.gradle.kts`), centralized version catalogs (`libs.versions.toml`), and reusable convention plugins (`build-logic` / `buildSrc`).

> **See also:** [commands.md](commands.md) for quick-lookup Gradle CLI tasks, flags, build scans, and debugging recipes.

---

## Table of Contents

- [1. Multi-Project Architecture Overview](#1-multi-project-architecture-overview)
- [2. Modern Gradle Pillars (Gradle 7.4+ / 8.x+)](#2-modern-gradle-pillars-gradle-74--8x)
  - [Version Catalogs (`gradle/libs.versions.toml`)](#version-catalogs-gradlelibsversionstoml)
  - [Convention Plugins (`build-logic`)](#convention-plugins-build-logic)
  - [Avoiding `subprojects {}` / `allprojects {}`](#avoiding-subprojects---allprojects-)
- [3. Production Code Example](#3-production-code-example)
  - [Directory Layout](#directory-layout)
  - [Version Catalog (`gradle/libs.versions.toml`)](#version-catalog-gradlelibsversionstoml-1)
  - [Root Settings (`settings.gradle.kts`)](#root-settings-settingsgradlekts)
  - [Root Build Script (`build.gradle.kts`)](#root-build-script-buildgradlekts)
  - [Shared Common Core (`common-core/build.gradle.kts`)](#shared-common-core-common-corebuildgradlekts)
  - [Consuming Application (`service-orders/build.gradle.kts`)](#consuming-application-service-ordersbuildgradlekts)
- [4. Dependency Management: `api` vs `implementation`](#4-dependency-management-api-vs-implementation)
- [5. Performance & Build Caching Best Practices](#5-performance--build-caching-best-practices)
  - [`gradle.properties` Performance Settings](#gradleproperties-performance-settings)
  - [Configuration Cache](#configuration-cache)
  - [Build Scans](#build-scans)
- [6. Common Anti-Patterns to Avoid](#6-common-anti-patterns-to-avoid)

---

## 1. Multi-Project Architecture Overview

Modern Gradle enterprise projects separate code into modular libraries and services linked via `settings.gradle.kts`:

```
enterprise-platform/
├── gradle/
│   ├── wrapper/
│   │   ├── gradle-wrapper.jar
│   │   └── gradle-wrapper.properties
│   └── libs.versions.toml             <-- Centralized dependencies & plugins
├── build-logic/                       <-- Convention plugins (compiler flags, jacoco)
├── settings.gradle.kts                <-- Defines all included sub-projects
├── build.gradle.kts                   <-- Root build script (clean task only)
├── gradle.properties                  <-- Memory, Daemon & Cache settings
├── common-core/                       <-- Shared models, utilities, DTOs
│   └── build.gradle.kts
├── common-security/                   <-- Shared JWT, RBAC filters
│   └── build.gradle.kts
└── service-orders/                    <-- Spring Boot REST API
    └── build.gradle.kts
```

---

## 2. Modern Gradle Pillars (Gradle 7.4+ / 8.x+)

### Version Catalogs (`gradle/libs.versions.toml`)
Introduced to eliminate hardcoded string versions (`"org.springframework.boot:spring-boot-starter-web:3.2.4"`) scattered across dozens of `build.gradle` files. It acts as a type-safe, centralized dependency store.

### Convention Plugins (`build-logic` or `buildSrc`)
Instead of copying compiler options, Jacoco setup, and test runner configurations into every child `build.gradle.kts`, you define internal reusable convention plugins (e.g., `enterprise.java-conventions`, `enterprise.spring-service`).

### Avoiding `subprojects {}` and `allprojects {}`
> **Important Modern Best Practice:** The legacy `subprojects { ... }` block in the root `build.gradle` creates cross-project configuration injection. This breaks **Project Isolation** and prevents Gradle from executing **Parallel Configuration** and instant **Configuration Cache**. Use **Convention Plugins** instead.

---

## 3. Production Code Example

### Directory Layout

```
enterprise-platform/
├── gradle/libs.versions.toml
├── settings.gradle.kts
├── build.gradle.kts
├── gradle.properties
├── common-core/
│   └── build.gradle.kts
└── service-orders/
    └── build.gradle.kts
```

---

### Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
java = "17"
spring-boot = "3.2.4"
spring-dependency-management = "1.1.4"
jackson = "2.16.1"
lombok = "1.18.30"
slf4j = "2.0.12"
junit = "5.10.2"

[libraries]
# Spring Boot
spring-boot-starter-web = { module = "org.springframework.boot:spring-boot-starter-web", version.ref = "spring-boot" }
spring-boot-starter-validation = { module = "org.springframework.boot:spring-boot-starter-validation", version.ref = "spring-boot" }
spring-boot-starter-test = { module = "org.springframework.boot:spring-boot-starter-test", version.ref = "spring-boot" }

# Utilities
jackson-databind = { module = "com.fasterxml.jackson.core:jackson-databind", version.ref = "jackson" }
lombok = { module = "org.projectlombok:lombok", version.ref = "lombok" }
slf4j-api = { module = "org.slf4j:slf4j-api", version.ref = "slf4j" }

# Testing
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit" }

[bundles]
# Group related dependencies to import in one line
common-utils = ["jackson-databind", "slf4j-api"]

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
spring-dependency-management = { id = "io.spring.dependency-management", version.ref = "spring-dependency-management" }
```

---

### Root Settings (`settings.gradle.kts`)

```kotlin
rootProject.name = "enterprise-platform"

// 1. Enable Type-Safe Dependency Resolution & Central Repositories
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        mavenCentral()
    }
}

// 2. Register Sub-Projects
include("common-core")
include("common-security")
include("service-orders")
```

---

### Root Build Script (`build.gradle.kts`)

Keep the root build file clean and focused:

```kotlin
plugins {
    // Declare base plugins if needed, or leave empty
}

tasks.register("cleanRoot") {
    description = "Cleans build directories across the entire repository"
    doLast {
        delete(rootProject.layout.buildDirectory)
    }
}
```

---

### Shared Common Core (`common-core/build.gradle.kts`)

```kotlin
plugins {
    `java-library` // Use java-library for reusable components
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(libs.versions.java.get()))
    }
}

dependencies {
    // 'api' exposes dependencies transitively to consuming modules
    api(libs.bundles.common.utils)

    // Lombok annotation processor
    compileOnly(libs.lombok)
    annotationProcessor(libs.lombok)

    // Unit testing
    testImplementation(libs.junit.jupiter)
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

tasks.test {
    useJUnitPlatform()
}
```

---

### Consuming Application (`service-orders/build.gradle.kts`)

```kotlin
plugins {
    `java`
    alias(libs.plugins.spring.boot)
    alias(libs.plugins.spring.dependency.management)
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(libs.versions.java.get()))
    }
}

dependencies {
    // 1. Link to Internal Common Framework
    implementation(project(":common-core"))
    implementation(project(":common-security"))

    // 2. Third-Party Dependencies (Type-safe catalog references)
    implementation(libs.spring.boot.starter.web)
    implementation(libs.spring.boot.starter.validation)

    // Lombok
    compileOnly(libs.lombok)
    annotationProcessor(libs.lombok)

    // Testing
    testImplementation(libs.spring.boot.starter.test)
}

tasks.test {
    useJUnitPlatform()
}
```

---

## 4. Dependency Management: `api` vs `implementation`

When writing reusable common libraries with `java-library`:

| Configuration | Behavior | Best Used When |
|---|---|---|
| **`implementation`** | Classes are **hidden** from downstream modules at compile time. Prevents accidental leakage into consumers' code. | Default choice for internal library dependencies. |
| **`api`** | Classes are **transitively exposed** to consumers' compile classpath. | Types/interfaces from this library appear in your public method signatures or return types. |

```
Project ':service-orders'
   └── depends on ':common-core'
          └── implementation('com.google.guava:guava')  --> Guava NOT visible in :service-orders
          └── api('org.slf4j:slf4j-api')               --> SLF4J IS visible in :service-orders
```

---

## 5. Performance & Build Caching Best Practices

### `gradle.properties` Performance Settings

Place these options in the root `gradle.properties`:

```properties
# 1. Memory allocation for Gradle Daemon
org.gradle.jvmargs=-Xmx4096m -XX:+UseG1GC -XX:+ParallelRefProcEnabled

# 2. Enable persistent Gradle Daemon
org.gradle.daemon=true

# 3. Enable parallel execution of independent sub-projects
org.gradle.parallel=true

# 4. Enable Local Build Cache
org.gradle.caching=true

# 5. Enable Configuration Cache (Instant configuration execution)
org.gradle.configuration-cache=true

# 6. File system watching (Detects modified files instantly)
org.gradle.vfs.watch=true
```

---

### Configuration Cache
Gradle's **Configuration Cache** skips the configuration phase entirely on subsequent runs if build scripts haven't changed:
```bash
./gradlew build --configuration-cache
```
*Cuts repeated command latency from 3–5 seconds to < 200 milliseconds.*

### Build Scans
Generate deep web-based diagnostics of performance bottlenecks, cache misses, and dependency graphs:
```bash
./gradlew build --scan
```

---

## 6. Common Anti-Patterns to Avoid

| Anti-Pattern | Why It Breaks | Modern Fix |
|---|---|---|
| **`subprojects {}` / `allprojects {}`** | Prevents parallel configuration and configuration caching | Use **Convention Plugins** via `build-logic` or `buildSrc` |
| **Hardcoded strings for libraries** | Inconsistent versions across modules, no IDE auto-completion | Centralize in `gradle/libs.versions.toml` |
| **Using `api` everywhere** | Full project recompilation cascades whenever any internal dependency changes | Use `implementation` by default; use `api` only for public API types |
| **Ignoring Gradle Wrapper** | Mismatched developer and CI Gradle versions causing subtle build failures | Always commit `gradlew`, `gradlew.bat`, and `gradle/wrapper/` |
