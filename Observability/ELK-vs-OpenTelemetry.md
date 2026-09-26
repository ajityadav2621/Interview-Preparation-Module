# ELK vs OpenTelemetry

## ELK Stack
- **Components**: Elasticsearch, Logstash, Kibana
- **Focus**: Log aggregation and analysis
- **Data**: Logs primarily
- **Use Case**: Log search, debugging, audit trails

## OpenTelemetry
- **Components**: API, SDK, Collector
- **Focus**: All observability signals (metrics, traces, logs)
- **Data**: Metrics, traces, logs
- **Use Case**: Comprehensive observability

## Comparison
| Aspect | ELK Stack | OpenTelemetry |
|--------|-----------|---------------|
| Primary Use | Log analysis | Full observability |
| Data Types | Logs | Metrics, traces, logs |
| Collection | Filebeat/Logstash | OTel Collector |
| Query | Kibana | Multiple backends |

## Interview Tip
Explain that ELK is great for logs, while OpenTelemetry provides a complete observability solution. They can be used together for comprehensive monitoring.