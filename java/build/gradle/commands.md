# Gradle — CLI & Command Reference

A quick-lookup reference for running tasks, multi-project navigation, debugging dependencies, and tuning performance with the Gradle Wrapper (`./gradlew`).

---

## Table of Contents

- [Core Multi-Project Tasks](#core-multi-project-tasks)
- [Targeting Specific Sub-Projects](#targeting-specific-sub-projects)
- [Dependency Analysis & Debugging](#dependency-analysis--debugging)
- [Performance & Build Caching Flags](#performance--build-caching-flags)
- [Testing & Verification](#testing--verification)
- [Daemon & Environment Maintenance](#daemon--environment-maintenance)

---

## Core Multi-Project Tasks

```bash
# Assemble (compile and package) all modules
./gradlew assemble

# Full build (compile, test, and package all modules)
./gradlew build

# Clean all sub-project build output directories
./gradlew clean

# List all included sub-projects in the build
./gradlew projects

# List all available tasks across the project
./gradlew tasks --all
```

---

## Targeting Specific Sub-Projects

```bash
# Build only a specific sub-project (e.g. :service-orders)
./gradlew :service-orders:build

# Run tests in a specific sub-project
./gradlew :common-core:test

# Run Spring Boot service directly
./gradlew :service-orders:bootRun

# Skip a specific task in the build chain
./gradlew build -x test
```

---

## Dependency Analysis & Debugging

```bash
# View runtime classpath dependency tree of a sub-project
./gradlew :service-orders:dependencies --configuration runtimeClasspath

# View compile classpath dependency tree of a sub-project
./gradlew :service-orders:dependencies --configuration compileClasspath

# Inspect why a specific dependency/version was selected (dependency insight)
./gradlew :service-orders:dependencyInsight --dependency jackson-databind --configuration runtimeClasspath

# Inspect a specific dependency across all configurations
./gradlew :service-orders:dependencyInsight --dependency spring-boot-starter-web
```

---

## Performance & Build Caching Flags

```bash
# Enable local build cache
./gradlew build --build-cache

# Run build in parallel across independent modules
./gradlew build --parallel

# Enable configuration caching (instant execution phase)
./gradlew build --configuration-cache

# Offline mode (resolve only from local Gradle cache ~/.gradle/caches)
./gradlew build --offline

# Generate interactive web-based Build Scan (diagnose performance & cache misses)
./gradlew build --scan

# Refresh all dynamic / SNAPSHOT dependencies from remote repositories
./gradlew build --refresh-dependencies
```

---

## Testing & Verification

```bash
# Run a specific unit test class
./gradlew test --tests "com.enterprise.platform.core.OrderValidatorTest"

# Run a specific test method
./gradlew test --tests "*OrderValidatorTest.testValidateValidOrder"

# Run all tests in a package
./gradlew test --tests "com.enterprise.platform.core.*"

# Force tests to re-run even if outputs are UP-TO-DATE / cached
./gradlew test --rerun-tasks
```

---

## Daemon & Environment Maintenance

```bash
# Check status of running Gradle daemons on the host
./gradlew --status

# Stop all idle and active Gradle daemons
./gradlew --stop

# Update Gradle Wrapper version to latest release
./gradlew wrapper --gradle-version 8.7
```
