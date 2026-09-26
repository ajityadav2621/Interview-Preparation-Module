# Monitoring Database Connection Pool

## Key Metrics
- **Active connections**: Currently in use
- **Idle connections**: Available for use
- **Wait time**: Time spent waiting for connections
- **Connection creation rate**: New connections per second
- **Connection usage**: Percentage of pool in use

## HikariCP Metrics
```yaml
spring.datasource.hikari.metrics.enabled=true
```

## Prometheus Queries
```promql
# Connection pool usage
hikaricp_connections_active / hikaricp_connections_max

# Average wait time
rate(hikaricp_connections_wait_seconds_sum[5m]) / rate(hikaricp_connections_wait_seconds_count[5m])
```

## Alerting
```yaml
groups:
- name: database-alerts
  rules:
  - alert: ConnectionPoolExhausted
    expr: hikaricp_connections_active == hikaricp_connections_max
    for: 5m
```

## Interview Tip
Explain that connection pool monitoring helps identify leaks and sizing issues before they cause outages.