# Detecting High CPU Usage

## JVM Monitoring
```bash
# Find high CPU process
top -p <pid>

# Find high CPU thread
top -H -p <pid>

# Get thread stack trace
jstack <pid> | grep -A 30 <hex_thread_id>
```

## Profiling
```bash
# Java Flight Recorder
jfr start --filename recording.jfr
jfr stop

# Async Profiler
./profiler.sh -d 30 -f profile.html <pid>
```

## Application Metrics
```java
private final Gauge cpuUsageGauge;

public CpuMetrics(MeterRegistry registry) {
    this.cpuUsageGauge = Gauge.builder("system.cpu.usage")
        .register(registry);
}
```

## Interview Tip
Explain that CPU investigation requires both system-level and application-level analysis. Start with top and jstack, then use profiling tools for deeper analysis.