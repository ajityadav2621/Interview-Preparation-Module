# Centralized Logging

## Architecture
```
Application → Log Aggregator → Storage → Query Interface
```

## Components
- **Log Collection**: Filebeat, Fluentd, Logstash
- **Storage**: Elasticsearch
- **Search**: Kibana
- **Processing**: Logstash pipelines

## Implementation
```yaml
# docker-compose.yml for ELK
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.0.0
    environment:
      - discovery.type=single-node

  logstash:
    image: docker.elastic.co/logstash/logstash:8.0.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf

  kibana:
    image: docker.elastic.co/kibana/kibana:8.0.0
    ports:
      - "5601:5601"
```

## Benefits
- Centralized log management
- Fast search across all services
- Retention policies
- Alerting on log patterns

## Interview Tip
Explain that centralized logging is essential for microservices where logs are distributed across multiple containers and services.