# Troubleshooting High CPU in a Service

## Investigation Steps

### 1. **Identify High CPU Process**
```bash
# Find process with high CPU
top -p <pid>
htop -p <pid>
```

### 2. **Find High CPU Thread**
```bash
# Get thread CPU usage
top -H -p <pid>

# Convert thread ID to hex for jstack
printf "%x\n" <thread_id>

# Get thread stack trace
jstack <pid> | grep -A 30 <hex_thread_id>
```

### 3. **Analyze Stack Trace**
- Look for busy loops
- Check for infinite recursion
- Identify blocking operations
- Review third-party library usage

### 4. **Profiling**
```bash
# Java Flight Recorder
jfr start --filename recording.jfr
# ... reproduce issue ...
jfr stop

# VisualVM
jvisualvm --pid <pid>

# Async Profiler
./profiler.sh -d 30 -f profile.html <pid>
```

## Common Causes
- Infinite loops
- Complex computations
- String concatenation in loops
- Regex backtracking
- Reflection overhead
- GC activity

## Interview Tip
Explain that CPU investigation requires both system-level and application-level analysis. Always start with top and jstack, then use profiling tools for deeper analysis.