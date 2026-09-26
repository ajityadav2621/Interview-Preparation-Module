# Interview Preparation Summary

## Folder Structure Overview

### Core Java (13 files)
- String immutability
- HashMap internals
- HashCode collision handling
- equals/hashCode contract
- ArrayList vs LinkedList
- HashMap vs ConcurrentHashMap
- Collection modification during iteration
- synchronized vs volatile
- Deadlock identification
- Runnable vs Callable
- ExecutorService vs manual threads
- CompletableFuture
- HashSet duplicate detection
- Optional usage
- Streams vs loops

### JVM (6 files)
- Memory leaks despite GC
- Heap vs Stack
- Full GC impact on latency
- Exception in thread
- GC metrics to monitor
- JVM memory analysis

### Concurrency (6 files)
- Thread pools
- ConcurrentHashMap Java 8+
- False sharing
- synchronized vs ReentrantLock
- Thread contention analysis
- GC metrics

### Spring Boot (5 files)
- DI internals
- Request idempotency
- Banking authentication
- Partial failures
- API versioning

### Database (3 files)
- Index internals
- Query optimization
- Schema design

### Distributed Systems (27 files)
- Downstream unavailable
- Cascading failures
- Service slow
- Timeout values
- Retry vs Circuit Breaker
- Retry worse outage
- Bulkhead pattern
- Partial failures
- Idempotent REST
- Duplicate requests
- Duplicate messages
- Kafka consumer crash
- Poison messages
- Dead Letter Queue
- Message ordering
- Database unavailable
- Connection pool exhaustion
- Memory exhaustion
- High CPU troubleshooting
- Pod restarting
- Readiness/liveness probes
- Service discovery failure
- API Gateway down
- Multiple API Gateways
- Distributed transactions
- Saga failure
- Choreography vs Orchestration
- Eventual consistency
- Network latency
- Intermittent failures
- Request tracing
- Latency source identification
- Missing logs
- Traffic spike
- Tenant resources
- Zero-downtime deployment
- Rollback
- Backward compatibility
- Dependency failures
- Order flow investigation

### Messaging (5 files)
- Duplicate messages
- Kafka consumer crash
- Poison messages
- Dead Letter Queue
- Message ordering

### Observability (33 files)
- Observability concepts
- Monitoring vs observability
- Three pillars
- Metrics vs logs vs traces
- Micrometer
- Spring Boot + Micrometer
- OpenTelemetry
- OpenTelemetry vs Micrometer
- Distributed tracing
- Trace ID vs Span ID
- Trace context propagation
- Spring Boot + OpenTelemetry
- Prometheus collection
- Prometheus pull model
- Grafana
- Prometheus vs Grafana
- JVM metrics
- Spring Boot metrics
- API latency monitoring
- Error rate monitoring
- P95/P99 latency
- Identifying slow service
- Log-trace correlation
- Structured logging
- Centralized logging
- ELK vs OpenTelemetry
- Custom Micrometer metrics
- Kafka consumer lag
- Connection pool monitoring
- Memory leak detection
- High CPU detection
- Kubernetes monitoring

### System Design (5 files)
- Transaction processing
- Distributed caching
- Audit logging
- Payment APIs retry
- Design patterns

### Performance (1 file)
- CPU/memory investigation

### DevOps (2 files)
- Deployment strategies
- Critical applications

### Algorithms (5 files)
- Sliding window maximum
- Next greater element
- Subarray sum equals K
- Flatten binary tree
- Core Java concurrency

## Key Topics Covered

1. **Core Java Fundamentals**: String, collections, concurrency, memory management
2. **JVM Internals**: GC, memory leaks, heap/stack, performance tuning
3. **Concurrency**: Thread pools, locks, false sharing, contention
4. **Spring Boot**: DI, transactions, caching, security, API design
5. **Database**: Indexing, query optimization, transactions
6. **Distributed Systems**: Resilience patterns, service discovery, distributed transactions
7. **Messaging**: Kafka, idempotency, ordering, DLQs
8. **Observability**: Metrics, logs, traces, monitoring tools
9. **System Design**: High availability, scalability, consistency
10. **Performance**: CPU/memory investigation, profiling
11. **DevOps**: Deployment strategies, monitoring, alerting
12. **Algorithms**: Core data structures and algorithms

## Interview Preparation Tips

1. **Start with Core Java**: Master fundamentals before moving to advanced topics
2. **Practice System Design**: Design patterns and architecture are critical
3. **Understand Distributed Systems**: Real-world scenarios are common
4. **Learn Observability**: Modern systems require monitoring expertise
5. **Practice Coding**: Algorithms and data structures are essential
6. **Study Real Cases**: Production scenarios and incident response

## Recommended Study Order

1. Core Java → JVM → Concurrency
2. Spring Boot → Database
3. Distributed Systems → Messaging
4. Observability → System Design
5. Algorithms → Performance → DevOps

This structure provides comprehensive coverage of all major Java interview topics with practical examples and real-world scenarios.