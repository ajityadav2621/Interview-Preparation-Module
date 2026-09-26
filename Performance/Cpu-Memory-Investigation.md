# Investigating CPU/Memory Spikes

## CPU Spike Investigation

### 1. **Identify High CPU Thread**
```bash
# Get process CPU usage
top -p <pid>

# Find high CPU threads
top -H -p <pid>

# Convert thread ID to hex for jstack
printf "%x\n" <thread_id>

# Get thread stack trace
jstack <pid> | grep -A 30 <hex_thread_id>
```

### 2. **Analyze Stack Trace**
- Look for busy loops
- Check for infinite recursion
- Identify blocking operations
- Review third-party library usage

### 3. **Profiling Tools**
```bash
# Java Flight Recorder
jfr start --filename recording.jfr
# ... reproduce issue ...
jfr stop

# VisualVM
jvisualvm --pid <pid>

# Async Profiler
async-profiler profiler.sh -d 30 -f profile.html <pid>
```

## Memory Spike Investigation

### 1. **Heap Dump Analysis**
```bash
# Generate heap dump
jmap -dump:live,format=b,file=heap.hprof <pid>

# Analyze with Eclipse MAT
mat heap.hprof

# Key metrics:
# - Dominator tree: who holds most memory
# - Leak suspects: potential memory leaks
# - Top consumers: largest objects
```

### 2. **Monitor Memory Usage**
```bash
# Monitor heap usage
jstat -gc <pid>

# Look for:
# Old generation increasing
# Full GC frequency
# Memory not being reclaimed
```

## Common Causes

### CPU Spikes
- Infinite loops
- Complex computations
- String concatenation in loops
- Regex backtracking
- Reflection overhead

### Memory Spikes
- Memory leaks (static collections, ThreadLocal)
- Unbounded caches
- Large object allocations
- Session accumulation
- String interning issues

## Interview Tip
Explain that investigation should start with monitoring tools, then drill down to specific code paths. Always reproduce the issue in a controlled environment when possible.