# Module 4: Kafka & Event-Driven Architecture

---

## PART 1: Fundamentals (must know cold)

- **Topic**: a named, append-only log that producers write to and consumers read from.
- **Partition**: a topic is split into partitions for parallelism/scale — Kafka only guarantees message *ordering within a single partition*, not across the whole topic.
- **Producer/Consumer**: producers publish messages; consumers subscribe and read, tracking their own **offset** (position) in each partition they read from.
- **Consumer group**: a set of consumers sharing the work of reading a topic — each partition is consumed by exactly one consumer within a group at a time, which is how Kafka parallelizes consumption across multiple consumer instances.
- **Delivery guarantees**: Kafka's default is **at-least-once** — a message can be redelivered (e.g., after a consumer crash before committing its offset), never silently dropped in normal operation. Exactly-once is achievable but requires specific configuration and idempotent producers/consumers.
- **Idempotent consumer**: since redelivery can happen, consumer logic must handle processing the same message twice without a bad side effect (e.g., checking an `event_id` against an already-processed table before acting).
- **Dead-letter topic/queue**: where a message goes after repeatedly failing processing, so one bad message doesn't block the whole partition.
- **Schema evolution**: producers and consumers should be able to change independently — using a schema registry (Avro/Protobuf) or versioned JSON schemas prevents a producer's change from silently breaking every consumer.

---

## PART 2: Interview Questions & Answers

**Q1: What happens if a Kafka consumer crashes mid-processing?**
> Kafka only advances a consumer's committed offset after it successfully finishes processing a message (or a batch, depending on config), so on restart the consumer resumes from the last committed offset — the message being processed at the time of the crash gets redelivered. That's why consumer logic needs to be idempotent: I'd check an event ID against a table of already-processed IDs before applying the change, so redelivery doesn't cause a duplicate side effect like double-decrementing inventory.

**Q2: Why use Kafka instead of having services call each other's REST APIs directly?**
> When multiple downstream services need to react to the same event, and the producer doesn't need an immediate response, Kafka decouples them — the producer publishes once, any number of consumers subscribe independently, and a consumer being slow or temporarily down just means it catches up later rather than blocking or breaking the producer. Direct REST calls would mean the producer has to know about, call, and wait on every consumer, which tightly couples services that should be independent.

**Q3: How do you guarantee ordering when it matters, given Kafka only orders within a partition?**
> I'd choose a partition key that groups related events together — for example, partitioning by `order_id` so every event for the same order lands in the same partition and is processed in order, while different orders can be processed in parallel across partitions. Choosing the wrong key (like a random UUID unrelated to what needs ordering) would spread related events across partitions and lose the ordering guarantee that actually matters.

**Q4: How would you handle a "poison message" that a consumer can never successfully process?**
> Without handling, that message would block the whole partition — the consumer would keep failing on it and never advance past it (or lose data if it were just skipped blindly). I'd configure a retry limit, and after exhausting retries, route the message to a dead-letter topic instead of retrying forever, so the consumer can move on to later messages while the poison message is preserved for investigation rather than silently dropped.

**Q5: How do producers and consumers stay compatible as the event schema evolves over time?**
> I'd use a schema registry (or at minimum a versioned schema convention) so a producer adding a new optional field doesn't break existing consumers still expecting the old shape, and consumers are written to tolerate unknown extra fields rather than failing on them. Breaking changes (removing/renaming a required field) get a new schema version or a new topic rather than mutating the existing one in place, since old messages already in the log still need to be readable under the old schema.

---

## PART 3: How It Works Internally

**How the log-based model actually works**: unlike a traditional message queue where a message is removed once consumed, a Kafka topic partition is an append-only, immutable log on disk. Producers append to the end; each message gets a sequential offset number. Consumers don't "take" a message — they just read from the log at whatever offset they're tracking, which is why multiple independent consumer groups can read the exact same topic at their own pace without interfering with each other (each group tracks its own offset).

**How partitioning enables parallelism**: each partition is an independent ordered log, potentially stored on a different broker (server) in the cluster. A producer's partitioner (often a hash of the message key) decides which partition a message goes to — same key always goes to the same partition, which is what gives you per-key ordering. Within a consumer group, Kafka assigns each partition to exactly one consumer instance at a time (a "rebalance" reassigns partitions if a consumer joins/leaves the group), which is the actual mechanism behind horizontal scaling of consumption — more partitions allow more parallel consumers, up to one consumer per partition.

**How offset commits provide the at-least-once guarantee**: a consumer periodically (or after each message/batch) tells the broker "I've processed up to offset X" — this is the commit. If the consumer crashes *before* committing but *after* processing, on restart it resumes from the last committed offset and reprocesses the message it had already handled — hence "at-least-once," not "exactly-once," by default. Achieving exactly-once requires either idempotent processing (the practical, commonly used approach) or Kafka's transactional producer/consumer APIs, which coordinate offset commits and produced messages atomically — more complex to set up and usually only worth it for specific financial/critical-accuracy use cases.

**How replication provides durability**: each partition is replicated across multiple brokers (one leader, N followers). Producers write to the leader; followers replicate the data. If the leader broker fails, one of the followers (that was sufficiently caught up) is elected as the new leader — this is what lets Kafka survive individual broker failures without losing committed data, as long as replication factor and acknowledgment settings (`acks=all`) are configured to wait for replication before considering a write successful.
