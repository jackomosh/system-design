# Single Server to Scaling

## Step 1: Building a Single Server Setup

This document explains the simplest production architecture: everything runs on a single server. The goal is to understand how requests travel from clients to the server and back before introducing the scaling patterns that come next.

---

## 1. The Starting Point

Imagine we have a small system with:

- One application server
- One database
- One cache
- One domain: `app.demo.com`
- One public IP address
- A small user base using either a mobile app or a browser on a desktop

At this stage, the system is intentionally simple:

- The web app runs on the same server
- The database runs on the same server
- The cache runs on the same server
- All business logic, storage, and serving happen in one place

This is a valid startup architecture for a small product, but it will eventually become a bottleneck as traffic grows.

---

## 2. The System Overview

A basic single-server setup looks like this:

- Client (mobile app or browser)
- Domain name `app.demo.com`
- DNS lookup to resolve the domain to a public IP
- Internet connection to the server
- HTTP or HTTPS request sent to the server
- App server handles the request
- Database fetches or writes data
- Cache stores data for faster access
- Response goes back to the client

This is the foundation for understanding scaling later.

---

## 3. How the URL Resolves

When a user opens a browser or mobile app and accesses:

`https://app.demo.com`

the following happens:

1. The client asks a DNS resolver: "What IP address belongs to `app.demo.com`?"
2. DNS returns the server's public IP address
3. The client opens a connection to that IP on port 80 or 443
4. The browser or app sends an HTTP or HTTPS request

For example, a request may look like:

```http
GET /login HTTP/1.1
Host: app.demo.com
User-Agent: Mozilla/5.0
Accept: */*
```

If HTTPS is enabled, the connection is encrypted and the browser negotiates TLS before sending the actual HTTP request.

---

## 4. How the Request Reaches the Server

At the network layer, the request travels over the public internet to the server's IP address.

The server is listening for incoming traffic on a port, usually:

- 80 for HTTP
- 443 for HTTPS

A server process such as Nginx, Apache, or application runtime listens for these requests.

The request flow is:

```text
Client -> DNS -> Public IP -> Server -> App -> Database/Cache -> Response -> Client
```

This is the most important mental model:

- The user does not connect directly to the application code
- The user connects to the server over the network
- The server decides how to handle the request

---

## 5. What Happens on the Server

Once the request reaches the machine, the server does a few things:

1. Accepts the TCP connection
2. Performs TLS termination if using HTTPS
3. Routes the request to the correct app
4. Reads URL path, headers, query parameters, and request body
5. Calls business logic
6. May read from cache
7. May query the database
8. Builds the response
9. Sends the result back to the client

A simplified flow looks like this:

```text
Incoming HTTP request
        |
        v
Load balancer / reverse proxy (optional in single server)
        |
        v
Web/App server
        |
        +--> Cache
        |
        +--> Database
        |
        v
Response
```

At the single-server stage, the reverse proxy and app code may all live on the same machine, and that is okay for a small user base.

---

## 6. The Role of the Database

If the request needs persisted data, the application server talks to the database.

Examples:

- Fetch user profile
- Save login session
- Read posts or orders
- Write a transaction

Typical database operations:

```text
App -> SQL query -> Database -> Result -> App -> Response
```

In a single-server setup, both the app and the database process run on the same machine, which is simple but means:

- Database CPU and memory compete with app CPU and memory
- Disk I/O is shared
- A slow query can affect the entire system
- A crash of the machine brings down the app, cache, and database together

---

## 7. The Role of the Cache

Many systems also run a cache like Redis or Memcached on the same server.

Caching is useful when data is:

- frequently requested
- not changing often
- expensive to compute or query

Examples:

- user profile
- recent product listings
- dashboard metrics
- session tokens

How the app uses it:

```text
Request arrives
    -> check cache
    -> if cache hit: return quickly
    -> if cache miss: query DB and store result in cache
```

The cache reduces database load and improves latency dramatically.

In a single-server architecture, cache is still helpful, but it is not a replacement for good database design or scaling strategy.

---

## 8. Example Request Flow

Here is a concrete example for a user logging in via the browser.

### Example: `https://app.demo.com/login`

1. The user enters the URL in the browser.
2. DNS resolves `app.demo.com` to the server's IP address.
3. The browser sends an HTTPS request to the server.
4. The web server receives the request.
5. The app route identifies the login endpoint.
6. The app validates the credentials.
7. It checks if the user is already cached.
8. If not, it reads or writes the needed data in the database.
9. It creates a session or token.
10. It sends back a success response such as HTML, JSON, or redirect.
11. The browser renders the final page or mobile app reads the response.

Response example:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "userId": 123,
  "name": "Jane",
  "status": "logged_in"
}
```

---

## 9. What the Client Sees

From the perspective of the client, the experience is simple:

- User types domain name
- Request is sent to the server over the internet
- Server responds with data
- Client renders it

This is a normal and straightforward interaction for a small user base.

---

## 10. Why This Works at First

A single-server setup is often the best first version because it is:

- cheap to run
- simple to deploy
- easy to debug
- quick to build
- good enough for a small user base

A single machine can support a surprisingly large number of users if the workload is light and traffic is modest.

---

## 11. Limitations of a Single Server

This design starts to break down as traffic, data, and complexity grow.

### Main bottlenecks

- CPU saturation: too many requests for one machine
- Memory pressure: app and database compete for memory
- Disk I/O bottleneck: DB and app read/write from the same disk
- Single point of failure: if the machine goes down, everything goes down
- Hard to scale horizontally: you cannot simply add more app instances without changing architecture
- Limited reliability: no redundancy, no failover, no isolation

A single server is ideal for learning and early adoption, but not for high availability or large-scale production.

---

## 12. What the Next Scaling Step Looks Like

Once the app grows, the next step is not always "rewrite everything". Usually the evolution looks like:

1. Move the app and database to separate machines if needed
2. Add a cache in front of the database
3. Add a load balancer and multiple app servers
4. Move to managed databases or replication
5. Add queues for background jobs
6. Add CDN for static assets
7. Add monitoring, autoscaling, and production infra

The key idea is that the single server is the baseline, and scaling happens by separating concerns and distributing traffic.

---

## 13. Summary

In a single-server setup, the client connects to `app.demo.com`, DNS resolves the domain to the server's public IP, and the request is sent over the internet to the machine hosting the app, database, and cache.

The server handles the request, reads or writes data, and sends a response back to the client.

This simple architecture is perfect for small systems, but it has major bottlenecks as users and complexity grow. The next phase of scaling will focus on distributing workloads, separating components, and improving availability.

---

## Next Topic

The next step is to understand how this single-server design evolves into a more scalable architecture by separating concerns and adding layers like:

- caching
- load balancing
- multiple application instances
- database replication
- CDNs
- production infrastructure
