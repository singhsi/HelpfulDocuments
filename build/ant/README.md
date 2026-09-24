# Apache Ant + Ivy — Modular Project Architecture & Best Practices

A pragmatic guide for organizing, maintaining, and modernizing multi-module enterprise Java projects using **Apache Ant** with **Apache Ivy** for dependency management.

> **See also:** [commands.md](commands.md) for quick-lookup Ant CLI commands, property overrides, and Ivy tasks.

---

## Table of Contents

- [1. Overview: Ant in Modern Enterprise Systems](#1-overview-ant-in-modern-enterprise-systems)
- [2. Multi-Module Project Architecture with Ant & Ivy](#2-multi-module-project-architecture-with-ant--ivy)
  - [Why Apache Ivy is Essential](#why-apache-ivy-is-essential)
  - [Master Build Script vs Sub-Module Builds](#master-build-script-vs-sub-module-builds)
- [3. Production Code Example](#3-production-code-example)
  - [Directory Layout](#directory-layout)
  - [Global Shared Properties (`common-build.properties`)](#global-shared-properties-common-buildproperties)
  - [Shared Common Targets (`common-build.xml`)](#shared-common-targets-common-buildxml)
  - [Master Aggregator Script (`build.xml`)](#master-aggregator-script-buildxml)
  - [Sub-Module Build Script (`common-core/build.xml`)](#sub-module-build-script-common-corebuildxml)
  - [Sub-Module Ivy Descriptor (`common-core/ivy.xml`)](#sub-module-ivy-descriptor-common-coreivyxml)
- [4. Enterprise Best Practices](#4-enterprise-best-practices)
  - [Target Granularity & Standard Lifecycle Names](#target-granularity--standard-lifecycle-names)
  - [Centralized Version Properties](#centralized-version-properties)
  - [Avoid Absolute File Paths](#avoid-absolute-file-paths)
- [5. Modernizing Legacy Ant Projects](#5-modernizing-legacy-ant-projects)
  - [Step 1: Replace committed `/lib/*.jar` with Apache Ivy](#step-1-replace-committed-libjar-with-apache-ivy)
  - [Step 2: Adopt standard Maven directory layout (`src/main/java`)](#step-2-adopt-standard-maven-directory-layout-srcmainjava)
  - [Step 3: Migration path to Maven or Gradle](#step-3-migration-path-to-maven-or-gradle)

---

## 1. Overview: Ant in Modern Enterprise Systems

Apache Ant is a procedural task-runner that provides total flexibility. However, without strict architectural conventions, Ant projects historically degraded into unmaintainable, thousands-of-lines XML files with committed binary JARs in source control.

To make an Ant multi-module project **production-grade**:
1. **Use Apache Ivy** to resolve dependencies from Maven Central rather than checking JARs into Git.
2. **Use a Shared Common Build XML** (`common-build.xml`) imported via `<import file="..."/>` to prevent copying `javac` and `junit` tasks into every sub-folder.
3. **Use a Master Orchestrator** to build dependencies in the correct order using `<subant>` or `<ant dir="..." target="..."/>`.

---

## 2. Multi-Module Project Architecture with Ant & Ivy

```
enterprise-platform/
├── build.xml                         <-- Master orchestrator build script
├── common-build.xml                  <-- Reusable compilation/testing macro targets
├── common-build.properties           <-- Global properties (Java version, paths)
├── ivysettings.xml                   <-- Repository config (Maven Central, Artifactory)
├── common-core/                      <-- Shared utility library
│   ├── build.xml
│   ├── ivy.xml                       <-- Dependencies (Jackson, SLF4J, JUnit)
│   └── src/
├── common-security/                  <-- Shared authentication library
│   ├── build.xml
│   ├── ivy.xml
│   └── src/
└── service-orders/                   <-- Consuming application (WAR / JAR)
    ├── build.xml
    ├── ivy.xml                       <-- Depends on common-core & security
    └── src/
```

---

## 3. Production Code Example

### Global Shared Properties (`common-build.properties`)

```properties
# Java Compiler Settings
java.source=17
java.target=17
build.encoding=UTF-8

# Directory Conventions
src.dir=src/main/java
resources.dir=src/main/resources
test.src.dir=src/test/java
test.resources.dir=src/test/resources

target.dir=target
classes.dir=${target.dir}/classes
test.classes.dir=${target.dir}/test-classes
dist.dir=${target.dir}/dist
report.dir=${target.dir}/reports
lib.dir=lib

# Dependency Versions (Centralized)
version.slf4j=2.0.12
version.jackson=2.16.1
version.junit=5.10.2
```

---

### Shared Common Targets (`common-build.xml`)

Reusable tasks imported by every sub-module:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project name="common-build" xmlns:ivy="antlib:org.apache.ivy.ant">

    <property file="${basedir}/../common-build.properties" />

    <!-- 1. Classpaths -->
    <path id="compile.classpath">
        <fileset dir="${lib.dir}/compile" erroronmissingdir="false">
            <include name="*.jar" />
        </fileset>
    </path>

    <path id="runtime.classpath">
        <path refid="compile.classpath" />
        <fileset dir="${lib.dir}/runtime" erroronmissingdir="false">
            <include name="*.jar" />
        </fileset>
        <pathelement location="${classes.dir}" />
    </path>

    <path id="test.classpath">
        <path refid="runtime.classpath" />
        <fileset dir="${lib.dir}/test" erroronmissingdir="false">
            <include name="*.jar" />
        </fileset>
        <pathelement location="${test.classes.dir}" />
    </path>

    <!-- 2. Clean Target -->
    <target name="clean" description="Deletes build directories">
        <delete dir="${target.dir}" />
        <delete dir="${lib.dir}" />
    </target>

    <!-- 3. Init Target -->
    <target name="init">
        <mkdir dir="${classes.dir}" />
        <mkdir dir="${test.classes.dir}" />
        <mkdir dir="${dist.dir}" />
        <mkdir dir="${report.dir}" />
    </target>

    <!-- 4. Ivy Resolve Dependencies -->
    <target name="resolve" description="Retrieve dependencies via Ivy">
        <ivy:resolve file="ivy.xml" />
        <ivy:retrieve pattern="${lib.dir}/[conf]/[artifact]-[revision].[ext]" conf="compile,runtime,test" />
    </target>

    <!-- 5. Compile Java Source -->
    <target name="compile" depends="init,resolve" description="Compiles Java source files">
        <javac srcdir="${src.dir}" 
               destdir="${classes.dir}" 
               source="${java.source}" 
               target="${java.target}" 
               encoding="${build.encoding}" 
               includeantruntime="false" 
               debug="true">
            <classpath refid="compile.classpath" />
        </javac>
        <copy todir="${classes.dir}">
            <fileset dir="${resources.dir}" erroronmissingdir="false" />
        </copy>
    </target>

    <!-- 6. Compile & Run Tests -->
    <target name="test" depends="compile" description="Executes unit tests">
        <javac srcdir="${test.src.dir}" 
               destdir="${test.classes.dir}" 
               source="${java.source}" 
               target="${java.target}" 
               encoding="${build.encoding}" 
               includeantruntime="false">
            <classpath refid="test.classpath" />
        </javac>

        <junitlauncher haltonfailure="true" printSummary="true">
            <classpath refid="test.classpath" />
            <testclasses outputdir="${report.dir}">
                <fileset dir="${test.classes.dir}">
                    <include name="**/*Test.class" />
                </fileset>
                <listener type="legacy-plain" sendSysOut="true" />
            </testclasses>
        </junitlauncher>
    </target>
</project>
```

---

### Master Aggregator Script (`build.xml`)

Controls execution order across all modules:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project name="enterprise-platform-master" default="build-all">

    <target name="clean-all" description="Cleans all modules">
        <ant dir="common-core" target="clean" inheritAll="false" />
        <ant dir="common-security" target="clean" inheritAll="false" />
        <ant dir="service-orders" target="clean" inheritAll="false" />
    </target>

    <target name="build-common-core">
        <ant dir="common-core" target="jar" inheritAll="false" />
    </target>

    <target name="build-common-security" depends="build-common-core">
        <ant dir="common-security" target="jar" inheritAll="false" />
    </target>

    <target name="build-service-orders" depends="build-common-security">
        <ant dir="service-orders" target="package" inheritAll="false" />
    </target>

    <target name="build-all" depends="build-service-orders" description="Builds entire platform in dependency order" />
</project>
```

---

### Sub-Module Build Script (`common-core/build.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project name="common-core" default="jar" xmlns:ivy="antlib:org.apache.ivy.ant">

    <!-- Import shared macro targets -->
    <import file="../common-build.xml" />

    <property name="jar.name" value="${dist.dir}/common-core-1.0.0.jar" />

    <target name="jar" depends="test" description="Packages common-core into a JAR">
        <jar destfile="${jar.name}" basedir="${classes.dir}">
            <manifest>
                <attribute name="Implementation-Title" value="Enterprise Platform :: Common Core" />
                <attribute name="Implementation-Version" value="1.0.0" />
            </manifest>
        </jar>
        
        <!-- Copy built JAR to local module repository so other modules can consume it -->
        <mkdir dir="../build-repo/common-core" />
        <copy file="${jar.name}" todir="../build-repo/common-core/" />
    </target>
</project>
```

---

### Sub-Module Ivy Descriptor (`common-core/ivy.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ivy-module version="2.0">
    <info organisation="com.enterprise.platform" module="common-core" revision="1.0.0" />

    <configurations>
        <conf name="compile" description="Required to compile" />
        <conf name="runtime" extends="compile" description="Required at runtime" />
        <conf name="test" extends="runtime" description="Required for testing" />
    </configurations>

    <dependencies>
        <!-- Third-party libraries resolved from Maven Central -->
        <dependency org="org.slf4j" name="slf4j-api" rev="2.0.12" conf="compile->default" />
        <dependency org="com.fasterxml.jackson.core" name="jackson-databind" rev="2.16.1" conf="compile->default" />
        <dependency org="org.junit.jupiter" name="junit-jupiter" rev="5.10.2" conf="test->default" />
    </dependencies>
</ivy-module>
```

---

## 4. Enterprise Best Practices

1. **Standardize Target Names:** Every sub-module should implement standard lifecycle targets: `clean`, `resolve`, `compile`, `test`, `jar`/`package`.
2. **Use `inheritAll="false"`:** When delegating tasks with `<ant dir="..." />`, always use `inheritAll="false"` to prevent property pollution from the master build into child builds.
3. **Never Check Binary JARs into Version Control:** Use Apache Ivy with an `ivysettings.xml` pointing to your company's Nexus/Artifactory repository.
4. **Use `<junitlauncher>` (JUnit 5):** Upgrade from legacy `<junit>` (JUnit 4) to modern `<junitlauncher>`.

---

## 5. Modernizing Legacy Ant Projects

If you are maintaining a legacy Ant build and plan to modernize:

| Phase | Action |
|---|---|
| **Phase 1 (Immediate)** | Introduce **Apache Ivy** to replace static checked-in `/lib/*.jar` files. |
| **Phase 2 (Structure)** | Adopt standard Maven/Gradle folder conventions (`src/main/java`, `src/test/java`). |
| **Phase 3 (Migration)** | Generate a `pom.xml` or `build.gradle.kts`. Since directory layouts and Ivy dependencies are already standardized, the migration will be straightforward and low-risk. |
