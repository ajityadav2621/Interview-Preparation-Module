# Readiness vs Liveness Probes

## Liveness Probe
- Determines if container needs restart
- If fails, container is restarted
- Use when application is unresponsive

### Configuration
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

## Readiness Probe
- Determines if container is ready to serve traffic
- If fails, endpoint is removed from service
- Use when application needs time to initialize

### Configuration
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  successThreshold: 1
  failureThreshold: 3
```

## Benefits
- **Liveness**: Restores unhealthy containers
- **Readiness**: Prevents traffic to unready pods
- **Zero downtime deployments**: Gradual traffic shift
- **Rollback support**: Automatic traffic removal

## Interview Tip
Explain that liveness keeps containers alive, while readiness controls traffic routing. Both are essential for reliable deployments.