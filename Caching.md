# Caching

## Overview

A **cache** is a faster or closer copy of data used to avoid repeating expensive work. It may store database query results, computed values, API responses, rendered pages, sessions, or static files.

Caching is useful because most systems have **locality**: recently used or frequently used data is likely to be used again. A cache trades extra storage and complexity for lower latency, higher throughput, lower origin load, and better availability.

A cache is not the source of truth unless deliberately designed to be authoritative. In most architectures, the database or another durable system remains canonical and the cache contains a copy that may be stale.

---

## 1. Why Caching Exists

Caching helps systems:

- reduce latency
- improve throughput
- reduce database and service load
- absorb traffic spikes
- improve availability during short upstream failures
- reduce cost for compute, bandwidth, and database capacity
- bring frequently requested data closer to users
- avoid repeated serialization and rendering
- protect origin systems from repetitive work

A cache is not automatically beneficial. It can add inconsistency, complexity, memory cost, and failure modes. Cache only when a measured bottleneck or repeated workload makes the tradeoff worthwhile.

---

## 2. Cache Hit and Cache Miss

- **Cache hit:** requested data is found in the cache.
- **Cache miss:** data is not found and must be retrieved or calculated from the origin.

A cache hit ratio is:

```text
cache hits / (cache hits + cache misses)
```

Useful metrics include hit ratio, miss latency, eviction count, memory utilization, stale response count, and cache-request error rate.

A high hit ratio does not guarantee good performance. A cache may have many hits for unimportant data while frequently missing critical hot keys.

---

## 3. Common Cache Levels

### Browser cache

Browsers cache HTTP responses, images, scripts, fonts, and API responses according to headers or application rules. This gives the lowest round-trip latency because no network request may be required.

### CDN cache

CDNs cache content at edge locations near users. It is especially effective for static assets and cacheable public dynamic responses.

### Reverse proxy cache

Nginx or another reverse proxy may cache complete HTTP responses, reducing application work.

### Application cache

An application process or framework caches rendered fragments, query results, or in-process objects.

### Distributed cache

Redis or Memcached stores shared cached data outside application instances. This is common for session data, database query results, counters, rate limits, and hot objects.

### Database cache

Databases maintain internal buffers and query caches. Administrators generally tune the database before relying on application caching for the same access pattern.

A typical stack may contain browser, CDN, reverse proxy, distributed cache, and database caches. Each level has a larger capacity and larger latency as requests move toward the origin.

---

## 4. Cache-Aside

**Cache-aside** is the most common pattern.

```text
Read:
1. Application checks cache.
2. On hit, return cached data.
3. On miss, read the database.
4. Store the result in cache.
5. Return the result.

Write:
1. Application writes the database.
2. Application updates or deletes the cache entry.
```

Advantages:

- cache failures do not prevent database reads
- cache contains only data that is actually requested
- simple to adopt incrementally

Disadvantages:

- cache misses may cause duplicate database reads
- application code must manage cache consistency
- concurrent writers can create race conditions

Cache invalidation is often handled by deleting the key after a successful write. The next reader repopulates it.

---

## 5. Read-Through

With **read-through caching**, the cache library or service loads missing data from the origin automatically.

```text
Application -> Cache
                 |
                 +-> Origin on miss
```

Advantages:

- application code is simpler
- cache-loading behavior is centralized

Disadvantages:

- cache becomes a required read dependency
- loading logic must support origin failures and timeouts

---

## 6. Write-Through

With **write-through caching**, the application updates the cache and origin in the write path.

```text
Application -> Cache -> Database
```

The cache is usually updated only after the origin write succeeds or as part of a coordinated transaction.

Benefits:

- cache stays more up to date
- reads have fewer misses for recently written objects

Costs:

- every write increases latency
- cache capacity is populated even for data that may never be read
- write path has an extra dependency

---

## 7. Write-Behind or Write-Back

With **write-behind caching**, an application first writes to a fast cache and a worker later writes to the durable database.

Benefits:

- low write latency
- batching and smoothing of write spikes
- high write throughput

Costs:

- higher complexity
- risk of data loss if buffered writes are not durable
- possible ordering and replay challenges

Write-behind is appropriate for workloads that accept an asynchronous durability model and have careful persistence and recovery design. It is dangerous for financial or strongly consistent core records unless durably queued with strict failure handling.

---

## 8. TTL and Expiration

A **time-to-live (TTL)** specifies how long an entry may remain cached.

Examples:

- static asset: hours or days
- public catalog: minutes
- rapidly changing order status: seconds
- financial balance: very short or no cache

TTL balances freshness and hit ratio:

- shorter TTL: fresher data, lower hit ratio, more origin traffic
- longer TTL: lower latency and origin load, but more stale data

TTL expiration is a maximum retention window, not an instantaneous update mechanism. Add jitter to long TTLs or refresh times to avoid synchronized expiration and cache stampedes.

---

## 9. Cache Invalidation

Cache invalidation means removing or replacing stale entries when source data changes.

### Key deletion

Delete the cached object after updating the source. This is simple and often preferred for cache-aside.

### Versioned keys

Include a version or entity version in cache keys:

```text
user:123:v8
```

Old entries remain but are no longer referenced. This avoids in-place update races and works well for immutable content.

### Event invalidation

Publish a data-change event and have cache workers delete or refresh affected keys. Events should be idempotent and should not be the only source of correctness.

### Passive expiration

Use TTL and accept that entries expire eventually.

### Refresh-ahead

Proactively refresh popular keys before expiration.

A strong invalidation strategy defines:

- what is stale
- who owns invalidation
- expected propagation time
- behavior after missed invalidation
- fallback TTL
- ordering across distributed services

As Phil Karlton said, cache invalidation is one of computing's hard problems. The safest design makes every cached value rejectably stale and includes a bounded TTL.

---

## 10. Cache Stampede

A **cache stampede** occurs when many requests miss the same expired key and simultaneously reload the origin.

Mitigations:

- TTL jitter
- request coalescing or locking
- refresh-ahead
- stale-while-revalidate
- serving stale data temporarily
- negative caching with short TTL
- reduced concurrency via a per-key lease

Negative caching stores a short result that an object does not exist, preventing repeated database queries for misspelled IDs or unavailable records. It can unintentionally cache a transient deletion or create privacy issues; use clear ownership and short TTLs.

---

## 11. Hot Keys and Large Values

A hot key receives disproportionate traffic. Mitigation:

- replicate the value across cache nodes or application memory
- partition or shard the key differently
- use a CDN or edge cache
- rate limit abusive access
- add randomization or local caching

Large values increase memory and network cost. Compress only when CPU and latency tradeoffs are favorable, and split collections into bounded objects or pages.

Cache keys should be:

- deterministic
- bounded in length
- namespaced
- version-aware
- free of secrets or sensitive personal data

---

## 12. Stale-While-Revalidate

Stale-while-revalidate allows a cache to serve stale data immediately while refreshing it asynchronously.

Benefits:

- avoids blocking on cache misses
- preserves availability
- smooths traffic spikes

Risks:

- stale data can be visible longer
- refresh can fail and leave stale data indefinitely
- not appropriate for correctness-critical operations

It is useful for product catalogs, public pages, recommendations, and non-critical dashboards, but less suitable for payment balances or inventory reservations.

---

## 13. Distributed Cache Patterns

### Redis

Redis provides in-memory structures, TTL, pub/sub, atomic operations, replication, persistence options, clustering, and Lua scripts. Common uses include caching, sessions, rate limiting, locks, and queues.

### Memcached

Memcached is a simpler distributed key-value cache optimized for memory efficiency and high throughput. It is less rich than Redis for data structures and coordination.

Redis or Memcached is useful when cache entries must be shared across application instances. A cache cluster needs replication, failover, eviction policy, memory sizing, and monitoring.

A cache may have side effects in distributed deployments:

- failover can lose volatile state
- node replacement can create stampedes
- network partitions isolate cache availability
- hot keys can overload a shard

Design cache fallbacks to work under failure.

---

## 14. Consistency and Correctness

Cache consistency must match the business requirement:

- **Strong consistency:** cache is not allowed to return conflicting stale data.
- **Bounded staleness:** data may be stale for a known duration.
- **Eventual consistency:** data converges after propagation.
- **Best-effort freshness:** stale data is acceptable for some features.

A read-modify-write operation should not mutate cache state based on stale cached data. Compute from the source or use atomic operations at the cache or database.

Never cache:

- passwords or raw credentials
- unencrypted secrets
- private data shared across authorization boundaries
- correctness-critical values without a documented freshness policy

Cache entries must preserve authorization and tenant isolation. A response cached for one user must not be returned to another.

---

## 15. Eviction Policies

When memory is full, an eviction policy decides which entries to remove:

- **LRU:** least recently used
- **LFU:** least frequently used
- **FIFO:** oldest inserted
- **TTL-based:** expired entries first or as they expire
- **Random:** simple fallback

LRU and LFU differ by workload. Sequentially scanning a large dataset may evict truly popular entries under a poor policy. Evaluate with production-like access patterns.

Set memory limits and monitoring. Cache exhaustion should degrade predictably to misses rather than destabilize the host or database.

---

## 16. Security Considerations

- use TLS for cache traffic
- restrict network access and require authentication
- isolate tenants and namespaces
- avoid caching sensitive response fields when unnecessary
- set secure TTL and eviction controls
- validate data loaded from cache before using it
- clear or rotate entries after authorization changes
- do not expose cache keys or internals publicly

A distributed cache is often a high-value data store and must receive the same threat modeling as a database.

---

## 17. When to Use Caching

Caching is a strong fit when:

- reads greatly exceed writes
- data is expensive to retrieve or compute
- values have acceptable bounded staleness
- data is accessed repeatedly
- origin systems need protection from spikes
- global users benefit from edge or local copies
- the cache key and invalidation strategy are clear

Caching may be inappropriate when:

- every read must reflect the latest write
- authorization differs sharply per response
- data volume exceeds the cache budget
- invalidation boundaries are unclear
- the cache introduces more operational risk than measured benefit

---

## 18. Cache Design Checklist

Before deploying a cache, decide:

1. What data is cached and why?
2. What is the source of truth?
3. Is the data safe to share across users or tenants?
4. What is the TTL and maximum acceptable staleness?
5. How are writes invalidated or propagated?
6. How are stampedes, hot keys, and large values handled?
7. What happens if the cache fails or has high latency?
8. How are hit, miss, eviction, memory, and stale-serving metrics measured?
9. How are secrets and network access protected?
10. What is the rollback and eviction procedure?

---

## 19. Summary

Caching improves latency, throughput, availability, and cost efficiency by storing reusable results close to the consumer or application. It is most effective with repeated reads, expensive calculations, bounded staleness, and clear invalidation ownership.

The key tradeoff is not storage: it is **freshness versus performance**. Good caching requires TTLs, invalidation, stampede protection, tenant isolation, monitoring, and a safe fallback. A cache should improve a working system rather than hiding an unknown source of database or application inefficiency.
