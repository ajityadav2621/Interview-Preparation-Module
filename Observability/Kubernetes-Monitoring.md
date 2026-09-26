# Monitoring Spring Boot in Kubernetes

## Kubernetes Monitoring
```yaml
# HPA configuration
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## Application Metrics
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

## Alerting
```yaml
groups:
- name: kubernetes-alerts
  rules:
  - alert: HighPodCpu
    expr: rate(container_cpu_usage_seconds_total[5m]) > 0.8
  - alert: HighMemoryUsage
    expr: container_memory_working_set_bytes > 0.9 * limit
```

## Interview Tip
Explain that monitoring in Kubernetes requires both application-level metrics and infrastructure-level metrics. Use HPA for auto-scaling and Prometheus for alerting.