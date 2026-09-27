# Module 9: System Design — Multi-Tenant ERP & Real-Time Analytics

---

## PART 1: Fundamentals (must know cold)

- **RBAC (Role-Based Access Control)**: users are assigned roles, roles are granted permissions — access checks happen against the role's permissions, not per-user, which scales far better than assigning permissions individually.
- **Tenant isolation**: no request, query, or cached value should ever be able to cross from one tenant's data into another's — this must be enforced server-side, never trusted from the client.
- **Server-side authorization**: any access control check (RBAC, tenant scoping) must be enforced in backend code on every request — a hidden UI button or greyed-out menu item is not a security control, since a request can always be crafted directly.
- **Stream processing**: continuously processing an unbounded flow of events (e.g., from Kafka) as they arrive, rather than in scheduled batches — used for live dashboards/monitoring.
- **Backpressure**: what happens when a consumer/downstream system is slower than the rate data is arriving — a well-designed pipeline buffers (e.g., in Kafka) rather than dropping data or crashing.
- **Consumer lag**: the gap between the latest produced event and what a consumer has actually processed — a key health metric for a real-time pipeline; growing lag means the system is falling behind.
- **Horizontal vs vertical scaling**: horizontal = add more instances/nodes (requires the system to be designed to distribute work — e.g., statelessness, partitioning); vertical = make one instance bigger (simpler but has a ceiling and a single point of failure).

---

## PART 2: Interview Questions & Answers

**Q1: How did you implement RBAC in the multi-tenant ERP?**
> After login, the auth token carries both the tenant ID and the user's role. Every request passes through middleware that extracts both from the token, and before the service layer executes an action, it checks whether the user's role has permission for that specific action. Data queries are additionally always scoped by `tenant_id` regardless of role — role controls *what* actions you can perform, tenant scoping separately controls *whose* data you can ever see, and both checks happen server-side so a directly crafted request bypassing the UI still gets the same enforcement.

**Q2: How would you design a real-time analytics platform to handle a spike in incoming event volume without losing data?**
> Kafka acts as a durable buffer between the data sources and the stream processor — since Kafka retains messages for a configured retention period regardless of consumer speed, a temporary spike just means the consumer's lag grows temporarily and it catches up once the spike passes, rather than data being dropped. I'd monitor consumer lag as a key health metric and have alerting on it, so a *sustained* growing lag (versus a brief spike) gets investigated before it becomes a real problem — that usually means scaling out the number of consumer instances (up to the number of partitions) to increase processing throughput.

**Q3: If a live dashboard shows stale data, how would you debug where the delay is coming from?**
> I'd check each stage of the pipeline in order: is the producer publishing events at the expected rate (check producer-side metrics), is the consumer's lag growing (check consumer group lag), or is the aggregation/storage step slow (check write latency to the analytics DB), or is the dashboard itself polling infrequently or caching aggressively. Instrumenting each stage with its own metric is what makes this debuggable in minutes instead of guessing — without that, "the dashboard is stale" could be any of four different systems.

**Q4: How would you design tenant-aware caching so one tenant's cached data can never leak to another?**
> Every cache key includes the tenant ID as part of the key itself (e.g., `tenant:123:user:456` rather than just `user:456`), so there's no possibility of one tenant's cached value being returned for another tenant's request — the keys simply never collide across tenants. I'd derive the tenant ID for the cache key server-side from the authenticated session, the same principle as query-level tenant scoping.

**Q5: How do you decide when a system should scale horizontally versus vertically?**
> I'd default to designing for horizontal scaling from the start where reasonably possible — keeping services stateless so any instance can handle any request — since it has a much higher ceiling and avoids a single point of failure. Vertical scaling is a reasonable short-term fix for a stateful component that's hard to horizontally scale (like a single primary database), but it has a hard resource ceiling and doesn't improve availability, so I'd treat it as a stopgap rather than the long-term answer for a component under sustained growth.

---

## PART 3: How It Works Internally

**How RBAC checks are enforced mechanically, end to end**: on login, the auth service issues a token (often a JWT) containing the user's ID, tenant ID, and role(s) as signed claims. On each subsequent request, middleware verifies the token's signature (confirming it hasn't been tampered with) and extracts these claims — no database lookup is needed to know *what* the token claims, only to verify *whether that role still has the specific permission* if permissions can change without re-issuing the token (some systems accept a short staleness window here for performance, refreshing role/permission data periodically instead of on every single request). The actual authorization check is typically a lookup against a permissions table/config (`role → [permissions]`) rather than hardcoded `if role == "admin"` checks scattered through the code, so adding a new role or adjusting permissions doesn't require a code change.

**How tenant scoping is enforced structurally, not just by convention**: the most robust implementations don't rely on every developer remembering to add `WHERE tenant_id = ?` to every query — instead, a base repository/DAO class automatically injects the tenant filter derived from the current request context into every query it builds, so it's structurally impossible to forget. PostgreSQL's row-level security (RLS) can enforce the same guarantee at the database engine level as a backstop — even a query that forgot to filter by tenant would have rows outside the current tenant silently excluded by the database itself, not just the application code.

**How a real-time analytics pipeline maintains freshness under load**: incoming events land in Kafka partitions; a stream processor (which could be a simple consumer group running aggregation logic, or a dedicated stream-processing framework) reads from these partitions continuously, maintaining running aggregates (e.g., a windowed count or sum) in local state, periodically flushing results to a fast-read datastore the dashboard queries. The key internal property that keeps this "real-time": consumers pull continuously rather than on a schedule, so end-to-end latency is bounded by processing speed and network latency, not by a batch interval — but this also means the pipeline's actual freshness is only as good as its slowest stage, which is why per-stage monitoring (mentioned in Q3) is what makes "why is the dashboard stale" answerable instead of a mystery.

**How horizontal scaling actually distributes load, mechanically**: a load balancer distributes incoming requests across multiple stateless service instances (round-robin, least-connections, or consistent hashing depending on the algorithm); because no instance holds session state another instance needs, any instance can handle any request, and adding more instances linearly increases capacity, up to the point where a shared downstream resource (like a single database) becomes the actual bottleneck — which is why horizontally scaling the stateless app tier is usually the easy part, and scaling the stateful data tier (via read replicas, sharding, or caching) is where the harder design work is.
