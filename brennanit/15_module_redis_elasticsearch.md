# Module 5: Redis (Caching) & ElasticSearch (Search)

---

## PART 1: Fundamentals (must know cold)

- **Cache-aside pattern**: app checks cache first → on miss, reads DB → writes result to cache → returns it. The most common caching pattern and the one to default to if asked "how would you add caching here."
- **TTL (time-to-live)**: an expiration on a cached key so stale data self-heals over time even without explicit invalidation.
- **Cache invalidation**: explicitly deleting/updating a cache entry when the underlying data changes, so reads immediately after a write aren't stale.
- **Redis data structures**: not just key-value strings — also lists, sets, sorted sets (great for leaderboards/rate limiting), hashes. Knowing this shows depth beyond "Redis = a cache."
- **Cache stampede**: many requests missing the cache simultaneously (e.g., right after a hot key expires) and all hammering the DB at once.
- **ElasticSearch's inverted index**: maps each term to the list of documents containing it — this is what makes full-text search fast, versus a relational DB's `LIKE '%term%'`, which can't use a normal index for substring matching.
- **When to use ElasticSearch vs a DB's native search**: ElasticSearch for full-text search, relevance ranking, fuzzy matching, faceted search over large text corpora; a relational DB's indexed columns for exact-match/range queries on structured data.

---

## PART 2: Interview Questions & Answers

**Q1: Walk me through the cache-aside pattern and why you'd choose it.**
> On a read, the app checks Redis first. If the key exists (a hit), return it immediately. If not (a miss), query the database, then store the result in Redis with a TTL before returning it. I'd choose it because it's simple, and the app has full control over what gets cached and for how long — the cache only ever holds what's actually been requested, rather than pre-populating everything.

**Q2: How do you prevent stale data after a write?**
> On update, I explicitly invalidate (delete or overwrite) the corresponding cache entry as part of the same write operation, so the next read is a cache miss that repopulates with fresh data. For data where a few seconds of staleness is acceptable, a short TTL alone is often good enough without needing explicit invalidation everywhere.

**Q3: What's a cache stampede, and how would you prevent one?**
> It's when a popular ("hot") cache key expires, and a burst of concurrent requests all miss the cache at the same moment and hit the database simultaneously — which can overload the DB even though the cache was supposed to protect it. I'd prevent it with a lock-based approach: the first request to miss acquires a short lock and repopulates the cache, while other concurrent requests either wait briefly for the repopulated value or serve a slightly stale value instead of all hitting the DB at once.

**Q4: Why is ElasticSearch faster than a SQL `LIKE '%term%'` query for text search?**
> A relational DB's B-tree index is built for exact-match and range queries on ordered values — it can't help with matching a substring anywhere inside a text field, so `LIKE '%term%'` typically forces a full scan. ElasticSearch instead builds an inverted index at write time — mapping each individual term to the documents containing it — so a search for a term is a direct lookup into that index, not a scan of every document's text.

**Q5: Beyond simple caching, what's a good use of Redis's other data structures?**
> Sorted sets are a good fit for a leaderboard or rate limiter — each member has a score, and Redis keeps them ordered, so getting "top 10" or "this user's rank" is efficient without re-sorting in application code. A sliding-window rate limiter can also be implemented with a sorted set keyed by timestamp, trimming entries older than the window on each check.

---

## PART 3: How It Works Internally

**How Redis stays fast**: Redis is an in-memory data store — all data lives in RAM, which is why reads/writes are on the order of microseconds versus milliseconds for a disk-backed database. It's mostly single-threaded for command execution (avoiding lock contention overhead for most operations), relying on an efficient event loop to handle many concurrent client connections. Durability (surviving a restart) is optional and configurable — via periodic snapshots (RDB) or an append-only log of writes (AOF) that can be replayed on restart — trading some durability guarantees for raw speed compared to a disk-first database.

**How TTL expiration actually happens internally**: Redis doesn't necessarily delete an expired key the instant its TTL hits zero. It uses a combination of **lazy expiration** (checked when the key is accessed — if expired, it's deleted then and treated as a miss) and **active expiration** (a background process periodically samples a subset of keys with TTLs and proactively removes expired ones), balancing memory reclamation against the CPU cost of constantly scanning every key.

**How ElasticSearch's inverted index is built and queried**: at index time, each document's text is analyzed — tokenized (split into words), lowercased, and often stemmed (e.g., "running" → "run") — and each resulting term is added to the inverted index with a pointer back to the document(s) containing it. At query time, a search term goes through the same analysis process, and ElasticSearch looks up matching terms in the index directly rather than scanning documents, then combines/scores multiple term matches (using relevance scoring like BM25, which factors in term frequency and document rarity) to rank results by relevance rather than just returning an unordered match list.

**How a distributed cache/search cluster scales**: both Redis (in cluster mode) and ElasticSearch shard data across multiple nodes based on a hash of the key/document ID, so no single node has to hold the entire dataset in memory, and read/write load is spread across the cluster. Each shard is typically replicated to at least one other node, so a node failure doesn't lose data — the same replication principle as Kafka's partition leader/follower model.
