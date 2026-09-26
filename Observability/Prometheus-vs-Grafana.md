# Prometheus vs Grafana

## Prometheus
- **Role**: Metrics collection and storage
- **Data**: Time-series numerical data
- **Query**: PromQL
- **Alerting**: Built-in alert manager
- **Storage**: Local time-series database

## Grafana
- **Role**: Visualization and analysis
- **Data**: Multiple sources (Prometheus, Elasticsearch, etc.)
- **Query**: Visual query builder
- **Alerting**: Dashboard-based
- **Storage**: Metadata only

## Comparison
| Aspect | Prometheus | Grafana |
|--------|------------|---------|
| Purpose | Metrics collection | Visualization |
| Data Source | Scraped endpoints | Multiple backends |
| Query Language | PromQL | Visual builder |
| Alerting | Built-in | Dashboard-based |

## Interview Tip
Explain that Prometheus collects and stores metrics, while Grafana visualizes and analyzes them. They are complementary tools.