# Java SE CLI — Command & Tool Reference

A quick-lookup cheat sheet for compiling, running, archiving, disassembling, and debugging Java SE programs purely via the command line.

---

## Table of Contents

- [Compilation (`javac`)](#compilation-javac)
- [Execution (`java` & `javaw`)](#execution-java--javaw)
- [Packaging (`jar`)](#packaging-jar)
- [Bytecode & Class Inspection (`javap`)](#bytecode--class-inspection-javap)
- [Debugging (`jdb`)](#debugging-jdb)
- [Diagnostics & JVM Inspection (`jcmd`, `jstat`, `jstack`)](#diagnostics--jvm-inspection-jcmd-jstat-jstack)
- [Classpath Recipes for Linux & Windows](#classpath-recipes-for-linux--windows)

---

## Compilation (`javac`)

```bash
# Compile a single file into current directory
javac HelloWorld.java

# Compile with target output directory (-d)
javac -d out src/com/example/Main.java

# Compile with sourcepath and multiple packages
javac -d out -sourcepath src src/com/example/Main.java

# Compile all .java files in a project (Linux / macOS)
javac -d out $(find src -name "*.java")

# Compile with external JAR libraries (Linux / macOS: colon)
javac -d out -cp "lib/*:lib/custom.jar" src/com/example/Main.java

# Compile with external JAR libraries (Windows: semicolon)
javac -d out -cp "lib/*;lib\custom.jar" src\com\example\Main.java

# Compile with specific Java language version compatibility
javac --release 17 -d out src/com/example/Main.java

# Compile with deprecation warnings and all compiler warnings
javac -Xlint:all -d out src/com/example/Main.java
```

---

## Execution (`java` & `javaw`)

```bash
# Run a compiled class from output folder
java -cp out com.example.Main

# Run with classpath containing directories and JARs (Linux / macOS)
java -cp "out:conf:lib/*" com.example.Main

# Run with classpath containing directories and JARs (Windows)
java -cp "out;conf;lib/*" com.example.Main

# Run an executable JAR directly
java -jar app.jar

# Run single source file directly without separate compile step (Java 11+)
java src/com/example/Main.java

# Pass system properties and JVM memory limits
java -Xms1024m -Xmx4096m -Denv=prod -Dconfig.path=/etc/app -cp out com.example.Main

# Run GUI application on Windows without console window
javaw -cp "out;lib/*" com.example.gui.MainFrame

# Enable remote debugging port on launch
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005 -cp out com.example.Main
```

---

## Packaging (`jar`)

```bash
# Create a standard JAR from compiled classes directory
jar cvf mylib.jar -C out/ .

# Create an executable JAR with a custom Manifest
jar cvfm myapp.jar MANIFEST.MF -C out/ .

# Create an executable JAR specifying main class directly (no manifest file needed)
jar cvfe myapp.jar com.example.Main -C out/ .

# List all files inside a JAR archive
jar tf myapp.jar

# Extract a JAR file
jar xvf myapp.jar

# Update / add a file to an existing JAR
jar uvf myapp.jar -C conf/ application.properties

# Inspect metadata / module info of a JAR
jar --describe-module --file myapp.jar
```

---

## Bytecode & Class Inspection (`javap`)

```bash
# Print public methods and fields of a class
javap com.example.Main

# Print private, protected, and public members (-p) with type signatures (-s)
javap -p -s com.example.Main

# Disassemble JVM bytecode instructions (-c)
javap -c com.example.Main

# Print complete detailed disassemble including constant pool and line numbers (-v)
javap -v -p com.example.Main
```

---

## Debugging (`jdb`)

```bash
# Launch class under interactive CLI debugger
jdb -classpath out com.example.Main

# Attach jdb to an already running JVM on debug port 5005
jdb -attach localhost:5005

# Useful jdb commands inside session:
# stop in com.example.Main.main       # Set breakpoint at method entry
# stop at com.example.Main:42         # Set breakpoint at line 42
# run                                 # Start execution
# step                                # Step into next line
# next                                # Step over next line
# print variableName                  # Inspect variable
# locals                              # Print local variables in scope
# cont                                # Continue execution
```

---

## Diagnostics & JVM Inspection (`jcmd`, `jstat`, `jstack`)

```bash
# List all running Java processes on the machine with their PIDs
jcmd -l

# Capture thread dump of a running Java process
jcmd <PID> Thread.print
# or:
jstack <PID>

# Inspect live heap class histogram (objects and byte count)
jcmd <PID> GC.class_histogram

# Trigger a manual full Garbage Collection
jcmd <PID> GC.run

# Inspect active JVM command-line flags and system properties
jcmd <PID> VM.command_line
jcmd <PID> VM.system_properties

# Monitor Garbage Collection metrics every 1 second (1000ms)
jstat -gcutil <PID> 1000
```

---

## Classpath Recipes for Linux & Windows

### Directory Structure:
```
app/
├── conf/
│   └── app.properties
├── lib/
│   ├── log4j-api-2.20.0.jar
│   └── log4j-core-2.20.0.jar
└── out/
    └── com/example/App.class
```

### Linux / macOS (Use `:` delimiter):
```bash
java -cp "conf:out:lib/*" com.example.App
```

### Windows (Use `;` delimiter and backslashes):
```cmd
java -cp "conf;out;lib\*" com.example.App
```

---

## Enterprise Cumulative Batch Classpath Recipe (`runBatch.bat`)

```bat
@echo off
setlocal enabledelayedexpansion

SET BATCH_HOME=%~dp0
SET LIB_DIR=%BATCH_HOME%\lib

SET ADD_CP=%BATCH_HOME%\conf;%BATCH_HOME%\app-main.jar
SET ADD_CP=!ADD_CP!;%LIB_DIR%\security-crypto-2.1.jar
SET ADD_CP=!ADD_CP!;%LIB_DIR%\audit-logger-1.4.jar
SET ADD_CP=!ADD_CP!;%LIB_DIR%\database-pool-3.0.jar
SET ADD_CP=!ADD_CP!;%LIB_DIR%\transaction-manager-5.1.jar

java -Xms1024m -Xmx2048m -cp "!ADD_CP!" com.enterprise.Main %*
```
