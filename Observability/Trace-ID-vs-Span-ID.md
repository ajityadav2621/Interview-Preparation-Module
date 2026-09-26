# Trace ID and Span ID

## Trace ID
- **Definition**: Unique identifier for entire request
- **Format**: Usually 16 bytes (32 hex characters)
- **Purpose**: Correlate all spans in a request
- **Propagation**: Passed through all services

## Span ID
- **Definition**: Unique identifier for individual operation
- **Format**: Usually 8 bytes (16 hex characters)
- **Purpose**: Identify specific operation within trace
- **Hierarchy**: Parent-child relationships

## Example
```
Trace ID: abc123def456ghi789jkl012mno345p
  Span 1: 1111111111111111 (API Gateway)
    Span 2: 2222222222222222 (Order Service)
      Span 3: 3333333333333333 (Payment Service)
```

## Interview Tip
Explain that trace ID correlates the entire request, while span ID identifies individual operations within that request.