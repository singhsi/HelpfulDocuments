# Java Performance Tuning & Memory Optimization Guide

A practical, beginner-to-senior guide to understanding Java memory, tuning the JVM, diagnosing performance bottlenecks, and configuring Garbage Collection for high-throughput and low-latency enterprise applications.

> **Related Guides:**
> - [README.md](README.md) — Pure Java SE fundamentals, CLI tools, and classpath management
> - [commands.md](commands.md) — Java CLI syntax and command-line reference

---

## Table of Contents

- [1. Java Memory Architecture in Plain English](#1-java-memory-architecture-in-plain-english)
  - [Heap Memory (Young vs Old Generation)](#heap-memory-young-vs-old-generation)
  - [Non-Heap Memory (Metaspace, Code Cache, Thread Stacks)](#non-heap-memory-metaspace-code-cache-thread-stacks)
  - [Memory Layout Diagram](#memory-layout-diagram)
- [2. Garbage Collectors Explained](#2-garbage-collectors-explained)
  - [Serial GC](#serial-gc)
  - [Parallel GC (Throughput Collector)](#parallel-gc-throughput-collector)
  - [G1 GC (Garbage-First — The Enterprise Standard)](#g1-gc-garbage-first--the-enterprise-standard)
  - [ZGC & Shenandoah (Ultra-Low Latency)](#zgc--shenandoah-ultra-low-latency)
  - [Which Garbage Collector Should You Choose?](#which-garbage-collector-should-you-choose)
- [3. Essential JVM Tuning Flags (Production Cheat Sheet)](#3-essential-jvm-tuning-flags-production-cheat-sheet)
  - [Heap Sizing (`-Xms`, `-Xmx`)](#heap-sizing--xms--xmx)
  - [Metaspace & Stack Sizing (`-XX:MaxMetaspaceSize`, `-Xss`)](#metaspace--stack-sizing--xxmaxmetaspacesize--xss)
  - [Container-Aware Memory Flags (`-XX:MaxRAMPercentage`)](#container-aware-memory-flags--xxmaxrampercentage)
  - [Crash Diagnostics (Heap Dumps on OOM)](#crash-diagnostics-heap-dumps-on-oom)
- [4. Production JVM Presets by Workload](#4-production-jvm-presets-by-workload)
  - [Preset A: Microservices / REST APIs (1GB – 4GB RAM)](#preset-a-microservices--rest-apis-1gb--4gb-ram)
  - [Preset B: Large Enterprise Monolith (8GB – 32GB RAM)](#preset-b-large-enterprise-monolith-8gb--32gb-ram)
  - [Preset C: High-Throughput Batch Job](#preset-c-high-throughput-batch-job)
  - [Preset D: Ultra-Low Latency Trading / Payment Engine](#preset-d-ultra-low-latency-trading--payment-engine)
- [5. Diagnosing & Fixing Performance Bottlenecks](#5-diagnosing--fixing-performance-bottlenecks)
  - [Memory Leaks (`OutOfMemoryError: Java heap space`)](#1-memory-leaks-outofmemoryerror-java-heap-space)
  - [Metaspace Leaks (`OutOfMemoryError: Metaspace`)](#2-metaspace-leaks-outofmemoryerror-metaspace)
  - [Thread Starvation / CPU Spikes](#3-thread-starvation--cpu-spikes)
  - [High Garbage Collection Pauses (GC Stalls)](#4-high-garbage-collection-pauses-gc-stalls)
- [6. Performance Diagnostic CLI Tools](#6-performance-diagnostic-cli-tools)
- [7. Code-Level Performance Best Practices](#7-code-level-performance-best-practices)

---

## 1. Java Memory Architecture in Plain English

When a Java program runs, the operating system allocates a block of memory to the Java process. The JVM divides this memory into distinct areas:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                               Total JVM Process Memory                                 │
├──────────────────────────────────────────────────────────┬─────────────────────────────┤
│                       HEAP MEMORY                        │       NON-HEAP MEMORY       │
│                   (-Xms / -Xmx)                          │                             │
│ ┌─────────────────────────────┬────────────────────────┐ │ ┌─────────────────────────┐ │
│ │       Young Generation      │     Old Generation     │ │ │ Metaspace               │ │
│ │  ┌─────────┬───────┬──────┐ │                        │ │ │ (Loaded Classes, Types) │ │
│ │  │  Eden   │ S0    │ S1   │ │  Long-lived objects    │ │ ├─────────────────────────┤ │
│ │  │ (Brand  │ (Sur- │ (Sur-│ │  (Caches, Singletons,  │ │ │ Code Cache (JIT Code)   │ │
│ │  │   New)  │ vivor)│ vivor│ │   Session state)       │ │ ├─────────────────────────┤ │
│ │  └─────────┴───────┴──────┘ │                        │ │ │ Thread Stacks (-Xss)    │ │
│ └─────────────────────────────┴────────────────────────┘ │ └─────────────────────────┘ │
└──────────────────────────────────────────────────────────┴─────────────────────────────┘
```

### Heap Memory (Young vs Old Generation)

The **Heap** is where all Java object instances created with `new` reside. It is managed automatically by the Garbage Collector.

1. **Young Generation:**
   - **Eden Space:** Where brand-new objects are created. Most objects (e.g. temporary strings, loop variables) die here almost immediately.
   - **Survivor Spaces (`S0` and `S1`):** Objects that survive an Eden cleanup (Minor GC) are moved between Survivor spaces. Each survival increments their age.
2. **Old (Tenured) Generation:**
   - Objects that survive multiple Minor GC cycles (reached the *tenuring threshold*) are promoted to the Old Generation.
   - Holds long-lived objects like Spring beans, database connection pools, in-memory caches, and HTTP session data.

---

### Non-Heap Memory (Metaspace, Code Cache, Thread Stacks)

Memory allocated outside the Java object heap:

1. **Metaspace (Class Metadata):**
   - Replaced `PermGen` in Java 8.
   - Stores class definitions, method bytecode descriptors, runtime constant pools, and annotations loaded by ClassLoaders.
   - Located in native OS memory (not heap). Without bounds, it can grow until host RAM is exhausted.
2. **Code Cache:**
   - Native machine code generated by the JIT (Just-In-Time) compiler for frequently executed methods.
3. **Thread Stacks (`-Xss`):**
   - Each thread in Java gets its own stack for executing methods and storing local primitive variables and method call frames.
   - Default is typically `1MB` per thread. 1,000 threads = ~1GB of native RAM just for thread stacks.

---

## 2. Garbage Collectors Explained

The Garbage Collector (GC) automatically finds unreferenced objects and frees their memory so the application doesn't run out of RAM.

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              Garbage Collector Evolution                               │
├─────────────────┬────────────────────┬────────────────────┬────────────────────────────┤
│   Parallel GC   │       G1 GC        │       ZGC          │        Shenandoah          │
│   (Throughput)  │(Balanced / Default)│(Ultra-Low Latency) │   (Ultra-Low Latency)      │
│  ─────────────  │  ────────────────  │  ────────────────  │   ───────────────────      │
│ • Batch jobs    │ • Web servers      │ • Monoliths >16GB  │ • Microservices & trading  │
│ • Long pauses   │ • Pauses <200ms    │ • Pauses <1ms      │ • Pauses <10ms             │
│ • Max CPU work  │ • Region-based     │ • TeraByte heaps   │ • Concurrent compaction    │
└─────────────────┴────────────────────┴────────────────────┴────────────────────────────┘
```

### Serial GC (`-XX:+UseSerialGC`)
- Single-threaded collector. Freezes the entire application during garbage collection.
- **Best for:** Single-CPU environments, small embedded devices, or CLI scripts under 100MB.

### Parallel GC (`-XX:+UseParallelGC`)
- Uses multiple CPU worker threads to clean the heap. Maximizes application throughput by doing work in fast, concentrated bursts.
- **Trade-off:** Application threads stop completely during full collections (Stop-The-World pauses).
- **Best for:** Background batch jobs, scientific data processing, non-interactive reporting where a 2-second pause is acceptable.

### G1 GC (`-XX:+UseG1GC` — Default in Java 9+)
- Divides the heap into hundreds of small equal-sized **regions** (1MB to 32MB).
- Cleans regions with the highest amount of garbage first ("Garbage-First").
- Allows you to set a target pause time: `-XX:MaxGCPauseMillis=200`.
- **Best for:** 90% of enterprise web applications, REST APIs, and microservices with heap sizes between 2GB and 32GB.

### ZGC (`-XX:+UseZGC`) & Shenandoah (`-XX:+UseShenandoahGC`)
- Modern **ultra-low-latency** collectors available in Java 11/17/21+.
- Performs object evacuation and memory compaction **concurrently** while your application threads continue to run.
- Guarantees pause times under **1 millisecond** (ZGC) regardless of whether your heap is 4GB or 16 Terabytes.
- **Best for:** Real-time financial trading, high-frequency payment gateways, gaming servers, and massive heaps (>32GB).

---

### Which Garbage Collector Should You Choose?

| Scenario / Workload | Recommended GC | Flag |
|---|---|---|
| Standard Web Service / Spring Boot / REST API | **G1 GC** (Default) | `-XX:+UseG1GC` |
| Massive Heap (>16GB) or Pause-Sensitive System | **ZGC** | `-XX:+UseZGC` |
| Offline Nightly Batch Job (Max Throughput) | **Parallel GC** | `-XX:+UseParallelGC` |
| Low-Memory Container (<256MB) | **Serial GC** | `-XX:+UseSerialGC` |

---

## 3. Essential JVM Tuning Flags (Production Cheat Sheet)

### Heap Sizing (`-Xms`, `-Xmx`)

```bash
-Xms4096m -Xmx4096m
```
- `-Xms`: Initial heap size at startup.
- `-Xmx`: Maximum heap size limit.
- **Production Best Practice:** Set `-Xms` **equal** to `-Xmx`. If initial and maximum heap differ, the JVM wastes CPU constantly expanding and shrinking the heap at runtime, causing latency spikes.

---

### Metaspace & Stack Sizing (`-XX:MaxMetaspaceSize`, `-Xss`)

```bash
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m -Xss1024k
```
- `-XX:MetaspaceSize=256m`: Initial threshold before triggering first classloader GC.
- `-XX:MaxMetaspaceSize=512m`: Prevents native memory leaks from runaway classloaders (e.g. infinite CGLIB / dynamic proxy generation).
- `-Xss1024k`: Stack size per thread (1024KB = 1MB). Lowering to `-Xss512k` allows hosting more threads with less native RAM if your code does not have deeply nested recursion.

---

### Container-Aware Memory Flags (`-XX:MaxRAMPercentage`)

When running inside Docker, Podman, or Kubernetes, hardcoding `-Xmx4g` can be dangerous if the container limit changes. Use percentage-based auto-sizing:

```bash
-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0
```
- Tells the JVM to dynamically read the container memory limit (cgroups) and assign **75% of total container RAM** to the Java Heap, leaving 25% for Metaspace, threads, and OS buffers.

---

### Crash Diagnostics (Heap Dumps on OOM)

Always configure your JVM to capture a snapshot when it runs out of memory:

```bash
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/app/oom-dump.hprof -XX:+ExitOnOutOfMemoryError
```
- `-XX:+HeapDumpOnOutOfMemoryError`: Automatically writes the entire memory state to disk the moment an `OutOfMemoryError` occurs.
- `-XX:HeapDumpPath`: Directory / filename for the `.hprof` file.
- `-XX:+ExitOnOutOfMemoryError`: Immediately terminates the dead process so Kubernetes / systemd can restart a healthy instance.

---

## 4. Production JVM Presets by Workload

### Preset A: Microservices / REST APIs (1GB – 4GB RAM)

For standard Spring Boot, Quarkus, or Micronaut applications:

```bash
java -Xms2048m -Xmx2048m \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=150 \
     -XX:MetaspaceSize=128m \
     -XX:MaxMetaspaceSize=256m \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/app/oom.hprof \
     -Djava.awt.headless=true \
     -Dfile.encoding=UTF-8 \
     -jar app.jar
```

---

### Preset B: Large Enterprise Monolith (8GB – 32GB RAM)

For large monolithic enterprise applications (WildFly, JBoss, Tomcat, or Spring monoliths) with deep call stacks:

```bash
java -Xms16384m -Xmx16384m \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -XX:G1ReservePercent=15 \
     -XX:MetaspaceSize=512m \
     -XX:MaxMetaspaceSize=1024m \
     -Xss1024k \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/app/oom-dump.hprof \
     -XX:+ExitOnOutOfMemoryError \
     -Djava.awt.headless=true \
     -Dfile.encoding=UTF-8 \
     -Djava.net.preferIPv4Stack=true \
     -jar monolith.jar
```

---

### Preset C: High-Throughput Batch Job

For non-interactive data migration, ETL processing, or report generation:

```bash
java -Xms8192m -Xmx8192m \
     -XX:+UseParallelGC \
     -XX:MetaspaceSize=256m \
     -XX:MaxMetaspaceSize=512m \
     -Dfile.encoding=UTF-8 \
     -jar batch-processor.jar
```

---

### Preset D: Ultra-Low Latency Trading / Payment Engine

For real-time systems requiring sub-millisecond response times:

```bash
java -Xms12288m -Xmx12288m \
     -XX:+UseZGC \
     -XX:+UnlockExperimentalVMOptions \
     -XX:MetaspaceSize=256m \
     -XX:MaxMetaspaceSize=512m \
     -XX:+AlwaysPreTouch \
     -Dfile.encoding=UTF-8 \
     -jar trading-engine.jar
```
*(Note: `-XX:+AlwaysPreTouch` pre-allocates and zeroes out physical RAM pages at startup so requests don't suffer page-fault latency during runtime).*

---

## 5. Diagnosing & Fixing Performance Bottlenecks

### 1. Memory Leaks (`OutOfMemoryError: Java heap space`)
- **Symptoms:** Heap usage grows steadily in a sawtooth graph without ever dropping after Full GC; application becomes unresponsive.
- **Common Causes:** Static collections holding object references, unclosed streams/readers, un-invalidated `ThreadLocal` variables in thread pools.
- **Diagnostic Steps:**
  1. Open the `.hprof` heap dump using **Eclipse MAT (Memory Analyzer Tool)** or **VisualVM**.
  2. Run the **"Leak Suspects Report"** to identify the object retaining the most memory (Dominator Tree).

---

### 2. Metaspace Leaks (`OutOfMemoryError: Metaspace`)
- **Symptoms:** Native process memory grows until the JVM crashes, even though the Heap is only 20% full.
- **Common Causes:** Dynamic proxy generation (Spring/CGLIB/Hibernate) repeatedly defining new anonymous classes without unloading classloaders.
- **Fix:** Set `-XX:MaxMetaspaceSize=512m` and inspect ClassLoader counts using `jcmd <PID> GC.class_histogram`.

---

### 3. Thread Starvation / CPU Spikes
- **Symptoms:** CPU utilization hits 100%, requests time out, HTTP thread pool is exhausted.
- **Diagnostic Steps:**
  1. Capture 3 consecutive thread dumps spaced 5 seconds apart:
     ```bash
     jcmd <PID> Thread.print > thread_dump_1.txt
     sleep 5
     jcmd <PID> Thread.print > thread_dump_2.txt
     ```
  2. Search for threads in `BLOCKED` or `WAITING` state waiting on monitors (deadlocks or synchronized locks).
  3. Identify threads stuck in infinite loops executing regex or deep recursion.

---

### 4. High Garbage Collection Pauses (GC Stalls)
- **Symptoms:** Application freezes for 2–5 seconds intermittently.
- **Diagnostic Steps:**
  Enable unified GC logging in Java 9+:
  ```bash
  -Xlog:gc*,gc+phases=debug:file=/var/log/app/gc.log:time,uptime,pid:filecount=5,filesize=50m
  ```
  Upload `gc.log` to **GCPlot** or **GCEasy.io** to check GC throughput, allocation rate, and pause duration.

---

## 6. Performance Diagnostic CLI Tools

The JDK includes built-in diagnostic tools that require zero third-party agent installation:

```bash
# 1. View all running Java processes and their PIDs
jcmd -l

# 2. Monitor real-time Garbage Collection memory every 1000ms
jstat -gcutil <PID> 1000

# 3. Print live object count and memory consumption on the heap
jcmd <PID> GC.class_histogram | head -n 30

# 4. Generate a live on-demand Heap Dump for analysis in VisualVM
jcmd <PID> GC.heap_dump /tmp/live-dump.hprof

# 5. Capture thread dump to find deadlocks and stuck threads
jcmd <PID> Thread.print > /tmp/threads.txt

# 6. View JVM startup arguments and system properties
jcmd <PID> VM.command_line
jcmd <PID> VM.system_properties
```

---

## 7. Code-Level Performance Best Practices

1. **Avoid Unnecessary Object Allocations in Loops:**
   - Pre-size collections: `new ArrayList<>(1000)` or `new HashMap<>(1024)` avoids expensive array copy operations during dynamic growth.
2. **Use `StringBuilder` for String Concatenation:**
   - Avoid `String s = a + b + c` inside tight loops (creates dozens of intermediate `String` and `StringBuilder` objects).
3. **Always Clean Up `ThreadLocal`:**
   - Application servers reuse threads from a pool. If you don't call `threadLocal.remove()`, user context data leaks into subsequent requests and prevents Garbage Collection.
4. **Use Primitives over Boxed Types in Hot Loops:**
   - Use `long` instead of `Long`, `int` instead of `Integer` in mathematical operations to avoid autoboxing object allocation overhead.
5. **Stream API vs Traditional For Loops:**
   - In ultra-hot performance loops executing millions of iterations per second, simple `for` loops outperform Streams and Lambdas by avoiding iterator/lambda allocation overhead.
6. **Pool Expensive Resources:**
   - Never create database connections or HTTP clients per request; reuse connection pools (e.g. HikariCP, Apache HttpClient).
