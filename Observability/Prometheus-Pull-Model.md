# Prometheus Pull Model

## How It Works
1. Prometheus server configuration defines scrape targets
2. Prometheus periodically sends HTTP GET requests to targets
3. Targets respond with metrics in text format
4. Prometheus stores time-series data

## Advantages
- Centralized control
- Easy target discovery
- Better error handling
- Natural service discovery integration

## Disadvantages
- Requires network access to targets
- Additional configuration needed

## Interview Tip
Explain that the pull model gives Prometheus control over when and how to collect metrics, unlike push-based systems.