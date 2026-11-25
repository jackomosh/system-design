# Content Delivery Networks (CDN)

## Overview

A **Content Delivery Network (CDN)** is a geographically distributed system of edge servers that delivers content on behalf of an origin. It reduces latency by keeping copies close to users and improves availability by absorbing traffic near the network edge.

CDNs commonly accelerate:

- images and video
- JavaScript, CSS, and fonts
- downloadable files
- software packages
- static or rendered HTML
- cacheable API responses
- live and on-demand media

CDNs are one type of caching system, but they operate across regions and are managed through edge locations, DNS routing, and HTTP cache behavior.

---

## 1. Why Use a CDN

A CDN helps with:

- **latency:** shorter physical and network distance
- **throughput:** traffic is distributed across many edge locations
- **origin offload:** cached content does not reach the origin
- **availability:** edge caches remain useful during short origin outages
- **traffic spikes:** edge capacity absorbs large bursts
- **security:** WAF, bot management, TLS, and DDoS filtering may operate at the edge
- **global reach:** users receive content through nearby network points
- **bandwidth cost:** reduces origin egress and repeated transport

A CDN may not improve an API with highly personalized, uncacheable, or rapidly changing responses. Those responses still benefit from TLS termination or edge routing, but caching value will be low.

---

## 2. Basic CDN Request Flow

```text
User
  |
  v
Nearest CDN edge
  |
  +--> cached response: return immediately
  |
  +--> cache miss: edge fetches origin
                         |
                         v
                    validate/cache/store
                         |
                         v
                    return to user
```

A cache miss may be fetched by one edge, many edges, or an origin shield depending on topology and cache partitioning.

The origin remains the source of truth for uncached content or when cache entries must be refreshed.

---

## 3. Edge, Regional, and Origin Locations

- **Edge:** geographically close to users and optimized for serving traffic
- **Regional cache:** larger cache tier that aggregates misses from edges
- **Origin shield:** dedicated cache or proxy placed near the origin to reduce fan-out
- **Origin:** application, object storage, or media service producing content

Larger cache tiers reduce origin request fan-out but add a potential hop and can increase miss latency. A topology should match regional demand and origin capacity.

---

## 4. Cache Keys and Variants

A CDN cache key identifies the representation to cache. It commonly includes:

- URL host and path
- protocol
- selected query parameters
- selected headers or cookies
- device or language variants

Every cache key variant that is included increases cache partitions and lowers hit ratio. Variants omitted incorrectly can leak another user's private response.

For public assets, cache by a stable path and content hash:

```text
/app.8f31ac.js
/hero.2c921f.webp
```

For APIs, cache only if authorization and personalization are compatible with the key. Tokens, cookies, and `Set-Cookie` values must not make private responses globally cacheable.

---

## 5. HTTP Cache Headers

CDNs use HTTP semantics together with provider configuration.

### Cache-Control

```http
Cache-Control: public, max-age=86400, s-maxage=3600
```

- `public`: shared caches may store the response.
- `private`: response is for one user and must not be stored by a shared cache.
- `no-store`: response must not be persistently cached.
- `no-cache`: cache may store it but must revalidate before reuse.
- `max-age`: browser cache lifetime.
- `s-maxage`: shared CDN cache lifetime.
- `immutable`: resource is assumed to remain unchanged while fresh.

### ETag

An entity tag identifies a response representation. The client or CDN sends `If-None-Match` and the origin may return `304 Not Modified`.

### Last-Modified

The origin supplies a modification timestamp. The CDN or client may send `If-Modified-Since`.

### Vary

`Vary` indicates that a response differs for another request header, such as `Accept-Encoding` or `Accept-Language`. Excessive variants reduce cache efficiency, while missing variants can serve incorrect content.

---

## 6. Static Asset Delivery

Static assets are ideal CDN candidates:

- URLs are predictable
- responses are reusable
- payloads are large
- client cache headers are effective
- invalidation can use hashed filenames

A deployment pattern is:

```text
Build app.js into app.8f31ac.js
Update HTML to reference app.8f31ac.js
Keep the old asset for a compatibility period
```

Long cache lifetimes become safe because a changed asset has a new URL. This avoids broad CDN purge operations for every deployment.

---

## 7. Dynamic Content

Dynamic content can still be CDN-cached when it is public or has controlled variance.

Examples:

- public home page
- product catalog
- blog article
- public API search result
- regional configuration
- generated documentation

Dynamic caching requires careful use of:

- TTL and stale-if-error
- cache keys
- authentication boundaries
- purge or versioning strategy
- Vary headers
- origin shielding
- surrogate keys or tags

Do not cache personalized account pages by URL alone. Use `private`, `no-store`, or a user-specific cache key only when the edge provider supports it safely.

---

## 8. Cache Miss and Cache Fill

On a miss, the CDN fetches content from an origin. Cache fill behavior affects:

- origin load
- tail latency
- bandwidth
- cache hit ratio
- regional consistency

Common concerns include:

- request coalescing
- origin shield fan-in
- stale-while-revalidate
- stale-if-error
- connection reuse
- compression
- range requests for large media

CDNs can serve stale content temporarily if origin requests fail or the origin is slow. Define the maximum acceptable staleness and never apply it to sensitive or correctness-critical data.

---

## 9. Cache Invalidation and Purging

### URL purge

Invalidate one known URL. It is simple but may miss variant keys.

### Prefix or wildcard purge

Invalidate a URL path set. It is powerful but can cause a burst of misses.

### Surrogate key or cache tag purge

Associate many objects with a tag such as `product:123`, then purge all representations of the tagged entity.

### Versioned URLs

Publish a new filename or path. This is usually the best default for static assets and requires no uncertain global cache purge.

Purges are asynchronous in some CDNs and differ by region. TTL should remain the final bounded fallback.

---

## 10. Edge Security

### TLS termination

A CDN may terminate HTTPS and establish a separate TLS connection to the origin. This reduces client-side crypto work and centralizes certificates, but the origin connection should remain encrypted unless a documented private network provides equivalent security.

### DDoS protection

Distributed edges absorb volumetric traffic and can filter abnormal request patterns before they reach origins.

### Web application firewall

A CDN WAF evaluates HTTP requests for injection, bots, malformed input, and known attacks. It is one layer of protection and cannot correct unsafe application code.

### Bot and abuse control

Edge rules can challenge or block abusive clients, throttle expensive routes, and enforce geographic or API policies.

Security controls should log and test their false-positive effects. Blocking legitimate crawlers or APIs can cause severe outages.

---

## 11. Origin Shielding

Without shielding, every edge location that misses may independently request the origin.

An origin shield is an intermediate cache or gateway that consolidates requests.

Benefits:

- lower origin request fan-out
- fewer cache fills
- improved origin protection
- potentially higher cache hit ratio

Costs:

- extra network hop
- may reduce regional independence
- can become a bottleneck or failure point

Choose the shield region based on origin location, traffic distribution, and latency.

---

## 12. Global Routing

CDNs and global load balancers use DNS, Anycast, or provider-managed routing to direct clients toward available edges.

Routing can consider:

- geographic distance
- network performance
- edge health
- regional capacity
- regulatory boundaries
- failover policy

Geo-routing does not guarantee data residency. Application data storage and regional replication require separate policy design.

---

## 13. Media and Streaming

CDNs accelerate:

- on-demand video segments
- live stream segments
- audio
- downloads
- software updates
- software binary releases

For video, cache short segment files and manifests while allowing the origin or live ingest system to control playback availability.

Use range requests, chunked delivery, resumable downloads, signed URLs for private media, and appropriate cache lifetimes for live manifests.

Live manifests should be short-lived or revalidated frequently. Immutable media segments can be cached for much longer.

---

## 14. Signed URLs and Private Content

Public caching is unsafe for access-controlled content. Private content can use:

- signed URLs
- signed cookies
- edge authentication
- short-lived tokens
- origin authorization

A signed URL should contain bounded expiry and scope. It should not expose a reusable secret or remain valid longer than necessary.

The edge must avoid mixing an authorized response with a generic cache key. Authorization can occur at the edge, after which only a public-safe representation may enter a shared cache.

---

## 15. Compression and Optimization

CDNs can compress text responses with Brotli or gzip. Compression should be applied only when it reduces total payload and latency.

Other optimizations include:

- HTTP/2 or HTTP/3 delivery
- connection reuse
- image format conversion
- responsive image variants
- range and byte serving
- preloading critical assets
- removing unnecessary cookies at edge paths

Image transformation at the edge can reduce bandwidth but adds compute cost and cache variants. Cache the transformed representation by source image and transformation parameters.

---

## 16. Observability

Measure:

- cache hit and miss ratio
- byte hit ratio
- origin request count
- request and byte egress
- p50/p95/p99 latency
- status and error rates
- stale response rate
- regional traffic
- purge coverage and completion
- bot and WAF block rates
- origin capacity and failure rate

A CDN dashboard should correlate edge, origin, and application metrics. Low latency may hide origin pressure if hit ratios are decreasing or cache fills are expensive.

---

## 17. Failover Behavior

A CDN failure policy should define:

- origin bypass or failover behavior
- stale-if-error windows
- regional edge fallback
- DNS TTL tradeoffs
- retry and timeout budgets
- error pages and health checks

For important systems, test that origins remain reachable only by intended networks and that CDN outages fail safely. Do not set a very high origin timeout because an edge outage can block every request.

---

## 18. When a CDN Is Useful

Use a CDN for:

- globally distributed static assets
- public pages or catalogs
- large binary downloads
- streaming and video
- traffic absorption and DDoS mitigation
- TLS termination and edge security
- cacheable API responses
- origin bandwidth reduction

A CDN is less useful by itself for:

- uncacheable personalized APIs
- write-heavy transactions
- data that must always be fresh
- private responses that differ for every request
- workloads that cannot tolerate stale data

It may still provide routing, TLS, DDoS, and WAF benefits to those workloads.

---

## 19. CDN Design Checklist

Before production deployment:

1. Identify cacheable and non-cacheable routes.
2. Define cache keys and variants.
3. Set TTL and stale policies.
4. Prevent private data from entering shared caches.
5. Configure purge or hashed asset versioning.
6. Add origin shielding if needed.
7. Configure TLS and private origin access.
8. Establish WAF and bot policy.
9. Monitor hit ratio, origin load, latency, and errors.
10. Test origin failure and regional failover.

---

## 20. Summary

A CDN moves reusable content closer to users and reduces origin load by distributing cache and delivery capacity across the edge. It improves latency, throughput, global availability, DDoS resilience, and bandwidth economics.

Effective CDN use requires correct cache headers and keys, bounded freshness, private-data isolation, purge and versioning strategy, origin shielding, security controls, and measurable monitoring. Hashing static assets and keeping private or highly dynamic responses uncached are two of the most reliable production practices.
