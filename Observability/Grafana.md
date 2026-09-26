# Grafana

## What is Grafana?
An open-source platform for visualizing and analyzing metrics, logs, and traces.

## Key Features
- **Dashboards**: Custom visualizations
- **Data Sources**: Multiple backends (Prometheus, Elasticsearch, etc.)
- **Alerting**: Threshold-based notifications
- **Plugins**: Extensible with community plugins

## Integration
```yaml
# Connect Grafana to Prometheus
data_sources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    access: proxy
```

## Dashboard Example
```json
{
  "title": "API Performance",
  "panels": [
    {
      "title": "Response Time",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))"
        }
      ]
    }
  ]
}
```

## Interview Tip
Explain that Grafana is for visualization and alerting, while Prometheus is for metrics collection. They work together but serve different purposes.