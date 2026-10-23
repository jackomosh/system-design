# Database Design and Selection

## Overview

Databases are the foundation of most production systems. They store the state of the application, support reads and writes, enforce consistency, and determine how data is queried and managed at scale.

The right database choice depends on the application's workload, data model, consistency needs, operational complexity, and growth expectations.

---

## 1. Why Database Choice Matters

A database is not just storage. It defines:

- how data is structured
- how quickly data can be read or written
- whether transactions are supported
- how the system behaves under concurrency
- how hard it becomes to scale
- how resilient the system is to failure

When building a system, the first major architectural decision is often this:

- Do we need a relational database?
- Or do we need a NoSQL database?

The answer depends on the workload.

---

## 2. Relational Databases

Relational databases organize data into tables with rows and columns and use SQL to query and manipulate data.

Examples include:

- MySQL
- PostgreSQL
- SQLite
- Oracle
- SQL Server

### Key Characteristics

- Strong schema definition
- Structured data model
- ACID transactions
- Foreign keys and joins
- Mature tooling and ecosystem
- Excellent for complex querying and consistency

### Common Use Cases

Relational databases are a good fit when:

- the data has clear relationships
- transactions are important
- we need structured reporting or analytics
- we need strong consistency
- a predefined schema is useful
- data integrity matters a lot

### Examples of workloads

- user accounts
- orders and payments
- inventory management
- financial systems
- internal business apps
- applications requiring joins across multiple tables

### Advantages

- Strong consistency guarantees
- Mature and well-understood model
- Excellent transaction support
- Great for complex queries and reporting
- Strong data integrity
- Works well with conventional application patterns

### Disadvantages

- Scaling horizontally can be harder
- Schema changes can be expensive
- Can become slow under extreme write-heavy workloads
- Vertical scaling may be required at some point
- More operational complexity at large scale

### Best When

Use a relational database when the system needs:

- reliable transactions
- structured relationships
- accurate reporting
- strong consistency
- predictable query behavior

---

## 3. NoSQL Databases

NoSQL databases are designed for flexible data models and high scalability. They are often used when the workload is large, distributed, and fast-changing.

Examples include:

- MongoDB
- Cassandra
- Redis
- DynamoDB
- Couchbase
- Elasticsearch

### Key Characteristics

- Flexible schema or schema-less design
- Better horizontal scaling
- Optimized for large-scale reads/writes
- Usually designed for distributed systems
- Different consistency models depending on the system

### Common Use Cases

NoSQL databases are commonly used when:

- the data is semi-structured or unstructured
- the workload is high-volume and read-heavy
- the system needs fast horizontal scaling
- data is distributed across many servers
- the schema changes often

### Examples of workloads

- event logs
- product catalogs
- user activity feeds
- session data
- time-series or telemetry data
- large-scale social or content platforms

### Advantages

- Flexible schemas
- Easy horizontal scaling
- High throughput for large workloads
- Better fit for distributed and large-scale systems
- Often simpler to handle massive unstructured data

### Disadvantages

- Weaker or more complex consistency guarantees
- Less mature for complex relational querying
- Joins may be limited or unavailable
- Harder to reason about cross-document consistency
- Operational complexity can be high in distributed systems

### Best When

Use a NoSQL database when:

- scale requirements are very high
- the schema changes frequently
- the workload is mostly key-value, document, or event-driven
- horizontal scaling is more important than relational modeling

---

## 4. Relational vs NoSQL: Side-by-Side

### Relational databases

- Good for structured, relational data
- Strong ACID guarantees
- Query with SQL and joins
- Better for reporting and analytics
- Usually scale vertically first

### NoSQL databases

- Good for flexible, high-scale, distributed workloads
- Optimized for speed and scalability
- Often sacrifices some transactional guarantees
- Better for large-scale read/write patterns
- Usually scale horizontally first

---

## 5. When Should We Use Which?

### Use a relational database when:

- the system needs transactions
- the data has strong relationships
- integrity matters
- the app requires complex queries
- reporting and analytics are important
- we need predictable consistency

Examples:

- banking systems
- e-commerce order processing
- inventory systems
- SaaS apps with business logic tied to relational structure

### Use a NoSQL database when:

- the system is large and distributed
- the schema may evolve frequently
- latency is critical at scale
- the workload is write-heavy or event-driven
- the data fits key-value or document models
- we need better horizontal scale

Examples:

- social networks
- recommendation engines
- activity streams
- telemetry systems
- real-time analytics

---

## 6. Hybrid Approaches

In practice, many production systems use both.

Examples:

- PostgreSQL for core transactional data
- Redis for caching and session data
- MongoDB for flexible metadata or content
- Elasticsearch for search

This is common because different systems have different needs. A system does not have to choose only one database forever.

---

## 7. Important Tradeoff: Consistency vs Scalability

One of the central ideas in database design is the tradeoff between:

- consistency
- availability
- partition tolerance

This is often discussed through the CAP theorem.

Relational databases usually prioritize consistency and correctness.
NoSQL systems may prioritize availability and scalability, sometimes with weaker consistency guarantees.

This creates a key design question:

- Do we need strict correctness?
- Or do we need maximum throughput and scale?

The answer influences the database decision.

---

## 8. Database and Request Flow in a Simple System

When a request needs data:

```text
Client -> Web Server -> App Logic -> Database -> Response
```

In a more optimized architecture:

```text
Client -> Cache -> App -> Database
```

This is where caching and database selection become deeply connected: the faster and more efficient the database, the less pressure the app experiences.

---

## 9. Summary

The database decision is one of the most important system design choices because it affects:

- performance
- complexity
- scaling strategy
- operational cost
- correctness

Relational databases are ideal for structured, transactional, and highly consistent systems.
NoSQL databases are ideal for large-scale, flexible, distributed systems where horizontal scalability matters more than strict relational modeling.

The best system is not always the most modern database; it is the one that best fits the application requirements.

---

## 10. Scaling Perspective

Once the database choice is made, the next step is to optimize the system by addressing scale:

- vertical scaling: increase resources on one machine
- horizontal scaling: add more machines and distribute traffic
- separation of application and data layers
- adding cache layers
- load balancing traffic across servers

This leads into the next major design topic: vertical vs horizontal scaling.

---

## Next Topic

The next section explores the difference between vertical scaling and horizontal scaling, and when each is suitable as a system grows.
