# Your Java API Suddenly Becomes Slow in Production. What JVM-Level Metrics Would You Check First?

## The Investigation Checklist

When your Java API suddenly becomes slow, you need to systematically check JVM-level metrics to identify the root cause. Here's the order in which you should check:

## 1. GC Activity (Most Common Cause)

### Check GC Frequency and Pause Times

```bash
# Enable GC logging (Java 9+)
-Xlog:gc*:gc.log:time

# Or for Java 8
-XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc.log
```

**What to look for**:
- **Frequent Full GC**: Indicates memory pressure
- **Long Full GC pauses**: Can cause multi-second request delays
- **High GC overhead**: > 50% of CPU time spent in GC

```bash
# Monitor GC with jstat
jstat -gc <pid> 1s

# Output columns:
# S0C: Survivor 0 capacity
# S1C: Survivor 1 capacity
# EC: Eden capacity
# OC: Old generation capacity
# MC: Metaspace capacity
# YGC: Young GC count
# YGCT: Young GC time
# FGC: Full GC count
# FGCT: Full GC time
# GCT: Total GC time
```

### Red Flags:
- `FGC` (Full GC count) increasing rapidly
- `FGCT` (Full GC time) > 1 second
- `GCT` (total GC time) > 10% of total uptime

## 2. Heap Memory Usage

### Check Heap Utilization

```bash
# Monitor heap with jstat
jstat -gc <pid> 1s

# Or use jmap for heap histogram
jmap -histo <pid> | head -20
```

**What to look for**:
- **Heap near max**: `OU` (Old Generation Used) close to `OC` (Old Generation Capacity)
- **Memory leaks**: Heap usage keeps growing after GC
- **High allocation rate**: Eden space filling up too quickly

### JVM Flags for Heap Monitoring

```bash
# Set heap size
-Xms4g -Xmx4g

# Enable GC logging
-Xlog:gc*:gc.log:time

# Print GC details
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
```

## 3. Thread States and Contention

### Check Thread Dump

```bash
# Take a thread dump
jstack <pid> > threaddump.txt

# Or send SIGQUIT
kill -3 <pid>
```

**What to look for**:
- **BLOCKED threads**: Threads waiting for locks
- **WAITING threads**: Threads waiting for conditions
- **Thread starvation**: Too many threads competing for few resources
- **Deadlock**: Threads waiting for each other

```bash
# Count thread states
jstack <pid> | grep -c "java.lang.Thread.State"
jstack <pid> | grep "BLOCKED" | wc -l
jstack <pid> | grep "WAITING" | wc -l
```

### Thread Count

```bash
# Check number of threads
jstack <pid> | grep "java.lang.Thread.State" | wc -l

# Or use jcmd
jcmd <pid> Thread.print | grep "java.lang.Thread.State" | wc -l
```

**Red Flags**:
- Thread count > 200 (may indicate thread leak)
- Many BLOCKED threads (lock contention)
- Threads stuck in WAITING state (potential deadlock)

## 4. CPU Usage

### Check CPU Breakdown

```bash
# Top processes
top -p <pid>

# Detailed CPU usage
top -H -p <pid>  # Shows per-thread CPU usage

# Or use jcmd
jcmd <pid> VM.native_memory summary
```

**What to look for**:
- **High CPU**: Could indicate busy loops, excessive GC, or computation
- **Low CPU + slow**: Could indicate I/O blocking, lock contention, or GC pauses

### CPU Profiling

```bash
# Use jcmd for sampling
jcmd <pid> VM.native_memory summary

# Or use external tools
# - Java Flight Recorder (JFR)
# - async-profiler
# - YourKit, JProfiler
```

## 5. JIT Compilation

### Check JIT Activity

```bash
# Enable JIT compilation logging
-XX:+PrintCompilation

# Or use jcmd
jcmd <pid> VM.class_hierarchy
```

**What to look for**:
- **Frequent compilation**: May indicate code is not being optimized
- **Deoptimization**: Code being deoptimized and recompiled

## 6. Class Loading

### Check Class Loading

```bash
# Monitor class loading
jstat -class <pid> 1s

# Or use jcmd
jcmd <pid> VM.classloader_stats
```

**What to look for**:
- **Rapid class loading**: May indicate classloader leaks
- **High class count**: May indicate memory pressure

## 7. Native Memory

### Check Native Memory

```bash
# Enable Native Memory Tracking (NMT)
-XX:NativeMemoryTracking=summary

# Check native memory
jcmd <pid> VM.native_memory summary
```

**What to look for**:
- **High native memory**: May indicate off-heap memory leaks
- **Thread stack memory**: Too many threads consuming stack space

## 8. JVM Flags and Configuration

### Check Current JVM Settings

```bash
# Print all JVM flags
jcmd <pid> VM.flags

# Print system properties
jcmd <pid> VM.system_properties

# Print JVM version
jcmd <pid> VM.version
```

## Diagnostic Commands Summary

```bash
# Quick health check
jps                           # List Java processes
jstat -gc <pid> 1s            # Monitor GC
jstack <pid>                  # Thread dump
jmap -heap <pid>              # Heap summary
jcmd <pid> VM.flags           # JVM flags
jcmd <pid> VM.native_memory summary  # Native memory
```

## What to Check First (Priority Order)

1. **GC logs** — Most common cause of sudden slowdowns
2. **Thread dumps** — Check for deadlocks and contention
3. **Heap usage** — Check for memory leaks
4. **CPU usage** — Check for busy loops or excessive GC
5. **Thread count** — Check for thread leaks
6. **Native memory** — Check for off-heap leaks

## Key Takeaway

> When your Java API suddenly becomes slow, **check GC activity first** — Full GC pauses are the most common cause of sudden latency spikes. Then check thread dumps for deadlocks and contention, followed by heap usage for memory leaks. Use `jstat`, `jstack`, and `jcmd` for quick diagnostics, and enable GC logging for detailed analysis.
