# API Design

## Overview

An **Application Programming Interface (API)** is a defined way for software components to communicate with each other. It specifies which operations are available, how callers invoke them, and what they receive in return.

APIs sit at important boundaries in a system: between a client and server, between frontend and backend services, or between independent microservices.

> **The best API is one that developers can use without even reading the documentation.**

A strong API should therefore be predictable, discoverable, consistent, and difficult to misuse.

---

## 1. API Contract

An API contract defines:

- **What requests can be made** — the available operations and resources
- **How to make them** — paths, methods, parameters, headers, and message format
- **What responses to expect** — status codes, data structures, errors, and metadata

A small REST contract might look like this:

```http
GET /users/123
Authorization: Bearer <token>
Accept: application/json

200 OK
{
  "id": "123",
  "name": "Ada Lovelace",
  "email": "ada@example.com"
}
```

The contract may be documented with OpenAPI, JSON Schema, Protocol Buffers, GraphQL schemas, or another formal description language. These definitions allow developers and tools to validate, generate, test, and document API interactions.

---

## 2. Key API Characteristics

### Abstraction mechanism

An API hides implementation details behind a stable interface. A caller does not need to know which database is used, how data is cached, or how internal services work.

This abstraction:

- reduces coupling between components
- enables internal implementation changes
- gives the provider control over business rules
- creates a clear boundary for validation and authorization

A good abstraction exposes useful capabilities rather than leaking internal tables or objects.

### Server boundary

The API is the boundary between the API provider and its consumers. Crossing that boundary usually means passing through a network, so the design must account for:

- latency
- partial failures
- serialization
- authentication
- authorization
- retries
- backward compatibility
- observability

The API also makes an architectural boundary explicit. It becomes a versioned contract that should be changed carefully and maintained independently from internal implementation.

---

## 3. APIs in System Architecture

APIs play several roles:

- **External access:** enable browsers, mobile applications, partners, and third-party developers to use a system
- **Internal communication:** connect frontend services with backend services
- **Service integration:** allow independently deployed services to exchange data
- **Gateway behavior:** define how requests are transformed, routed, secured, and limited
- **System decoupling:** let teams develop and deploy components with fewer shared dependencies

A typical architecture might contain browser and mobile clients, an API gateway, application services, and databases:

```text
Browser / Mobile / Partner
           |
           v
      API Gateway
     /      |      \
    v       v       v
 User    Orders   Billing
 Service Service  Service
    |       |        |
    v       v        v
 Database  Database  Database
```

The gateway may handle TLS termination, authentication, routing, rate limits, request validation, caching, and tracing. Individual services expose APIs for direct inter-service communication.

---

## 4. Microservices

**Microservices** are a system design approach in which an application is divided into small, independently deployable services organized around business capabilities.

Examples may include:

- account management
- catalog search
- orders
- payments
- notifications
- inventory

A microservice owns a focused responsibility and commonly owns its own data. Services communicate through APIs or asynchronous events.

Benefits include:

- independent deployment
- smaller, focused codebases
- technology choices appropriate to each service
- isolation of some failures
- independent scaling

Costs include:

- distributed-system failures
- network and serialization overhead
- observability complexity
- service discovery
- duplicated data and eventual consistency
- increased deployment and operational work

A microservice architecture requires strong API and contract design. Breaking a public service contract may break every downstream consumer.

---

## 5. Core API Styles

The three most common API styles are REST, GraphQL, and gRPC.

| Style | Main model | Transport | Common use | Strengths | Tradeoffs |
|---|---|---|---|---|---|
| REST | Resources and representations | HTTP/HTTPS | Public APIs, web and mobile backends | Familiar, cacheable, broadly supported | Can over-fetch or under-fetch data and require multiple requests |
| GraphQL | Graph of typed operations | Usually HTTP | Frontends with varied data needs | Precise data fetching, one endpoint, strongly typed schema | Caching and complex query-cost control require care |
| gRPC | Typed remote procedures | HTTP/2 | Internal microservice communication | Fast binary protocol, streaming, code generation | Less convenient in browsers and less human-readable |

There is no universally best style. Many systems use more than one: REST or GraphQL for external clients and gRPC or messaging for internal services.

---

## 6. REST

**Representational State Transfer (REST)** is an architectural style based on resources, standard methods, HTTP semantics, and representations such as JSON.

A REST API commonly represents domain resources in URLs:

```text
/users
/users/{id}
/users/{id}/orders
/orders/{id}
```

### HTTP methods

| Method | Intended purpose | Typical example | Safe | Idempotent |
|---|---|---|---|---|
| `GET` | Retrieve data | `GET /users/123` | Yes | Yes |
| `POST` | Create or trigger an operation | `POST /orders` | No | No |
| `PUT` | Replace a resource | `PUT /users/123` | No | Yes |
| `PATCH` | Apply a partial update | `PATCH /users/123` | No | Usually yes |
| `DELETE` | Remove a resource | `DELETE /users/123` | No | Yes |

- **Safe** means the request should not alter server state.
- **Idempotent** means repeated identical requests should have the same intended result.

REST APIs should use status codes consistently:

- `200` — successful request
- `201` — resource created
- `204` — successful request with no response body
- `400` — invalid request
- `401` — not authenticated
- `403` — authenticated but not authorized
- `404` — resource not found
- `409` — conflict
- `429` — rate limited
- `5xx` — server-side failure

REST is commonly used for public HTTP APIs, web applications, mobile backends, partner integrations, and CRUD-oriented systems. HTTP caching, proxies, CDNs, and conventional tooling are major advantages.

### REST example

```http
POST /orders
Content-Type: application/json

{
  "userId": "123",
  "items": ["item-1", "item-2"]
}
```

```http
201 Created
Location: /orders/789

{
  "id": "789",
  "status": "created"
}
```

---

## 7. GraphQL

**GraphQL** is a query language and runtime for APIs. A client sends a typed operation and specifies the fields it wants. It normally uses a single HTTP endpoint.

GraphQL supports:

- **queries** for reads
- **mutations** for writes
- **subscriptions** for real-time updates
- typed objects and relationships

Example query:

```graphql
query {
  user(id: "123") {
    name
    email
    orders {
      id
      status
    }
  }
}
```

Example schema:

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  orders: [Order!]!
}

type Query {
  user(id: ID!): User
}
```

GraphQL is useful when different clients require different fields or nested related data. It reduces over-fetching and under-fetching and can avoid many client-specific endpoints.

However, it requires deliberate design for:

- authorization on fields or resolvers
- query depth and complexity limits
- pagination
- caching
- N+1 database queries
- operationally expensive or abusive queries

GraphQL is most commonly used in frontend and backend-for-frontend architectures where one API is consumed by web and mobile applications with different data needs.

---

## 8. gRPC

**gRPC** is a high-performance RPC framework. It uses Protocol Buffers to define schemas and HTTP/2 for transport.

Example service definition:

```proto
syntax = "proto3";

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
}

message GetUserRequest {
  string id = 1;
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
}
```

The generated client and server code provide typed methods that feel like local procedure calls.

gRPC supports:

- unary requests
- server streaming
- client streaming
- bidirectional streaming
- generated clients and validation
- efficient binary serialization

It is mostly used for internal microservice communication, especially in high-throughput, low-latency environments. It is less convenient for public APIs because browser support and human-readable debugging are weaker than for REST.

---

## 9. How to Choose an API Style

Consider:

- consumer type: browser, mobile, partner, or internal service
- data shape: fixed resources or varied nested queries
- performance and payload size
- streaming requirements
- existing ecosystem and tooling
- security boundaries
- team familiarity
- caching needs
- backward compatibility

A common pattern is:

- REST for simple public resource APIs
- GraphQL for flexible frontend data access
- gRPC for strongly typed internal service-to-service communication

The style should solve the requirements; it should not be selected solely because it is fashionable.

---

## 10. HTTP Methods and Query Language

### HTTP methods

HTTP methods express intent. They should be used consistently rather than sending every operation through `POST /doSomething`.

Good REST endpoint designs are resource-oriented:

```text
GET    /orders
POST   /orders
GET    /orders/123
PATCH  /orders/123
DELETE /orders/123
```

### Query parameters

Query parameters are commonly used for filtering, sorting, fields, pagination, and search:

```text
GET /orders?status=paid&sort=-createdAt&limit=20&offset=40
```

Names and semantics should be consistent across endpoints. For example, if `limit` means page size in one endpoint, it should not mean a maximum total count in another.

### GraphQL as a query language

GraphQL is not a general HTTP verb replacement. It is a typed query language whose schema exposes fields and operations. It gives clients control over the requested shape while preserving a server-defined contract.

A GraphQL API also needs pagination conventions, especially for large collections:

```graphql
query {
  orders(first: 20, after: "cursor") {
    nodes {
      id
      status
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

Cursor pagination is often better than offset pagination for changing or very large datasets.

---

## 11. Four Key Design Principles

### Consistency

APIs should use repeatable conventions for:

- naming
- URL structure
- status codes
- authentication
- pagination
- filtering and sorting
- timestamps and identifiers
- error formats
- success envelopes

Consistency makes an API predictable. Developers should not have to learn a separate pattern for every endpoint.

Example of a consistent error response:

```json
{
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Order 789 was not found.",
    "requestId": "req_01JX"
  }
}
```

Avoid inconsistent fields such as `orderID`, `order_id`, and `orderId` for the same concept.

### Security

Security must be built into the API design, not added only as a gateway rule.

Key concerns include:

- authentication to establish identity
- authorization to check permissions
- least-privilege access
- input validation
- TLS/HTTPS
- secret and key management
- rate limits and quotas
- protection against injection attacks
- data minimization
- audit logging
- safe handling of credentials and personal data

Authorization should enforce that a caller can perform the requested operation on the requested object. Authentication alone is insufficient.

Example:

```text
A user may read /orders/789 only if they own it or have an administrator role.
```

### Simplicity

A simple API has:

- clear names
- few required concepts
- intuitive resource or operation boundaries
- stable, descriptive data models
- predictable errors
- minimal special cases
- sensible defaults

Simplify by starting with the smallest useful contract and adding fields or operations only when real use cases require them. Avoid exposing internal implementation details, such as database IDs or table names, unless they are intentionally part of the public contract.

Good API design is often more about removing unnecessary complexity than adding features.

### Performance

Performance should include both latency and efficient resource use.

Consider:

- payload size
- response compression
- database query efficiency
- pagination
- caching and cache invalidation
- connection reuse
- bulk operations
- timeouts
- cancellation
- backpressure
- asynchronous processing for expensive work

Support a pending operation when work cannot complete synchronously:

```http
202 Accepted
Location: /operations/op_456
```

The client can then poll or subscribe for status. Never force it to hold an HTTP connection open during a long-running operation.

Measure p50, p95, and p99 latency, throughput, error rate, and saturation rather than relying only on averages.

---

## 12. API Protocols

An **application protocol** is the rules and conventions used to exchange messages. Protocol choice affects message format, transport behavior, tooling, caching, and failure handling.

### HTTP/1.1

HTTP/1.1 is broad, mature, and commonly used for REST and GraphQL. It has excellent support from browsers, proxies, caches, and APIs. Request multiplexing is limited compared with HTTP/2.

### HTTP/2

HTTP/2 supports multiplexed streams, header compression, and efficient reuse of connections. It is used by gRPC and can improve many modern HTTP workloads.

### WebSockets

WebSockets provide persistent bidirectional communication after an HTTP upgrade. They are useful for chat, live collaboration, games, and real-time dashboards. They require separate protocol and reconnection design for messages, ordering, heartbeats, and authentication.

### Server-Sent Events

Server-Sent Events provide a simple one-way stream from server to client, usually for progress updates or live event feeds.

### Messaging protocols

AMQP and MQTT, and systems such as Kafka, are commonly used for event-driven communication rather than request/response APIs. They support topics, queues, acknowledgments, and asynchronous delivery.

---

## 13. How Protocols Influence API Design

Protocol decisions affect the API contract:

- **HTTP** enables methods, resource paths, status codes, headers, caching, redirects, and web tooling
- **HTTP/2** supports efficient connection reuse and gRPC streaming
- **WebSockets** shift APIs toward persistent events and application-level message schemas
- **GraphQL** is commonly transported over HTTP but has a separate operation and schema model
- **gRPC** favors typed procedures and code-generated contracts over readable JSON resources
- **Async messaging** emphasizes events, delivery semantics, retries, duplicate handling, and consumer coordination

For every protocol, define:

- request and response formats
- status or acknowledgment semantics
- retries and idempotency
- timeout behavior
- ordering guarantees
- authentication and authorization
- observability and tracing
- versioning
- backward compatibility

Protocol support should match the actual consumer. A protocol optimized for internal microservices may be unsuitable for browser-based public access, and a browser-friendly REST API may not provide the streaming or efficiency needs of high-volume service communication.

---

## 14. API Design Process

A useful API design process follows these stages:

### 1. Identify core use cases

Start with the users and jobs to be done.

Examples:

- A mobile app needs to show a user's account and recent orders.
- A partner needs to submit and track an order.
- A billing service needs to charge a payment method and receive status updates.

Write user stories or API scenarios before defining endpoints.

### 2. Identify consumers

Determine whether consumers are:

- public or internal
- human developers or other services
- browsers, mobile apps, servers, or batch systems
- low or high trust
- strongly or loosely coupled

### 3. Define scope

Decide:

- which resources or operations are required
- which actions are out of scope
- ownership of each resource
- which service is the source of truth
- expected schema and lifecycle
- public versus internal contract

### 4. Establish performance requirements

Specify measurable targets for:

- latency
- throughput
- payload size
- availability
- concurrency
- peak traffic
- regional behavior
- acceptable degradation

Example:

```text
The order status read API should return a p95 latency below 150 ms
for up to 2,000 requests per second across all regions.
```

### 5. Identify security constraints

Define:

- authentication mechanism
- authorization model
- sensitive fields
- tenant isolation
- audit requirements
- compliance needs
- allowed data residency
- rate limits

### 6. Model the contract

Create resource models or typed operations, including:

- fields and types
- required versus optional values
- valid ranges
- relationships
- stable identifiers
- timestamps and time zones
- pagination
- error representations

The contract should describe behavior, not implementation.

### 7. Prototype and validate

Use API specifications and mock servers to verify that the contract supports the scenarios. Review it with consumer developers and domain experts before implementation.

Tests should cover:

- successful requests
- invalid input
- permission failures
- missing resources
- conflicts
- retries
- pagination
- partial failures
- performance targets

### 8. Build, test, observe, and document

The implementation should include:

- contract tests
- integration tests
- load tests
- metrics and tracing
- structured logs
- API documentation
- examples and SDKs when useful
- deprecation and migration guidance

### 9. Iterate safely

Observe real usage and evolve the API based on evidence. Add backward-compatible fields or operations when possible. Avoid breaking consumers and provide migration paths when breaking changes are necessary.

---

## 15. API Design Approaches

### Top-down design

Top-down design starts from consumer needs and the desired external contract. Teams define the API first, independently of existing implementation details.

Advantages:

- aligns directly with use cases
- hides implementation complexity
- provides a stable consumer-focused interface
- works well for new products and public APIs

Risks:

- may be disconnected from legacy systems
- can underestimate implementation complexity
- may create contracts that are difficult to realize or maintain

### Bottom-up design

Bottom-up design starts with existing services, databases, or reusable capabilities and exposes an interface over them.

Advantages:

- faster for existing systems
- reflects operational capabilities
- reuses proven implementation components

Risks:

- leaks internal structure
- creates inconsistent or overly granular APIs
- exposes implementation details through the contract
- may produce APIs that are not easy for consumers to use

### Contract-first design

Contract-first design creates and agrees on a formal API contract before implementation. Tools such as OpenAPI, GraphQL schemas, or Protocol Buffers can define the contract.

Advantages:

- enables parallel client and server development
- supports mocks and generated code
- provides executable documentation
- exposes breaking changes during review
- improves testing through shared schemas

Risks:

- requires discipline to keep documentation and implementation synchronized
- may delay validation of implementation feasibility

In practice, contract-first and top-down approaches are often combined for external APIs. Internal APIs may use contract-first design while evolving gradually with existing services.

---

## 16. API Versioning and Compatibility

API versions separate contracts that may not be able to evolve safely at the same time.

Common approaches include:

- URL versions, such as `/v1/users`
- header or media-type versions
- GraphQL field deprecation
- protocol buffer field evolution

Prefer backward-compatible changes whenever possible:

- add optional fields
- add endpoints or operations
- add enum values carefully
- avoid changing the meaning of existing fields
- avoid renaming or removing fields abruptly
- preserve existing status-code behavior

For breaking changes:

- document the reason and impact
- provide overlap between old and new versions
- communicate timelines to consumers
- instrument usage of deprecated contracts
- remove only after consumers migrate

---

## 17. Idempotency and Retries

Distributed systems lose responses, experience timeouts, and repeat requests.

An operation is **idempotent** if repeating it produces the same intended result. `DELETE` and `PUT` are commonly designed this way, while `POST` may create duplicate resources when retried.

Use an idempotency key for operations that must be safely retried:

```http
POST /payments
Idempotency-Key: payment-2026-11-24-a91

{
  "orderId": "789",
  "amount": 5000,
  "currency": "USD"
}
```

The server stores the key and result, allowing repeated requests to return the original outcome instead of creating another payment.

Retries should use:

- retry limits
- exponential backoff
- jitter
- retry only for transient failures
- explicit timeouts
- circuit breakers when appropriate

---

## 18. Error Design

A useful error response provides enough information to diagnose and handle a problem without exposing secrets or internal implementation.

Include:

- machine-readable error code
- human-readable message
- relevant field or resource identifier
- request or trace ID
- documentation or help link when appropriate
- `retryable` status only if it is accurate

Avoid exposing:

- database statements
- stack traces
- server paths
- secrets or credentials
- sensitive customer data

Errors should support both developer debugging and stable client behavior.

---

## 19. API Lifecycle Management

API lifecycle management covers the API from planning through retirement.

### Plan

- identify use cases and consumers
- set scope, security, and performance requirements
- align ownership and funding
- assess build versus buy

### Design

- create a consumer-oriented contract
- choose API style and protocol
- define versioning and compatibility rules
- review security and privacy

### Implement

- validate requests
- enforce authorization
- implement typed models and stable errors
- add observability and documentation

### Test

- unit and integration tests
- contract tests
- security tests
- load tests
- backward compatibility checks

### Publish

- deploy to controlled environments
- publish API documentation
- provide onboarding examples and SDKs
- configure rate limits and monitoring

### Operate

- monitor latency, errors, throughput, and saturation
- trace requests across services
- manage credentials and keys
- respond to incidents
- track quota and abuse

### Govern

- review API consistency and standards
- identify duplicate or obsolete APIs
- manage access and ownership
- control sensitive data exposure
- maintain lifecycle metadata and dependencies

### Deprecate and retire

- announce deprecation clearly
- provide migration guidance
- preserve a supported overlap period
- measure remaining usage
- remove infrastructure only after the contract is no longer used

Lifecycle management treats APIs as products: they have owners, users, service levels, documentation, deprecation policies, and operational costs.

---

## 20. Summary

A good API is a stable contract that makes system boundaries clear and safe.

Strong API design starts with real use cases and requirements, then combines:

- the right API style and protocol
- a clear and consistent contract
- security and authorization
- predictable behavior
- adequate performance
- compatibility and lifecycle planning

REST, GraphQL, and gRPC solve different problems. REST is natural for HTTP resources, GraphQL is useful for flexible client-defined reads, and gRPC is strong for typed, high-performance internal communication.

The goal is not simply to expose endpoints or methods. The goal is to create an interface that consumers can understand, trust, and use safely for many years.
