# P95, P99 Latency and Why Average is Misleading

## Percentiles
- **P50 (Median)**: Middle value, half requests faster
- **P95**: 95% of requests faster than this value
- **P99**: 99% of requests faster than this value

## Why Average is Misleading
- **Outliers skew average**: A few slow requests can significantly impact average
- **Hides tail latency**: Doesn't show worst-case performance
- **Not representative**: Most requests may be fast, but some are very slow

## Example
```
Request latencies (ms): 10, 12, 15, 18, 20, 25, 30, 100, 500, 1000
Average: 173ms
P95: 500ms
P99: 1000ms

Most requests are fast (~20ms), but average suggests moderate performance
```

## Interview Tip
Explain that percentiles provide a better picture of user experience. P99 is especially important for identifying tail latency issues.