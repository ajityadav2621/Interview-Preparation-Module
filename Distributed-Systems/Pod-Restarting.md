# Kubernetes Pod Keeps Restarting

## Causes
- CrashLoopBackOff
- Out of memory
- Liveness probe failures
- Readiness probe failures
- Application crashes
- Dependency failures

## Investigation

### 1. **Check Pod Status**
```bash
kubectl describe pod <pod-name>

# Look for:
# - Last state (terminated, waiting)
# - Reason and message
# - Restart count
```

### 2. **Check Logs**
```bash
# View logs
kubectl logs <pod-name>

# View previous pod logs
kubectl logs <pod-name> --previous

# Follow logs
kubectl logs -f <pod-name>
```

### 3. **Check Events**
```bash
kubectl get events --watch

# Look for:
# - OOMKilled
# - CrashLoopBackOff
# - Liveness probe failures
```

## Solutions

### 1. **Fix Application Issues**
```java
// Check for:
// - Uncaught exceptions
// - Memory leaks
// - Thread deadlocks
// - Resource exhaustion
```

### 2. **Configure Probes**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 3. **Resource Limits**
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

## Interview Tip
Explain that pod restarts are usually caused by application issues. Check logs first, then investigate resource limits and probe configurations.