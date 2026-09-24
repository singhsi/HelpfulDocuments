# Apache Ant — CLI & Command Reference

A quick-lookup reference for running targets, debugging properties, managing Ivy dependencies, and building multi-module Ant projects.

---

## Table of Contents

- [Running Targets](#running-targets)
- [Property Overrides & Diagnostics](#property-overrides--diagnostics)
- [Apache Ivy Dependency Commands](#apache-ivy-dependency-commands)
- [Sub-Module Target Execution](#sub-module-target-execution)
- [Testing & Quality](#testing--quality)

---

## Running Targets

```bash
# Run default target defined in build.xml
ant

# Run a specific target (e.g. clean, compile, test, jar)
ant compile
ant test
ant jar

# Clean and rebuild
ant clean compile jar

# Execute with specific build file
ant -f custom-build.xml package
```

---

## Property Overrides & Diagnostics

```bash
# Override a property defined in build.properties
ant -Djava.target=21 compile

# Enable verbose logging (shows full javac and copy details)
ant -v compile

# Enable debug logging (maximum diagnostic output)
ant -d compile

# Quiet mode (suppress informative messages, show only errors/warnings)
ant -q package

# List all public targets and their descriptions in the project
ant -p
```

---

## Apache Ivy Dependency Commands

```bash
# Resolve and download all dependencies defined in ivy.xml
ant resolve

# Clear local Ivy cache (~/.ivy2/cache)
ant -Divy.cache.dir=/tmp/ivy-cache resolve

# Generate an HTML dependency report using Ivy
ant -Divy.report.output=report.html report
```

---

## Sub-Module Target Execution

```bash
# Execute master build that runs all sub-modules in sequence
ant build-all

# Execute a target directly inside a sub-module directory
cd common-core
ant jar

# Clean all sub-modules from root orchestrator
ant clean-all
```

---

## Testing & Quality

```bash
# Run all unit tests and halt on first failure
ant test

# Run tests and generate HTML report
ant test-report

# Run tests ignoring failures (record all results)
ant -Dhaltonfailure=false test
```
