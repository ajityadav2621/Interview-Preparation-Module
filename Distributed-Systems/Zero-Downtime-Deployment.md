# Zero-Downtime Deployment

## Strategies

### 1. **Blue-Green Deployment**
```
Current: Blue (v1) ← Production
          |
          ↓
New:      Green (v2) ← Staging
          |
          ↓ (after validation)
Switch:   Green (v2) ← Production
```

### 2. **Canary Deployment**
```
100% → v1
95% → v1, 5% → v2
90% → v1, 10% → v2
0% → v1, 100% → v2
```

### 3. **Rolling Deployment**
```
Stop Instance 1 → Update → Start
Stop Instance 2 → Update → Start
Stop Instance 3 → Update → Start
```

## Implementation

### 1. **Blue-Green with Kubernetes**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: app:v1

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 0
  template:
    spec:
      containers:
      - name: app
        image: app:v2

# Switch service to point to green deployment
kubectl patch service app-service -p '{"spec":{"selector":{"app":"app-green"}}}'
```

### 2. **Canary with Istio**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app
spec:
  http:
  - route:
    - destination:
        host: app
      weight: 90
    - destination:
        host: app-canary
      weight: 10
```

## Interview Tip
Explain that zero-downtime deployments require careful planning. Blue-green provides instant rollback, while canary provides gradual validation.