# Metrics vs Logs vs Traces

## Metrics
- **Nature**: Aggregated numerical data
- **Storage**: Time-series databases
- **Query**: Aggregation functions
- **Use Case**: Trending, alerting
- **Example**: p99 latency = 250ms

## Logs
- **Nature**: Timestamped text events
- **Storage**: Log aggregators
- **Query**: Full-text search
- **Use Case**: Debugging, auditing
- **Example**: "ERROR: Payment failed for order 123"

## Traces
- **Nature**: Request flow visualization
- **Storage**: Distributed tracing systems
- **Query**: Graph analysis
- **Use Case**: Performance debugging
- **Example**: Request took 2s: API Gateway (50ms) → Order Service (1500ms) → Payment (400ms)

## When to Use Each
- **Metrics**: "Is the system healthy?"
- **Logs**: "What happened at 2:30 PM?"
- **Traces**: "Why did this request take 2 seconds?"

## Interview Tip
Explain that metrics show trends, logs provide details, and traces show flows. All three are needed for comprehensive observability.