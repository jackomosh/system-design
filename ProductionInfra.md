# Production Infrastructure

## Overview

**Production infrastructure** is everything required to run an application reliably outside development: compute, networking, storage, containers or virtual machines, deployment pipelines, secrets, monitoring, backups, security, and incident operations.

Production readiness is not only deployment. A service is production-ready when it can be deployed, observed, scaled, secured, recovered, and operated with clear ownership and acceptable risk.

---

## 1. Production Responsibilities

A production platform typically provides:

- runtime infrastructure
- load balancing and traffic routing
- DNS and TLS certificates
- identity and secrets management
- databases and caches
- deployment automation
- autoscaling and capacity management
- monitoring and alerting
- centralized logs and tracing
- backup and disaster recovery
- security and network isolation
- incident response and rollback
- cost and resource accounting

Platform teams and application teams share responsibility. Ownership boundaries should be explicit.

---

## 2. Compute

### Virtual machines

Virtual machines provide a complete OS and are useful for legacy dependencies, strict isolation, or workloads that require custom kernel or OS configuration.

Benefits:

- familiar operational model
- strong isolation
- broad software compatibility

Costs:

- slower provisioning
- more OS patching
- lower resource density

### Containers

Containers package application code, runtime dependencies, and configuration. They provide portable and repeatable deployment units.

Benefits:

- fast startup
- consistent build and runtime environment
- efficient resource sharing
- support for scaling and rollout automation

Costs:

- image supply-chain risk
- container runtime and networking complexity
- need for orchestration at larger scale

### Serverless functions

Serverless platforms run request-triggered code with provider-managed compute. They are useful for event processing, periodic tasks, webhooks, and bursty workloads.

Benefits:

- no server management
- automatic scaling
- fine-grained cost for some workloads

Costs:

- cold starts
- execution and payload limits
- vendor coupling
- difficult local and distributed debugging
- cost and latency surprises at sustained scale

---

## 3. Containers and Kubernetes

A container image is an immutable deployment artifact. Production containers should:

- run as a non-root user
- have minimal base images
- pin or digest dependencies
- use read-only filesystems where possible
- declare CPU and memory resources
- include health checks
- scan for vulnerable packages
- avoid embedded secrets

**Kubernetes** orchestrates containers through resources such as Deployments, Services, Pods, ConfigMaps, Secrets, Ingress or Gateway resources, and Jobs.

Important Kubernetes concepts include:

- desired state reconciliation
- liveness, readiness, and startup probes
- horizontal pod autoscaling
- rolling deployments and rollbacks
- resource requests and limits
- namespaces and network policies
- persistent volumes
- service discovery
- pod disruption budgets

Orchestrators add operational leverage but are not required for every system. Use managed Kubernetes or a simpler platform if the application and team can operate it reliably.

---

## 4. Service Architecture

### Monolith

A monolith packages application components in one deployable unit. It can be fast to develop and operationally simple for small teams.

Benefits:

- simple deployment
- in-process calls
- fewer network failures
- straightforward transactions and observability

Costs:

- scaling is coarse-grained
- code coupling can increase
- release blast radius is broad

### Microservices

Microservices divide the system into independently deployable business services.

Benefits:

- independent scaling and release
- team ownership boundaries
- fault isolation
- technology flexibility

Costs:

- distributed transactions
- network latency and partial failures
- operational overhead
- difficult debugging and contract management

Do not split services solely because a diagram looks modern. Independent deployment, team ownership, and scaling requirements should justify distribution.

---

## 5. Networking

### Virtual networks

Production systems use private networks, subnets, route tables, and security groups or network policies to isolate components.

Common zones:

- public load balancers
- application subnets
- private data subnets
- administrative access paths
- egress gateways

Databases and caches should not be directly public unless there is a strong, documented requirement.

### DNS

DNS maps names to addresses or routing providers. Production DNS requires:

- low-enough TTL for failover
- health-aware routing when needed
- domain ownership and registrar security
- certificate automation
- no stale internal or public records

### TLS

Use TLS for public and internal service traffic where practical. Terminate TLS at a load balancer or edge with secure origin connections behind it. Centralize certificate renewal and alert on expiration.

### Load balancing

Load balancers distribute traffic across app instances, health-check backends, and remove unhealthy nodes. Configure timeouts and retry limits carefully to prevent a slow backend from amplifying failures.

---

## 6. Databases and Storage

Production storage systems require:

- backups and restore tests
- replication or multi-AZ design
- monitoring
- capacity planning
- encryption at rest
- access control
- retention policies
- failover drills
- data migration tools

Object storage is suitable for files, logs, media, and backups. Block storage supports disks and databases. File storage supports shared POSIX-like workloads. The storage type should match access and consistency needs.

Database scaling often begins with indexing and query optimization, then read replicas, caching, partitioning, or sharding.

---

## 7. High Availability

**High availability** means the system continues serving within an expected degradation envelope after component failures.

Improve availability with:

- multiple instances
- health checks
- load balancers
- managed databases or replication
- multi-zone deployment
- graceful degradation
- queues and retries
- capacity for one or more component failures
- backups and disaster recovery

Availability is often expressed as a percentage:

```text
99.9% uptime = about 43.8 minutes downtime/month
99.99% uptime = about 4.38 minutes downtime/month
```

Such targets are not simply achieved with a larger server. They require architecture, operations, dependencies, budget, and error budgets.

---

## 8. Reliability Patterns

### Redundancy

Run enough independent instances or zones for the workload. Avoid a regional, subnet, or deployment pipeline single point of failure.

### Failover

Automatic or manual failover moves traffic or writes to a healthy system. Test failover periodically and monitor split-brain and data consistency.

### Timeouts

Every network call needs a timeout. The timeout should account for downstream SLOs and the caller's remaining request budget.

### Retries

Retry only transient failures. Use exponential backoff and jitter, and avoid retry storms. A retrying call should be idempotent or protected by an idempotency key.

### Bulkheads

Separate pools and quotas by subsystem or tenant so that one workload cannot consume all capacity.

### Circuit breakers

Stop calling a failing dependency after a threshold and recover with controlled probes. Return cached or degraded responses where safe.

### Graceful degradation

Deliver reduced functionality rather than an entire outage. Examples include disabling recommendations, serving cached catalog data, or postponing non-critical notifications.

---

## 9. Autoscaling

Autoscaling adds or removes capacity based on metrics.

Common signals include:

- CPU utilization
- memory use
- request count
- queue depth
- event lag
- concurrency
- response latency
- custom business metrics

A scale-up policy may react in seconds, while capacity may start in minutes. A scale-down policy must avoid oscillation and maintain spare capacity.

Configure:

- minimum and maximum replicas
- stabilization windows
- target utilization
- headroom
- warm-up behavior
- cooldown periods
- pod or instance limits

For queue-based workloads, scale workers based on backlog and processing latency rather than CPU alone.

---

## 10. CI/CD

**Continuous Integration (CI)** automatically builds and tests changes. **Continuous Delivery/Deployment (CD)** safely promotes tested artifacts to environments.

A production pipeline should:

1. Build an immutable artifact.
2. Run unit and integration tests.
3. Run security and dependency checks.
4. Validate infrastructure and configuration.
5. Deploy to a staging environment.
6. Run smoke and contract tests.
7. Deploy gradually to production.
8. Measure health and rollback on failure.

Prefer promoting the same artifact across environments instead of rebuilding it in every stage.

---

## 11. Deployment Strategies

### Rolling deployment

Replace instances gradually. It has low overhead but may temporarily mix old and new versions.

### Blue-green deployment

Run two environments and switch traffic to the new one. Fast rollback is possible, but duplicated capacity may be costly.

### Canary release

Send a small percentage of traffic to the new version. It detects regressions before broad rollout.

### Shadow testing

Send copies of production traffic to a new implementation without using its responses. It can reveal capacity or correctness issues but may create side effects.

Every deployment needs:

- backward compatibility
- database migration sequencing
- health and smoke checks
- rollback procedure
- feature flags when needed
- monitoring guardrails

Expand-and-contract database migrations avoid requiring old and new application versions to agree on one schema transition:

1. add a compatible column or table
2. deploy dual-write or backfill logic
3. migrate readers
4. remove old structures only after both versions are retired

---

## 12. Configuration and Secrets

Configuration should be:

- environment-specific
- version-controlled for non-secret values
- validated at startup
- immutable or controlled after deployment
- traceable to the release

Secrets should be:

- stored in a secret manager or encrypted store
- injected at runtime rather than embedded in image or source
- least privileged
- rotated without downtime
- audited
- revoked when exposure is detected

Never put secrets in logs, query parameters, shell history, public environment definitions, or source files.

---

## 13. Observability

### Metrics

Metrics are numerical measurements over time:

- latency
- throughput
- error rate
- CPU and memory
- queue depth
- saturation
- cache hit ratio
- database connections

Use p50, p95, and p99 latency rather than averages alone.

### Logs

Logs record events. Use structured, correlatable fields and avoid sensitive payloads. A log may include:

- timestamp
- severity
- service
- trace and request IDs
- operation
- result status
- safe context

### Traces

Distributed traces show one request across services. Propagate trace headers and measure downstream spans, retries, and bottlenecks.

### Alerting

Alert on user-impacting symptoms and sustained SLO risk, not every possible implementation detail.

Good alerts include:

- clear description
- severity and ownership
- affected service or tenant
- runbook link
- query or dashboard link
- symptom and threshold
- escalation path

Avoid noisy alerts that train operators to ignore pages.

---

## 14. SLOs, SLIs, and Error Budgets

An **SLI** is a measured service level indicator, such as successful request percentage or p95 latency.

An **SLO** is an objective, such as 99.9% successful requests in a 30-day window.

An **error budget** is the allowed amount of failure under the SLO.

Example:

```text
Target: 99.9% successful responses
30-day budget: 0.1% failed or invalid responses
```

Error budgets help balance feature velocity and reliability. Teams may slow releases or invest in resilience when budgets are exhausted.

---

## 15. Backups and Restore

Backups are useful only if restoration has been tested.

Design for:

- backup frequency
- retention period
- encrypted storage
- immutable or isolated copies
- regional recovery
- restore time objective (RTO)
- recovery point objective (RPO)
- application consistency
- validation and reconciliation

A backup system is not enough for a logical data bug that has already replicated. Retention, point-in-time recovery, and replayable events may be required.

---

## 16. Disaster Recovery

A disaster recovery plan defines how to recover from a major outage such as regional loss, database corruption, or cloud account compromise.

Common strategies:

- **Backup and restore:** lower cost, longer recovery
- **Pilot light:** minimal core infrastructure stays warm
- **Warm standby:** scaled-down replica receives or synchronizes data
- **Active-active:** regions serve traffic concurrently

Evaluate failover cost, data consistency, identity dependencies, DNS behavior, capacity, and runbook steps. Perform recovery drills and record actual recovery time.

---

## 17. Security in Production

Core practices include:

- identity-based access and MFA
- least privilege
- short-lived credentials
- private networking
- host and container hardening
- image and dependency scanning
- secret rotation
- audit logs
- encryption in transit and at rest
- vulnerability patching
- firewall and WAF rules
- tenant isolation
- supply-chain verification
- incident detection and response

Production access should be audited and temporary. Break-glass accounts require strong storage, MFA, monitoring, and regular review.

---

## 18. Incident Response

A mature incident process has:

1. Detection and severity assignment.
2. Incident lead and response roles.
3. Communication and status updates.
4. Mitigation before root-cause investigation.
5. Evidence and timeline capture.
6. Recovery verification.
7. Post-incident review.

Mitigation examples include rollback, traffic shedding, dependency isolation, failover, or feature disabling.

Post-incident reviews should focus on systemic causes and fixes rather than individual blame. Follow corrective actions with owners and deadlines.

---

## 19. Capacity Planning

Capacity planning estimates traffic, storage, CPU, memory, network, and database capacity under:

- normal load
- seasonal spikes
- launch events
- failures
- migrations
- regional failover
- growth forecasts

Load testing should include realistic concurrency, payload size, cache state, retries, and downstream dependencies. A benchmark that bypasses database, network, or queue behavior is not representative.

Track bottleneck metrics and reserve headroom for incident and autoscaling response.

---

## 20. Cost Optimization

Production cost should be measured without sacrificing reliability.

Techniques include:

- rightsizing instances
- autoscaling to demand
- reserved or savings plans for stable baseline capacity
- cache and CDN reduction of compute and egress
- storage lifecycle policies
- log sampling and retention tiers
- elimination of unused resources
- workload placement in appropriate regions
- cost tags and ownership attribution

A low-cost configuration that fails under launch traffic is not economically optimal. Compare cost against error budgets and revenue or user impact.

---

## 21. Production Readiness Checklist

Before launch:

- architecture and dependency ownership are documented
- SLOs and capacity targets are set
- health checks and graceful shutdown are implemented
- TLS and secrets are configured
- authentication and authorization are enforced
- rate limits and abuse controls are configured
- monitoring, tracing, and alerts are validated
- runbooks exist for common failures
- backups and restores have been tested
- failover and recovery have been tested
- rollback and feature kill switches are available
- deployment is repeatable and immutable
- database migrations are compatible
- load testing covers peak and degraded operation
- security review and vulnerability management are complete
- on-call ownership and escalation are clear

---

## 22. Summary

Production infrastructure is the combination of runtime, network, data, deployment, observability, security, and operations required to deliver reliable service. It must handle success, partial failure, malicious traffic, growth, and recovery.

The strongest production systems are deliberately simple: clear ownership, immutable releases, small deployable units, redundant capacity, bounded dependencies, tested backups, actionable telemetry, and rehearsed incident procedures.
