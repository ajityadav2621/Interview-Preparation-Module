# Module 2: Java Spring Boot & Microservice Architecture

---

## PART 1: Fundamentals (must know cold)

- **Spring Boot layered structure**: `@RestController` (handles HTTP) → `@Service` (business logic) → `@Repository` (data access, usually extending `JpaRepository`) → Entity (`@Entity`, maps to a DB table).
- **Dependency Injection (DI)**: Spring manages object creation and wiring via `@Autowired`/constructor injection — you don't manually `new` your dependencies; Spring's container does it and hands you the instance.
- **Microservice vs monolith**: a microservice owns its own database and is independently deployable; services communicate over the network (REST, gRPC, or messaging) instead of in-process function calls.
- **Sync vs async inter-service communication**: REST call = caller waits for a response (tight coupling, immediate consistency); message/event (Kafka) = fire-and-forget, consumer processes independently (loose coupling, eventual consistency).
- **Multithreading basics in Java**: `Thread`, `ExecutorService`/thread pools, `synchronized`/locks for shared mutable state, the difference between concurrency (interleaved) and parallelism (simultaneous on multiple cores).
- **Circuit breaker pattern**: stop calling a failing downstream service for a cooldown period instead of letting every caller pile up waiting on a dead dependency.
- **Statelessness for scaling**: a stateless service instance can be freely load-balanced/replicated because no instance holds session state another instance needs.

---

## PART 2: Interview Questions & Answers

**Q1: Explain Spring's dependency injection and why it matters.**
> Instead of a class creating its own dependencies with `new`, Spring's IoC (Inversion of Control) container creates and injects them — usually via constructor injection into a class annotated `@Service` or `@Component`. This matters because it decouples a class from the concrete implementation of its dependencies, making it trivial to swap implementations (e.g., a mock in tests) without changing the class itself.

**Q2: How do you handle a downstream service being down in a microservice architecture?**
> I'd wrap the call in a circuit breaker (e.g., Resilience4j) that tracks recent failure rate — after it crosses a threshold, the breaker "opens" and fails fast locally for a cooldown period instead of letting every request wait on a timeout against a dead service. I'd also define a fallback (a cached value, a degraded response, or a queued retry) so the caller doesn't just error out to the end user.

**Q3: When would you use Kafka instead of a direct REST call between two of your Spring Boot services?**
> When the caller doesn't need an immediate response, and especially when multiple services need to react to the same event. A direct REST call couples the producer to knowing about and calling every consumer and blocks until each responds; publishing to Kafka lets the producer move on immediately and any number of consumers subscribe independently, so a slow or temporarily-down consumer doesn't affect the producer.

**Q4: How would you design database ownership across microservices for a CRM/ERP/TMS system like the ones you've worked on?**
> Each service owns its own database, and no other service is allowed to query it directly — if the ERP service needs CRM data, it either calls the CRM service's API, or subscribes to CRM's published events and keeps its own local read copy. This avoids the classic "distributed monolith" trap where services are deployed separately but are still tightly coupled through a shared database schema.

**Q5: How do you use multithreading safely when multiple threads update shared state?**
> I'd minimize shared mutable state in the first place — prefer each thread working on independent data. Where shared state is unavoidable (e.g., an in-memory counter or cache), I'd use `synchronized` blocks or higher-level concurrent structures (`ConcurrentHashMap`, `AtomicInteger`) rather than manual locking, since they're built and tested specifically to avoid race conditions and are less error-prone than hand-rolled locks.

---

## PART 3: How It Works Internally

**What Spring Boot's DI container actually does at startup**: on application boot, Spring scans for classes annotated `@Component`/`@Service`/`@Repository`/`@RestController` (component scanning), builds a dependency graph of what each class needs, and instantiates them in dependency order — a class needing another `@Service` gets Spring's managed singleton instance injected into its constructor. This is why you rarely see `new SomeService()` in Spring code — Spring's container owns the object lifecycle (this is the "Inversion of Control" part: control over object creation is inverted from your code to the framework).

**What a circuit breaker does mechanically**: it wraps a call and tracks a rolling window of recent successes/failures. Three states: **Closed** (normal, calls pass through), **Open** (failure threshold exceeded, calls fail immediately without even attempting the network call), **Half-Open** (after a cooldown, it lets a limited number of test calls through to see if the downstream has recovered — if they succeed, it closes again; if they fail, it reopens). This is what prevents "everyone retries a dead service simultaneously" pile-ups.

**How Kafka decouples services internally** (deeper detail in Module 4's file): a producer publishes a message to a topic (an append-only log); consumers independently track their own offset (position) into that log, so one consumer being slow or down doesn't block another consumer or the producer — they're reading the same log at their own pace.

**Java thread pool internals (`ExecutorService`)**: rather than creating a new OS thread per task (expensive — thread creation/context-switching has real overhead), a thread pool maintains a fixed set of worker threads and a queue of pending tasks; each worker pulls the next task off the queue when it's free. This is why using a bounded thread pool (`Executors.newFixedThreadPool(n)`) for concurrent I/O-bound calls (like the third-party SDK integration on your resume) gives you controlled concurrency without the overhead or resource exhaustion risk of unbounded thread creation.
