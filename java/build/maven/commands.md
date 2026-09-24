# Maven — CLI & Command Reference

A quick-lookup cheat sheet for building, inspecting, testing, and debugging multi-module Maven projects.

---

## Table of Contents

- [Reactor & Multi-Module Targeting](#reactor--multi-module-targeting)
- [Parallel Builds & Performance](#parallel-builds--performance)
- [Dependency Analysis & Debugging](#dependency-analysis--debugging)
- [Testing & Quality](#testing--quality)
- [Profiles & Property Overrides](#profiles--property-overrides)
- [Release & Publishing](#release--publishing)

---

## Reactor & Multi-Module Targeting

```bash
# Build specific module and its required dependencies (Also Make)
mvn clean install -pl :service-orders -am

# Build specific module by folder path
mvn clean install -pl service-orders -am

# Build module and everything that depends on it (Also Make Dependents)
mvn clean install -pl :common-core -amd

# Resume a failed multi-module build from the point of failure
mvn compile -rf :service-billing

# Build everything EXCEPT a specific module
mvn clean install -pl !:service-billing
```

---

## Parallel Builds & Performance

```bash
# Build using 1 thread per CPU core
mvn clean package -T 1C

# Build using 4 threads
mvn clean package -T 4

# Offline mode (skip remote repository checks, use local ~/.m2/repository cache)
mvn clean install -o

# Skip test execution (compiles tests, skips running them)
mvn clean package -DskipTests

# Skip test compilation and execution completely
mvn clean package -Dmaven.test.skip=true
```

---

## Dependency Analysis & Debugging

```bash
# View complete dependency tree for the entire project
mvn dependency:tree

# View dependency tree for a specific child module
mvn dependency:tree -pl :service-orders

# Search for where a specific transitive dependency is coming from
mvn dependency:tree -Dincludes=com.fasterxml.jackson.core:*

# Analyze unused declared dependencies and used undeclared dependencies
mvn dependency:analyze

# Display effective POM (all inheritance and profiles resolved)
mvn help:effective-pom

# Display effective settings (~/.m2/settings.xml)
mvn help:effective-settings

# Check for newer versions of dependencies and plugins
mvn versions:display-dependency-updates
mvn versions:display-plugin-updates
```

---

## Testing & Quality

```bash
# Run a specific unit test class
mvn test -Dtest=OrderServiceTest

# Run a specific test method
mvn test -Dtest=OrderServiceTest#testCreateOrder

# Run all tests matching a wildcard pattern
mvn test -Dtest=*ServiceTest

# Run integration tests (failsafe plugin)
mvn verify -DskipUnitTests

# Generate JaCoCo code coverage report
mvn jacoco:report
```

---

## Profiles & Property Overrides

```bash
# Activate a specific Maven profile
mvn clean package -Pproduction

# Activate multiple profiles
mvn clean package -Pproduction,docker

# Override a POM property on the CLI
mvn clean install -Djava.version=21 -DskipTests

# List all active profiles for the build
mvn help:active-profiles
```

---

## Release & Publishing

```bash
# Update versions across all child POMs in the reactor simultaneously
mvn versions:set -DnewVersion=1.1.0-SNAPSHOT

# Commit version changes after versions:set
mvn versions:commit

# Revert version changes if needed
mvn versions:revert

# Deploy artifacts to remote enterprise Nexus/Artifactory repository
mvn clean deploy -DskipTests
```
