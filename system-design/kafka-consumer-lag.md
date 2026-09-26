# Kafka Consumer Lag Keeps Increasing. What Could Be the Reason?

## What is Consumer Lag?

**Consumer lag** is the difference between the latest message produced to a Kafka topic and the last message consumed by a consumer group. When lag increases, it means consumers are falling behind producers.

```
Topic: orders
Partition 0: [M1][M2][M3][M4][M5][M6][M7][M8][M9][M10]
             ↑                                    ↑
           Consumer position              Latest produced
           (offset 3)                     (offset 10)
           
Lag = 10 - 3 = 7 messages behind
```

## Common Causes

### 1. Consumer Processing Too Slow

The consumer can't process messages as fast as they're being produced.

```java
@KafkaListener(topics = "orders")
public void processOrder(OrderEvent event) {
    // This takes 5 seconds per message
    // But messages arrive every 100ms
    // Lag will grow indefinitely
    Thread.sleep(5000);  // Simulating slow processing
    process(event);
}
```

**Solutions**:
- Optimize processing logic
- Increase parallelism (more consumer instances)
- Move heavy processing to async/background

### 2. Consumer Not Running

The consumer is down, crashed, or not consuming.

```bash
# Check if consumer is running
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group order-consumer

# Output showing no active members:
# GROUP           TOPIC  PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-consumer  orders  0         100             500             400
# (no active members)
```

**Solutions**:
- Restart the consumer
- Check for crashes or OOM errors
- Verify consumer group configuration

### 3. Consumer Rebalancing

Consumers are frequently rebalancing, causing processing pauses.

```
Timeline:
1. Consumer A processes messages
2. Consumer B starts → rebalance
3. Consumer A stops processing → lag increases
4. Rebalance completes → Consumer A resumes
5. Repeat
```

**Causes**:
- New consumers joining/leaving the group
- Long GC pauses causing heartbeat timeouts
- Network issues causing missed heartbeats

**Solutions**:
- Increase `max.poll.interval.ms`
- Increase `session.timeout.ms`
- Reduce GC pause times
- Use `max.poll.records` to limit batch size

### 4. Single Consumer for Multiple Partitions

If one consumer handles too many partitions, it can't keep up.

```yaml
# Bad: One consumer handling 100 partitions
spring:
  kafka:
    consumer:
      group-id: order-consumer
      # Only 1 instance, but 100 partitions
```

**Solution**: Scale consumers to match partition count.

### 5. Database or External Service Bottleneck

The consumer is waiting for a slow database or external service.

```java
@KafkaListener(topics = "orders")
public void processOrder(OrderEvent event) {
    // Database is slow — 2 seconds per query
    Order order = orderRepository.findById(event.getOrderId());
    // External API is slow — 3 seconds per call
    externalService.notify(order);
    // Total: 5 seconds per message
}
```

**Solutions**:
- Optimize database queries (indexes, connection pool)
- Add caching
- Use async processing for external calls
- Increase database capacity

### 6. Consumer Configuration Issues

```yaml
# Bad configuration causing lag
spring:
  kafka:
    consumer:
      max-poll-records: 500  # Too many records per poll
      max-poll-interval-ms: 300000  # 5 minutes — too long
      enable-auto-commit: true  # May cause reprocessing
      fetch-min-bytes: 1  # Fetches too frequently
```

**Better configuration**:
```yaml
spring:
  kafka:
    consumer:
      max-poll-records: 100  # Smaller batches
      max-poll-interval-ms: 300000  # 5 minutes
      enable-auto-commit: false  # Manual commit for reliability
      fetch-min-bytes: 1024  # Wait for more data before fetching
      fetch-max-wait-ms: 500  # But not too long
```

### 7. Network Issues

Network latency or bandwidth limitations between the consumer and Kafka brokers.

```bash
# Check network latency
ping kafka-broker

# Check bandwidth
iperf3 -c kafka-broker
```

### 8. Topic Configuration

```bash
# Check topic configuration
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name orders --describe

# Check partition count
kafka-topics --bootstrap-server localhost:9092 \
  --describe --topic orders
```

**Issues**:
- Too few partitions (limits parallelism)
- Retention policy too short (messages deleted before consumed)
- Replication factor too low (availability issues)

## Monitoring Consumer Lag

### 1. Kafka Built-in Tools

```bash
# Check lag for a consumer group
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group order-consumer

# Output:
# GROUP           TOPIC  PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-consumer  orders  0         1000            1500            500
# order-consumer  orders  1         800             1200            400
```

### 2. Prometheus + Grafana

```yaml
# Kafka Exporter configuration
kafka_exporter:
  web:
    listen-address: ":9308"
  kafka:
    server: "localhost:9092"
```

### 3. Programmatic Monitoring

```java
@Component
public class ConsumerLagMonitor {
    
    @Autowired
    private KafkaAdminClient adminClient;
    
    @Scheduled(fixedRate = 30000)
    public void checkLag() {
        Collection<String> topics = adminClient.listTopics().names();
        
        for (String topic : topics) {
            TopicDescription description = adminClient
                .describeTopics(Collections.singletonList(topic))
                .values().get(topic);
            
            for (TopicPartitionInfo partition : description.partitions()) {
                // Get end offset
                long endOffset = adminClient
                    .listOffsets(Collections.singletonMap(
                        new TopicPartition(topic, partition.partition()),
                        OffsetSpec.latest()))
                    .values().get(new TopicPartition(topic, partition.partition()));
                
                // Get committed offset
                long committedOffset = adminClient
                    .listConsumerGroupOffsets("order-consumer")
                    .values()
                    .get(new TopicPartition(topic, partition.partition()))
                    .offset();
                
                long lag = endOffset - committedOffset;
                if (lag > 1000) {
                    log.warn("High lag on {}-{}: {}", topic, partition.partition(), lag);
                }
            }
        }
    }
}
```

## Solutions Summary

| Cause | Solution |
|-------|----------|
| Slow processing | Optimize code, increase parallelism |
| Consumer down | Restart, fix crashes |
| Rebalancing | Tune timeouts, reduce GC pauses |
| Too few consumers | Scale consumers to match partitions |
| DB/external bottleneck | Optimize queries, add caching |
| Bad configuration | Tune `max.poll.records`, `max.poll.interval.ms` |
| Network issues | Check bandwidth, latency |
| Topic config | Increase partitions, adjust retention |

## Key Takeaway

> Consumer lag increases when consumers can't keep up with producers. Common causes include slow processing, consumer crashes, rebalancing, insufficient parallelism, and external service bottlenecks. Monitor lag with `kafka-consumer-groups`, tune consumer configuration, scale consumers to match partition count, and optimize processing logic. The key is to identify whether the bottleneck is in the consumer itself or in its dependencies.

---

## Production Topic: Event-Driven Design with Kafka

### Topics Are Not Queues — This Distinction Matters

```java
// Queue: Point-to-point, one consumer gets each message
// Topic: Publish-subscribe, multiple consumer groups can read same message

// WRONG mental model: "Kafka is just a queue"
// RIGHT mental model: "Kafka is a distributed commit log"

// One topic can have multiple consumer groups:
// - Group A: Order processing
// - Group B: Analytics
// - Group C: Audit logging
// All three groups receive ALL messages independently
```

### Partition Key Design Determines Your Throughput Ceiling

```java
// BAD: No partition key — all messages go to one partition
kafkaTemplate.send("orders", order);  // All orders in partition 0

// GOOD: Partition by customer ID — distributes across partitions
kafkaTemplate.send("orders", customerId, order);
// Orders for different customers go to different partitions
// Enables parallel processing

// Partition count = maximum parallelism
// If you have 10 partitions, you can have at most 10 consumers
// in a single consumer group processing in parallel
```

**Key Rule**: Choose partition key based on how you want to parallelize:
- **Customer ID**: Parallelize by customer
- **Region**: Parallelize by geographic region
- **Order ID**: Parallelize by order (if you need ordering per order)

### Ordering Guarantees

```java
// Kafka guarantees ordering WITHIN a partition, not ACROSS partitions

// If all orders for customer A go to partition 0:
// - Order 1, Order 2, Order 3 → processed in order ✅

// If orders for customer A go to different partitions:
// - Partition 0: Order 1, Order 3
// - Partition 1: Order 2
// - Processing order: Order 1, Order 2, Order 3 (if lucky) or Order 2, Order 1, Order 3 ❌
```

### No Idempotent Consumers → Duplicate Processing → Duplicate Charges

```java
// BAD: Not idempotent — duplicate messages cause duplicate charges
@KafkaListener(topics = "payments")
public void processPayment(PaymentEvent event) {
    // If message is delivered twice (network retry, consumer restart)
    // This will charge the customer twice!
    paymentService.charge(event.getOrderId(), event.getAmount());
}

// GOOD: Idempotent consumer — safe to process duplicates
@KafkaListener(topics = "payments")
public void processPayment(PaymentEvent event) {
    // Check if already processed
    if (processedRepository.existsById(event.getMessageId())) {
        return;  // Already processed, skip
    }
    
    // Process payment
    paymentService.charge(event.getOrderId(), event.getAmount());
    
    // Mark as processed
    processedRepository.save(new ProcessedMessage(event.getMessageId()));
}
```

**Idempotency Strategies**:
1. **Message ID deduplication**: Store processed message IDs
2. **Database unique constraints**: `UNIQUE(order_id, status)`
3. **Idempotency keys**: Client-generated keys for exactly-once semantics
4. **Transactional outbox**: Write event and state in same transaction

### Consumer Group Design

```java
// Each consumer group has its own offset tracking
// Multiple instances in same group = load balancing
// Multiple groups = independent processing

// Example: Order topic with 10 partitions
// Group A (order processors): 10 consumers → each gets 1 partition
// Group B (analytics): 5 consumers → each gets 2 partitions
// Group C (audit): 2 consumers → each gets 5 partitions

// If Group A has only 5 consumers:
// - Consumer 1: partitions 0, 1
// - Consumer 2: partitions 2, 3
// - Consumer 3: partitions 4, 5
// - Consumer 4: partitions 6, 7
// - Consumer 5: partitions 8, 9
// Each consumer processes 2 partitions sequentially
```

### Key Takeaways

- **Topics are not queues** — they're distributed commit logs with publish-subscribe semantics
- **Partition key design determines throughput ceiling** — choose keys that enable parallel processing
- **No idempotent consumers → duplicate processing → duplicate charges** — always make consumers idempotent
- **Ordering is per-partition, not per-topic** — design partition strategy around ordering needs
- **Consumer group count = independent processing pipelines** — each group gets all messages
