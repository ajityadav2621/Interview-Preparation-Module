# Prometheus Metrics Collection

## Pull Model
- **Definition**: Prometheus server actively scrapes metrics from targets
- **Targets**: Configured endpoints exposing metrics
- **Interval**: Regular scraping (e.g., every 15 seconds)
- **Format**: Prometheus text format

## Configuration
```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
  metrics:
    export:
      prometheus:
        enabled: true

# prometheus.yml
scrape_configs:
  - job_name: 'application'
    static_configs:
      - targets: ['localhost:8080']
    scrape_interval: 15s
```

## Metrics Endpoint
```
GET /actuator/prometheus
# HELP http_server_requests_seconds
# TYPE http_server_requests_seconds histogram
http_server_requests_seconds_bucket{exception="none",status="200",uri="/api/orders",le="0.25"} 1234
```

## Interview Tip
Explain that Prometheus uses a pull model, which is different from push-based systems. This allows for better control and scalability.