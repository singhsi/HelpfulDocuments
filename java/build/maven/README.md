# Maven Multi-Module Architecture — Best Practices Guide

A comprehensive, production-grade guide for organizing, managing, and building enterprise multi-module Maven projects with shared common libraries, domain frameworks, and microservices/monoliths.

> **See also:** [commands.md](commands.md) for quick-lookup Maven CLI commands, reactor flags, and debugging recipes.

---

## Table of Contents

- [1. Multi-Module Project Architecture](#1-multi-module-project-architecture)
- [2. Three Core Patterns: Parent vs BOM vs Aggregator](#2-three-core-patterns-parent-vs-bom-vs-aggregator)
  - [The Root Aggregator & Parent POM](#the-root-aggregator--parent-pom)
  - [The Bill of Materials (BOM) Pattern](#the-bill-of-materials-bom-pattern)
- [3. Production Code Example](#3-production-code-example)
  - [Directory Layout](#directory-layout)
  - [Root `pom.xml`](#root-pomxml)
  - [Platform BOM `platform-bom/pom.xml`](#platform-bom-platform-bompomxml)
  - [Common Framework `common-core/pom.xml`](#common-framework-common-corepomxml)
  - [Consuming Service `service-orders/pom.xml`](#consuming-service-service-orderspomxml)
- [4. Dependency Management Rules](#4-dependency-management-rules)
  - [Never specify `<version>` in child modules](#never-specify-version-in-child-modules)
  - [Enforce dependency convergence](#enforce-dependency-convergence)
  - [Manage plugin versions centrally](#manage-plugin-versions-centrally)
- [5. CI/CD & Build Performance Tuning](#5-cicd--build-performance-tuning)
  - [Parallel Multi-Threaded Builds](#parallel-multi-threaded-builds)
  - [Building Subset of Modules (Reactor Flags)](#building-subset-of-modules-reactor-flags)
  - [Build Cache & Daemon](#build-cache--daemon)
- [6. Common Anti-Patterns to Avoid](#6-common-anti-patterns-to-avoid)

---

## 1. Multi-Module Project Architecture

In enterprise applications, multiple applications (REST services, worker jobs, web portals) frequently share a common foundation:
- Domain exceptions and utilities (`common-core`)
- Security filters and token parsing (`common-security`)
- Database base entities and repositories (`common-persistence`)

```
enterprise-platform/                      <-- Root Aggregator & Parent
├── pom.xml                               <-- Defines modules & pluginManagement
├── platform-bom/                         <-- Bill of Materials (manages versions)
│   └── pom.xml
├── common-core/                          <-- Shared utilities, models, exceptions
│   └── pom.xml
├── common-security/                      <-- Shared authentication/authorization
│   └── pom.xml
├── common-persistence/                   <-- Shared JPA entities & datasources
│   └── pom.xml
├── service-orders/                       <-- Consuming application 1 (WAR/JAR)
│   └── pom.xml
└── service-billing/                      <-- Consuming application 2 (WAR/JAR)
    └── pom.xml
```

---

## 2. Three Core Patterns: Parent vs BOM vs Aggregator

Understanding the distinction between these three Maven concepts is vital:

| Concept | Purpose | Defined By |
|---|---|---|
| **Aggregator POM** | Groups modules to build together in a single reactor execution | `<modules><module>...</module></modules>` |
| **Parent POM** | Shares configuration (plugins, properties, compiler settings) down to child modules | `<parent>...</parent>` |
| **BOM (Bill of Materials)** | Manages version alignment across internal and external dependencies without forced inheritance | `<dependencyManagement><type>pom</type><scope>import</scope>` |

> **Best Practice:** Combine the Root Aggregator and Parent POM into the top-level project `pom.xml`, and define a dedicated `platform-bom` module so external microservices or external teams can consume your common libraries without inheriting your parent POM.

---

## 3. Production Code Example

### Directory Layout

```
enterprise-platform/
├── pom.xml
├── platform-bom/pom.xml
├── common-core/pom.xml
├── common-security/pom.xml
├── service-orders/pom.xml
```

---

### Root `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.enterprise.platform</groupId>
    <artifactId>enterprise-platform</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>Enterprise Platform :: Aggregator &amp; Parent</name>
    <description>Root POM managing modules, plugin versions, and build standards</description>

    <!-- 1. Define Sub-modules -->
    <modules>
        <module>platform-bom</module>
        <module>common-core</module>
        <module>common-security</module>
        <module>service-orders</module>
    </modules>

    <!-- 2. Global Properties -->
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>17</java.version>
        <maven.compiler.release>${java.version}</maven.compiler.release>

        <!-- Core Dependency Versions -->
        <spring-boot.version>3.2.4</spring-boot.version>
        <jackson.version>2.16.1</jackson.version>
        <lombok.version>1.18.30</lombok.version>
        <slf4j.version>2.0.12</slf4j.version>

        <!-- Maven Plugin Versions -->
        <maven-compiler-plugin.version>3.13.0</maven-compiler-plugin.version>
        <maven-surefire-plugin.version>3.2.5</maven-surefire-plugin.version>
        <maven-failsafe-plugin.version>3.2.5</maven-failsafe-plugin.version>
        <maven-enforcer-plugin.version>3.4.1</maven-enforcer-plugin.version>
        <jacoco-maven-plugin.version>0.8.11</jacoco-maven-plugin.version>
    </properties>

    <!-- 3. Import Internal BOM & External Starters -->
    <dependencyManagement>
        <dependencies>
            <!-- Import internal platform BOM -->
            <dependency>
                <groupId>com.enterprise.platform</groupId>
                <artifactId>platform-bom</artifactId>
                <version>${project.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- Import external Spring Boot BOM (Bill of Materials) -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!-- 4. Central Plugin Management -->
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>${maven-compiler-plugin.version}</version>
                    <configuration>
                        <release>${java.version}</release>
                        <parameters>true</parameters>
                    </configuration>
                </plugin>

                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <version>${maven-surefire-plugin.version}</version>
                </plugin>

                <!-- Code Coverage -->
                <plugin>
                    <groupId>org.jacoco</groupId>
                    <artifactId>jacoco-maven-plugin</artifactId>
                    <version>${jacoco-maven-plugin.version}</version>
                    <executions>
                        <execution>
                            <goals>
                                <goal>prepare-agent</goal>
                            </goals>
                        </execution>
                        <execution>
                            <id>report</id>
                            <phase>test</phase>
                            <goals>
                                <goal>report</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </pluginManagement>

        <!-- Global Build Plugins (Active for all modules) -->
        <plugins>
            <!-- Enforce Clean Build Constraints -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-enforcer-plugin</artifactId>
                <version>${maven-enforcer-plugin.version}</version>
                <executions>
                    <execution>
                        <id>enforce-rules</id>
                        <goals>
                            <goal>enforce</goal>
                        </goals>
                        <configuration>
                            <rules>
                                <requireMavenVersion>
                                    <version>[3.8.6,)</version>
                                </requireMavenVersion>
                                <requireJavaVersion>
                                    <version>[17,)</version>
                                </requireJavaVersion>
                                <!-- Prevent duplicate/conflicting dependencies across transitive trees -->
                                <dependencyConvergence />
                            </rules>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

---

### Platform BOM `platform-bom/pom.xml`

A standalone BOM allows internal modules and external projects to import your shared library versions without tightly coupling to your parent POM.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.enterprise.platform</groupId>
    <artifactId>platform-bom</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>Enterprise Platform :: BOM</name>
    <description>Bill of Materials for all internal platform modules</description>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.enterprise.platform</groupId>
                <artifactId>common-core</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.enterprise.platform</groupId>
                <artifactId>common-security</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

---

### Common Framework `common-core/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.enterprise.platform</groupId>
        <artifactId>enterprise-platform</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <artifactId>common-core</artifactId>
    <packaging>jar</packaging>

    <name>Enterprise Platform :: Common Core</name>

    <dependencies>
        <!-- Notice NO <version> tags here - versions inherited from parent dependencyManagement -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

---

### Consuming Service `service-orders/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.enterprise.platform</groupId>
        <artifactId>enterprise-platform</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <artifactId>service-orders</artifactId>
    <packaging>jar</packaging>

    <name>Enterprise Platform :: Service Orders</name>

    <dependencies>
        <!-- 1. Consume Internal Common Frameworks (No versions specified) -->
        <dependency>
            <groupId>com.enterprise.platform</groupId>
            <artifactId>common-core</artifactId>
        </dependency>
        <dependency>
            <groupId>com.enterprise.platform</groupId>
            <artifactId>common-security</artifactId>
        </dependency>

        <!-- 2. Web Tier Starter (Version managed by Spring Boot BOM) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Spring Boot packaging plugin -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## 4. Dependency Management Rules

### 1. Never specify `<version>` in child modules
- All versions belong in the Root POM `<dependencyManagement>` or `<properties>` block.
- Specifying `<version>` in child modules causes **version drift**, where different sub-projects use mismatched versions of third-party libraries (e.g., Spring 3.1 vs 3.2), causing runtime `NoSuchMethodError` or `ClassNotFoundException`.

### 2. Enforce Dependency Convergence
Use `maven-enforcer-plugin` with `<dependencyConvergence/>`. If `common-core` pulls Jackson `2.15.0` transitively, but `service-orders` pulls Jackson `2.16.1`, the enforcer immediately fails the build with a clear explanation, preventing hidden classpath bugs.

### 3. Manage Plugin Versions Centrally
Always declare plugin versions inside `<pluginManagement>` in the root POM. Child POMs should simply specify `<groupId>` and `<artifactId>`.

---

## 5. CI/CD & Build Performance Tuning

### Parallel Multi-Threaded Builds
Maven can analyze the dependency graph between your modules and compile independent modules in parallel across CPU cores:

```bash
# Build with 1 thread per available CPU core
mvn clean install -T 1C

# Build with exactly 4 threads
mvn clean install -T 4
```

### Building Subset of Modules (Reactor Flags)
You do not need to rebuild all 20 modules when only modifying one service:

```bash
# Build only service-orders and all modules it depends on (e.g. common-core)
mvn clean install -pl :service-orders -am

# Build a module and all downstream modules that depend on it
mvn clean install -pl :common-core -amd
```
- `-pl` / `--projects`: Comma-separated list of module artifactIds or relative paths.
- `-am` / `--also-make`: Also build prerequisite modules.
- `-amd` / `--also-make-dependents`: Also build dependent modules.

---

## 6. Common Anti-Patterns to Avoid

| Anti-Pattern | Why It Breaks | Modern Fix |
|---|---|---|
| **Circular Module Dependencies** | Module A depends on Module B, and Module B depends on Module A | Extract shared classes into a new `common-core` module |
| **Hardcoded Versions in Children** | Causes version mismatch and silent classpath conflicts | Put all versions in `<dependencyManagement>` |
| **Monolithic "god" common module** | Dumping everything into one giant `common.jar` forces heavy DB/security dependencies on light services | Split into focused modules: `common-core`, `common-security`, `common-persistence` |
| **Skipping BOM for external clients** | Forces external apps to inherit your entire internal parent POM | Publish a clean `platform-bom` |
| **Using `LATEST` or `RELEASE` versions** | Non-deterministic, un-reproducible builds | Use fixed semantic versions |
