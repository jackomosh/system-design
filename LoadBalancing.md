# Load Balancing

## Overview

Load balancing is the process of distributing incoming traffic across multiple servers so that no single server becomes overloaded. It is one of the core building blocks of scalable, resilient systems.

At a high level, a load balancer sits in front of one or more application servers and routes each incoming request to a healthy backend instance.

This helps with:

- higher availability
- better scalability
- reduced latency
- fault tolerance
- more efficient use of infrastructure

---

## 1. Why Load Balancing Matters

Without a load balancer, requests go directly to a single server. This creates several problems:

- one server becomes a bottleneck
- if that server fails, the system goes down
- traffic spikes can overwhelm the application
- scaling is limited to a single machine

By introducing a load balancer, we can:

- distribute traffic across multiple servers
- send requests away from unhealthy instances
- increase capacity without changing the app itself
- improve reliability and uptime

This is one of the first major steps in moving from a single-server design to a scalable architecture.

---

## 2. Core Benefits of Load Balancing

### Traffic distribution

A load balancer spreads incoming requests across multiple app instances. This prevents one server from receiving all traffic and being overloaded.

### Fault tolerance

A load balancer can detect unhealthy servers and stop sending traffic to them until they recover.

### Scalability

As traffic increases, we can add more servers behind the load balancer. The service can scale horizontally without changing the client-facing endpoint.

### Improved availability

If one server fails, the system continues serving traffic through the others.

### Better resource utilization

The load balancer can route requests to the least busy or most responsive server, improving resource efficiency.

---

## 3. Basic Load Balancing Architecture

A standard setup looks like this:

```text
Client
  |
  v
Load Balancer
  |
  +--> Server 1
  +--> Server 2
  +--> Server 3
```

The client connects to the load balancer instead of a single origin server.

The load balancer receives traffic and forwards it to the best backend based on its algorithm and health checks.

---

## 4. How Load Balancers Detect Faults

A good load balancer does not just route traffic; it also monitors health.

Common health checks include:

- HTTP 200 response from the server
- TCP connection success
- response time thresholds
- custom health endpoint checks

If a server fails or becomes unhealthy, the load balancer removes it from the eligible pool and sends traffic to healthy backends only.

This helps maintain system stability even when some instances fail.

---

## 5. Types of Load Balancing Algorithms

Different algorithms fit different workloads. The “best” choice depends on traffic patterns, server capacity, and application behavior.

---

## 6. Round Robin

Round robin is the simplest algorithm.

The load balancer sends requests to servers in rotation:

- Server 1 receives request 1
- Server 2 receives request 2
- Server 3 receives request 3
- then it cycles back to Server 1

### Advantages

- simple to implement
- fair distribution for equal servers
- easy to understand

### Disadvantages

- assumes all servers are equally powerful
- ignores differences in CPU, memory, or current workload
- not ideal when backends have different capacities

### Best for

- homogeneous server pools
- simple web applications
- equal-capacity backend nodes

---

## 7. Least Connections

This algorithm forwards traffic to the server with the fewest active connections.

It is useful when requests have different durations and some servers may be more loaded than others.

### Advantages

- better than round robin when servers have uneven workloads
- helps prevent overloading one server
- works well for long-lived connections

### Disadvantages

- still does not consider server capacity or response time directly
- connection count is not always a perfect signal of real workload

### Best for

- systems with uneven request durations
- workloads where connection count is a good indicator of load

---

## 8. Least Response Time

The load balancer chooses the server with the fastest recent response time and/or lowest current latency.

This algorithm is more performance-aware than round robin or least connections.

### Advantages

- picks the fastest server for user traffic
- improves responsiveness under uneven backend performance
- useful when some servers are slower than others

### Disadvantages

- requires more monitoring and metrics
- more complex to implement correctly
- not always necessary for simple systems

### Best for

- latency-sensitive apps
- systems with uneven server performance
- environments with real-time responsiveness needs

---

## 9. IP Hash

The load balancer calculates a hash based on the client IP address (or sometimes the source address and port) and maps it to a backend server.

This means the same client usually stays on the same server.

### Advantages

- helps with sticky sessions
- useful when backend servers keep session data locally
- can improve consistency for some workloads

### Disadvantages

- if a server goes down, redistributing traffic can be uneven
- not ideal for stateless services
- can create imbalances if client IP distribution is uneven

### Best for

- apps that depend on session affinity
- legacy systems that store session state locally
- environments where a client should consistently hit the same server

---

## 10. Weighted Algorithms

Weighted load balancing assigns different traffic percentages to each server based on capacity.

Example:

- Server A weight = 3
- Server B weight = 1
- Server C weight = 2

Then the scheduler directs traffic in proportion to those weights.

### Advantages

- handles uneven hardware capacity well
- lets larger or stronger servers take more load
- useful during staged rollouts or capacity planning

### Disadvantages

- requires capacity planning and configuration
- can still be imperfect if server health changes over time

### Best for

- heterogeneous infrastructure
- deployments with different size instances
- gradual traffic migration between servers

---

## 11. Geographical Algorithms

Geographical load balancing routes traffic based on the client's location or region.

Examples:

- route users in Europe to European servers
- route users in the US to US regions
- fail over to another region if a region is unhealthy

### Advantages

- reduces latency for users
- improves regional performance
- helps with disaster recovery and failover

### Disadvantages

- more complex deployment and data replication requirements
- cross-region traffic can be expensive
- data consistency across regions needs planning

### Best for

- global applications
- multi-region deployments
- high-latency-sensitive systems

---

## 12. Consistent Hashing

Consistent hashing is a technique where requests are mapped onto a ring or hashed key space so that small changes in the server set do not cause large redistribution of traffic.

This is especially useful when servers are added or removed frequently.

### Advantages

- minimal remapping when nodes are added or removed
- good for distributed caching and sharded systems
- reduces churn in large-scale systems

### Disadvantages

- more complex to design and reason about
- not always necessary for small or medium apps
- needs careful handling of key distribution and replicas

### Best for

- distributed caches
- sharded systems
- storage systems and key-based partitioning

---

## 13. Load Balancer Health and Failover

A load balancer is not just routing; it is also protecting the system from unhealthy nodes.

Typical health checks include:

- checking if a server responds successfully
- verifying app health endpoint
- watching response latency
- removing failed nodes automatically

If a server becomes unhealthy:

1. the load balancer stops sending new traffic to it
2. existing connections may be drained
3. healthy servers continue serving requests
4. the unhealthy node can recover or be restarted

This is a core part of resiliency.

---

## 14. Layer 4 vs Layer 7 Load Balancing

Load balancers can operate at different layers of the network stack.

### Layer 4 (L4)

Routes traffic based on TCP/UDP information such as IP and port.

- faster
- simpler
- good for raw traffic routing
- does not inspect application content

### Layer 7 (L7)

Routes traffic based on application-level data such as HTTP headers, paths, cookies, or hostnames.

- more intelligent routing
- can route by URL or API route
- useful for microservices and API gateways
- more expensive but more flexible

### Example

A Layer 7 load balancer can route:

- `/api/*` to the API servers
- `/images/*` to static asset servers
- `/admin/*` to a separate admin service

---

## 15. Load Balancing in Practice

A typical production setup might use:

- a public load balancer in front of app servers
- multiple application instances running behind it
- health checks to identify broken nodes
- session affinity only when needed
- different routing rules for API and static traffic

For example:

```text
Internet
  |
  v
Load Balancer
  |
  +--> App Server 1
  +--> App Server 2
  +--> App Server 3
```

This gives better uptime and lets the system serve more concurrent users.

---

## 16. Summary

Load balancing is a critical component in scaling a system. It helps us:

- distribute traffic across multiple servers
- reduce overload on any single instance
- improve fault tolerance and uptime
- scale more effectively as demand grows

Common algorithms include:

- Round Robin
- Least Connections
- Least Response Time
- IP Hash
- Weighted Algorithms
- Geographical Algorithms
- Consistent Hashing

Choosing the right algorithm depends on:

- server capacity
- traffic type
- session requirements
- regional needs
- application behavior

A system with good load balancing is much more resilient and scalable than a single-server system.

---

## Next Topic

The next step is to move from traffic distribution into broader scaling patterns, such as:

- multiple application servers behind a load balancer
- database replication and failover
- caching layers
- CDN integration
- production infrastructure and autoscaling
