# Core Java SE & CLI Runtimes Guide

A pure, fundamental guide to Java Standard Edition (Java SE) from first principles — zero IDEs, zero build tools. Understand the Java compilation model, CLI executables (`java`, `javac`, `javaw`, `jar`, `javap`, `jdb`, `jcmd`), the classpath mechanism, manifest classpaths, and how production enterprise software structures and executes classpaths on the command line.

> **Related Guides:**
> - [performance.md](performance.md) — Java performance tuning, GC selection, JVM presets, and memory optimization
> - [commands.md](commands.md) — Quick command-line syntax and CLI cheat sheet

---

## Table of Contents

- [1. The Core Java Execution Model (Source to CPU)](#1-the-core-java-execution-model-source-to-cpu)
  - [JDK vs JRE vs JVM](#jdk-vs-jre-vs-jvm)
  - [Bytecode, Classloaders, and JIT Compilation](#bytecode-classloaders-and-jit-compilation)
- [2. Java CLI Tooling Breakdown](#2-java-cli-tooling-breakdown)
  - [`javac` — Java Compiler](#javac--java-compiler)
  - [`java` — Java Application Launcher (JVM)](#java--java-application-launcher-jvm)
  - [`javaw` — Windowed Java Launcher (No Console)](#javaw--windowed-java-launcher-no-console)
  - [`jar` — Java Archive Tool](#jar--java-archive-tool)
  - [`javap` — Class File Disassembler / Bytecode Viewer](#javap--class-file-disassembler--bytecode-viewer)
  - [`jdb` — Java Debugger](#jdb--java-debugger)
  - [`jcmd` & `jstat` — Diagnostic & Monitoring Utilities](#jcmd--jstat--diagnostic--monitoring-utilities)
- [3. What is the Classpath?](#3-what-is-the-classpath)
  - [The Concept](#the-concept)
  - [Classpath Syntax & OS Delimiters](#classpath-syntax--os-delimiters)
  - [Classpath vs Modulepath (Java 9+)](#classpath-vs-modulepath-java-9)
- [4. Pure CLI Compilation & Execution Patterns](#4-pure-cli-compilation--execution-patterns)
  - [Single File Execution (Java 11+ Shebang & Direct Run)](#single-file-execution-java-11-shebang--direct-run)
  - [Multi-Package Compilation with Output Directory (`-d`)](#multi-package-compilation-with-output-directory--d)
  - [Compiling with External Third-Party JARs](#compiling-with-external-third-party-jars)
  - [Building and Running Executable JARs (`META-INF/MANIFEST.MF`)](#building-and-running-executable-jars-meta-infmanifestmf)
- [5. How Classpath is Set in Large Real-World Java Software](#5-how-classpath-is-set-in-large-real-world-java-software)
  - [1. Shell Wrapper Scripts (`startup.sh` / `run.sh`)](#1-shell-wrapper-scripts-startupsh--runsh)
  - [2. Wildcard Classpath Directory Loading (`lib/*`)](#2-wildcard-classpath-directory-loading-lib)
  - [3. Manifest Classpath (`Class-Path:` Header)](#3-manifest-classpath-class-path-header)
  - [4. Argument Files (`@argument_file`) to Avoid OS Length Limits](#4-argument-files-argument_file-to-avoid-os-length-limits)
  - [5. Custom Classloaders & Plugin Architecture](#5-custom-classloaders--plugin-architecture)
- [6. Common Classpath Errors & Troubleshooting](#6-common-classpath-errors--troubleshooting)

---

## 1. The Core Java Execution Model (Source to CPU)

Java follows the **"Write Once, Run Anywhere" (WORA)** philosophy through an intermediate compilation step:

```
┌─────────────────┐       javac        ┌─────────────────┐        java (JVM)       ┌─────────────────┐
│   Source Code   │ ─────────────────> │    Bytecode     │ ──────────────────────> │ Native Machine  │
│   (App.java)    │                    │   (App.class)   │  [Interpreter + JIT]    │ Instructions    │
└─────────────────┘                    └─────────────────┘                         └─────────────────┘
```

### JDK vs JRE vs JVM

| Component | Full Name | What It Contains | Target Audience |
|---|---|---|---|
| **JVM** | Java Virtual Machine | The execution engine: Classloader subsystem, Garbage Collector, JIT compiler, and runtime memory (Heap, Metaspace, Stacks). | System runtime |
| **JRE** | Java Runtime Environment | JVM + Standard Class Libraries (`java.base`, `java.sql`, etc.). *(Note: Post-Java 11, monolithic JRE downloads are replaced by custom runtimes via `jlink` or full JDKs).* | End users running apps |
| **JDK** | Java Development Kit | JRE/JVM + Developer tools (`javac`, `jar`, `javap`, `jdb`, `jcmd`, headers). | Developers writing & compiling code |

### Bytecode, Classloaders, and JIT Compilation
1. **Source Compilation:** `javac` parses `.java` source files, verifies syntax, and produces architecture-neutral `.class` files containing JVM bytecode instructions.
2. **Class Loading:** When running `java com.example.Main`, the JVM uses hierarchical classloaders (**Bootstrap -> Platform/Extension -> Application/System**) to find and load `.class` files into memory on demand.
3. **Execution & JIT Compilation:** The JVM initially interprets bytecode. Hot code paths (frequently executed loops/methods) are compiled on-the-fly into native CPU machine code by the **Just-In-Time (JIT) Compiler** (Tiered Compilation: C1 compiler for fast startup, C2 compiler for high peak performance).

---

## 2. Java CLI Tooling Breakdown

The JDK `bin/` directory contains several essential command-line tools:

```
$JAVA_HOME/bin/
├── javac      # Compiles .java to .class
├── java       # Launches the JVM console applications
├── javaw      # Launches the JVM without a console window (GUI apps on Windows)
├── jar        # Packages classes and resources into compressed zip/jar archives
├── javap      # Disassembles .class files to inspect bytecode and signatures
├── jdb        # Command-line interactive debugger
└── jcmd       # Sends diagnostic command requests to a running JVM
```

---

### `javac` — Java Compiler

`javac` reads Java declarations and statements and compiles them into bytecode class files.

```bash
# Basic compilation
javac HelloWorld.java

# Compile with destination directory and sourcepath
javac -d out -sourcepath src src/com/example/Main.java

# Compile using specific Java language release syntax
javac --release 17 -d out src/com/example/Main.java
```

---

### `java` — Java Application Launcher (JVM)

`java` starts a Java Virtual Machine instance, loads the specified class, and invokes its `public static void main(String[] args)` method.

```bash
# Execute a compiled class
java -cp out com.example.Main

# Execute an executable JAR file
java -jar app.jar arg1 arg2

# Pass JVM memory arguments and system properties
java -Xms512m -Xmx2048m -Dapp.env=production -cp out com.example.Main
```

---

### `javaw` — Windowed Java Launcher (No Console)

- On **Windows**, `javaw.exe` is identical to `java.exe`, except it runs as a Windows GUI process rather than a console application.
- It does **not** associate with a console window and does not wait for standard input/output to close.
- **When used:** Desktop GUI applications (Swing, JavaFX, SWT) where you do not want an empty black command prompt window staying open in the background.
- On **Linux / macOS**, there is generally no separate `javaw` binary; running `java &` or backgrounding via standard process management serves the same purpose.

---

### `jar` — Java Archive Tool

Combines multiple `.class` files, metadata, and resources into a single ZIP-compressed archive file with a `.jar` extension.

```bash
# Create an archive (c = create, v = verbose, f = filename)
jar cvf mylib.jar -C out/ .

# Create an executable JAR with a custom Manifest
jar cvfm myapp.jar src/META-INF/MANIFEST.MF -C out/ .

# Extract archive contents
jar xvf mylib.jar

# List contents of an archive
jar tf mylib.jar
```

---

### `javap` — Class File Disassembler / Bytecode Viewer

Disassembles compiled `.class` files to inspect method signatures, constants, and JVM bytecode instructions without needing source code.

```bash
# Print public method signatures (useful to verify API)
javap com.example.User

# Print internal type signatures and private members
javap -p -s com.example.User

# Disassemble full JVM bytecode instructions
javap -c -v com.example.User
```

---

### `jdb` — Java Debugger

A CLI-based interactive debugger that can attach to running JVM processes or launch classes directly.

```bash
# Launch a class with jdb
jdb -classpath out com.example.Main

# Inside jdb:
# stop in com.example.Main.main
# run
# step
# print variableName
```

---

### `jcmd` & `jstat` — Diagnostic & Monitoring Utilities

```bash
# List all running Java processes and their PIDs
jcmd

# Trigger thread dump from CLI
jcmd <PID> Thread.print

# Trigger GC heap inspection
jcmd <PID> GC.class_histogram

# Monitor Garbage Collector memory statistics every 1000ms
jstat -gcutil <PID> 1000
```

---

## 3. What is the Classpath?

### The Concept
The **Classpath** is the search path that the Java Virtual Machine and the Java compiler use to locate `.class` files, packages, and resources during compilation and execution.

When your code says:
```java
import com.example.service.OrderService;
```
The JVM does **not** scan the entire hard drive. It looks strictly at the entries defined on the **Classpath**, replaces package dots with directory slashes (`com/example/service/OrderService.class`), and searches each classpath entry from left to right.

---

### Classpath Syntax & OS Delimiters

Classpath entries can be:
1. **Directories** containing package roots (e.g., `out/`, `bin/`, `/var/app/classes`).
2. **JAR / ZIP files** containing `.class` files (e.g., `lib/gson-2.10.1.jar`).
3. **Wildcard directory expansion** (e.g., `lib/*` loads all `.jar` files in that folder).

#### Operating System Delimiters:

| Operating System | Path Separator (Separator between classpath entries) | Example Syntax |
|---|---|---|
| **Linux / macOS / Unix** | Colon (`:`) | `-cp "out:lib/log4j.jar:lib/*"` |
| **Windows** | Semicolon (`;`) | `-cp "out;lib\log4j.jar;lib\*"` |

> **Critical Rule:** When using wildcard syntax (`lib/*`), do **not** write `lib/*.jar`. The JVM wildcard format is `lib/*` or `"lib/*"`.

---

### Classpath vs Modulepath (Java 9+)

Starting with Java 9 (Project Jigsaw):
- **Classpath (`-cp` / `--class-path`):** Legacy, flat namespace. All classes are visible to all other classes. No module-level encapsulation.
- **Modulepath (`-p` / `--module-path`):** Modular JARs containing `module-info.class`. Enforces strict explicit export and require contracts.

Most enterprise software continues to utilize the classpath or a hybrid modulepath approach.

---

## 4. Pure CLI Compilation & Execution Patterns

### Single File Execution (Java 11+ Direct Run)

Since Java 11, you can execute a single `.java` source file without manually invoking `javac` first:

```bash
# Runs directly in memory without producing a .class file on disk
java HelloWorld.java

# On Linux, you can even use a Shebang at the top of the file:
# !/usr/bin/env java --source 17
```

---

### Multi-Package Compilation with Output Directory (`-d`)

Given this directory layout:
```
myproject/
├── src/
│   └── com/
│       └── enterprise/
│           ├── model/
│           │   └── User.java
│           └── Main.java
```

#### Compilation:
```bash
# 1. Create output folder
mkdir -p out

# 2. Compile specifying source directory and output directory (-d)
javac -d out -sourcepath src src/com/enterprise/Main.java src/com/enterprise/model/User.java

# Or compile all .java files recursively in Linux/macOS:
javac -d out $(find src -name "*.java")
```

#### Execution:
```bash
# Provide output folder to -cp and run the fully-qualified class name
java -cp out com.enterprise.Main
```

---

### Compiling with External Third-Party JARs

Given third-party libraries in a `lib/` directory:
```
myproject/
├── lib/
│   ├── slf4j-api-2.0.12.jar
│   └── gson-2.10.1.jar
├── src/
│   └── com/enterprise/Main.java
```

#### Compilation:
```bash
# Linux / macOS (colon separator)
javac -d out -cp "lib/*" src/com/enterprise/Main.java

# Windows (semicolon separator)
javac -d out -cp "lib/*" src\com\enterprise\Main.java
```

#### Execution:
```bash
# Linux / macOS: combine out directory and library directory
java -cp "out:lib/*" com.enterprise.Main

# Windows:
java -cp "out;lib/*" com.enterprise.Main
```

---

### Building and Running Executable JARs (`META-INF/MANIFEST.MF`)

An executable JAR contains a `META-INF/MANIFEST.MF` text file specifying the class that contains the `main` method.

#### 1. Create Manifest file (`manifest.txt`):
```text
Manifest-Version: 1.0
Main-Class: com.enterprise.Main

```
*(Note: Manifest files MUST end with a trailing newline).*

#### 2. Package into JAR:
```bash
jar cvfm app.jar manifest.txt -C out .
```

#### 3. Run:
```bash
java -jar app.jar
```

---

## 5. How Classpath is Set in Large Real-World Java Software

In large production systems (monoliths, banking systems, backend batch jobs) without container layers or IDEs, classpath management is automated via clean shell/batch scripts.

```
/opt/enterprise-app/
├── bin/
│   ├── start.sh                      <-- Classpath assembly & JVM launcher
│   └── stop.sh
├── conf/
│   ├── log4j2.xml                    <-- Config files loaded via Classpath
│   └── application.properties
├── lib/                              <-- 150+ third-party & module JAR files
│   ├── enterprise-core.jar
│   ├── enterprise-services.jar
│   ├── spring-core-6.1.4.jar
│   └── ...
└── logs/
```

---

### 1. Enterprise Batch Scripts (Windows `.bat` / `.cmd` and Linux `.sh`)

In enterprise legacy systems, batch jobs, and standalone tools, developers often write startup scripts that build a cumulative classpath variable (e.g. `ADD_CP` or `CLASSPATH`) by appending each required JAR and folder entry one by one.

#### Windows Batch Example (`runBatch.bat`)

Here is a real-world enterprise pattern using `%~dp0` (the directory where the script is located) to build a dynamic classpath and launch a Java application:

```bat
@echo off
setlocal enabledelayedexpansion

:: 1. Resolve Home Directory based on script location
SET BATCH_HOME=%~dp0
SET LIB_DIR=%BATCH_HOME%\lib
SET CONF_DIR=%BATCH_HOME%\conf

:: 2. Initialize Classpath with configuration directory and application main JAR
SET ADD_CP=%CONF_DIR%;%BATCH_HOME%\order-processing-batch.jar

:: 3. Append required framework and library JARs explicitly
SET ADD_CP=!ADD_CP!;%LIB_DIR%\security-crypto-2.1.jar
SET ADD_CP=!ADD_CP!;%LIB_DIR%\audit-logger-1.4.jar
SET ADD_CP=!ADD_CP!;%LIB_DIR%\database-pool-3.0.jar
SET ADD_CP=!ADD_CP!;%LIB_DIR%\transaction-manager-5.1.jar
SET ADD_CP=!ADD_CP!;%LIB_DIR%\json-parser-2.16.jar
SET ADD_CP=!ADD_CP!;%LIB_DIR%\sql-driver-8.0.jar
SET ADD_CP=!ADD_CP!;%LIB_DIR%\metrics-collector-4.2.jar

:: 4. Set JVM Memory, System Properties, and Main Class
SET JVM_OPTS=-Xms1024m -Xmx2048m -XX:+UseG1GC
SET SYS_PROPS=-Dbatch.home="%BATCH_HOME%" -Dfile.encoding=UTF-8
SET MAIN_CLASS=com.enterprise.batch.OrderProcessingApplication

:: 5. Execute Java Program with the assembled Classpath
echo Starting Order Processing Batch Application...
java %JVM_OPTS% %SYS_PROPS% -cp "!ADD_CP!" %MAIN_CLASS% %*

:: Check Exit Code
if %ERRORLEVEL% NEQ 0 (
    echo Batch execution failed with exit code %ERRORLEVEL%
    exit /b %ERRORLEVEL%
)
```

#### Linux Shell Equivalent (`runBatch.sh`)

```bash
#!/usr/bin/env bash
set -e

# 1. Resolve Directory of Script
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LIB_DIR="${SCRIPT_DIR}/lib"
CONF_DIR="${SCRIPT_DIR}/conf"

# 2. Initialize Classpath
ADD_CP="${CONF_DIR}:${SCRIPT_DIR}/order-processing-batch.jar"

# 3. Append required framework and library JARs
ADD_CP="${ADD_CP}:${LIB_DIR}/security-crypto-2.1.jar"
ADD_CP="${ADD_CP}:${LIB_DIR}/audit-logger-1.4.jar"
ADD_CP="${ADD_CP}:${LIB_DIR}/database-pool-3.0.jar"
ADD_CP="${ADD_CP}:${LIB_DIR}/transaction-manager-5.1.jar"
ADD_CP="${ADD_CP}:${LIB_DIR}/json-parser-2.16.jar"
ADD_CP="${ADD_CP}:${LIB_DIR}/sql-driver-8.0.jar"
ADD_CP="${ADD_CP}:${LIB_DIR}/metrics-collector-4.2.jar"

# 4. JVM Options & Main Class
JVM_OPTS="-Xms1024m -Xmx2048m -XX:+UseG1GC"
SYS_PROPS="-Dbatch.home=${SCRIPT_DIR} -Dfile.encoding=UTF-8"
MAIN_CLASS="com.enterprise.batch.OrderProcessingApplication"

# 5. Execute
echo "Starting Order Processing Batch Application..."
exec java ${JVM_OPTS} ${SYS_PROPS} -cp "${ADD_CP}" ${MAIN_CLASS} "$@"
```

#### Explanation of Key Concepts in the Batch Script:
- **`%~dp0` (Windows) / `$(dirname "${BASH_SOURCE[0]}")` (Linux):** Dynamically points to the folder containing the `.bat` or `.sh` script. This ensures the script works regardless of which folder the user is in when executing it.
- **Incremental Appending (`SET ADD_CP=%ADD_CP%;...`):** Adds each JAR to the search chain while preserving the exact order.
- **Separators:** Semicolon (`;`) for Windows vs Colon (`:`) for Linux/Unix.
- **Double Quotes (`-cp "%ADD_CP%"`)**: Essential on Windows when directory paths contain spaces (e.g. `C:\Program Files\...`).
- **`%*` / `"$@"`**: Forwards any command-line parameters passed to the script directly to the Java `main(String[] args)` method.

---

### 2. Wildcard Classpath Directory Loading (`lib/*`)

- Introduced in Java 6 to replace mile-long strings of `lib/a.jar:lib/b.jar:lib/c.jar...`.
- `java -cp "lib/*"` automatically expands to all `.jar` and `.JAR` files within the `lib/` directory.
- **Important:** Wildcards do **not** search sub-directories recursively (e.g. `lib/subfolder/` is not searched).

---

### 3. Manifest Classpath (`Class-Path:` Header)

In large distributed packages, the main application JAR can specify its dependent libraries inside its own `MANIFEST.MF`:

```text
Manifest-Version: 1.0
Main-Class: com.enterprise.Main
Class-Path: lib/common-core.jar lib/slf4j-api-2.0.12.jar lib/gson-2.10.1.jar

```

When running `java -jar app.jar`, the JVM automatically loads all relative JARs specified in `Class-Path:`.

> **Rule:** If you use `java -jar app.jar`, the `-cp` command-line argument is **completely ignored by the JVM**. All classpath entries must reside in the manifest or system classloader.

---

### 4. Argument Files (`@argument_file`) to Avoid OS Length Limits

On Windows and some Unix systems, the maximum command-line character length (e.g., 8191 characters on Windows command prompt) can be exceeded when listing hundreds of explicit JAR files.

Java allows passing an **argument file** prefixed with `@`:

```bash
# Create an argument file (args.txt)
echo "-cp" > args.txt
echo "/opt/app/conf:/opt/app/lib/*" >> args.txt
echo "com.enterprise.Main" >> args.txt

# Execute using argument file
java @args.txt
```

---

### 5. Custom Classloaders & Plugin Architecture

Large software (application servers, IDEs, plugin systems) does not rely solely on the static flat CLI classpath. They instantiate `java.net.URLClassLoader`:

```java
// Dynamically load a plugin JAR at runtime from disk without restarting JVM
File pluginJar = new File("/opt/plugins/custom-connector.jar");
URL[] urls = new URL[]{ pluginJar.toURI().toURL() };

URLClassLoader pluginClassLoader = new URLClassLoader(urls, this.getClass().getClassLoader());
Class<?> pluginClass = Class.forName("com.plugin.CustomConnector", true, pluginClassLoader);
Object pluginInstance = pluginClass.getDeclaredConstructor().newInstance();
```

---

## 6. Common Classpath Errors & Troubleshooting

| Error | Root Cause | Fix |
|---|---|---|
| `ClassNotFoundException: com.foo.Bar` | Dynamic runtime lookup (`Class.forName`) could not locate the `.class` file on the active classpath. | Verify the JAR containing `com.foo.Bar` is included in `-cp` and spelled correctly. |
| `NoClassDefFoundError: com.foo.Bar` | The class was present at compile time, but missing from the runtime classpath, OR class static initialization failed. | Check runtime `-cp` or check logs for earlier static block `<clinit>` exceptions. |
| `NoSuchMethodError` / `NoSuchFieldError` | **Classpath collision:** Multiple conflicting versions of the same library exist on the classpath (e.g., Library v1.0 and Library v2.0). The JVM loaded the wrong one first. | Inspect classpath order; ensure only one version of each JAR exists in `lib/`. |
| `Could not find or load main class` | 1. Classpath does not point to the root package folder.<br>2. Ran `java com/foo/Main.class` instead of `java com.foo.Main`.<br>3. `package` statement in `.java` does not match folder structure. | Pass directory containing the root package to `-cp` and use dot-separated class name without `.class`. |
| Manifest trailing line error | `MANIFEST.MF` does not end with a blank line / carriage return. | Open manifest file and press `Enter` to ensure a final trailing newline exists. |
