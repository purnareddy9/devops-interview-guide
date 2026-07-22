# 🏛️ System Design — DevOps Interview Guide

> [← SRE](./SRE.md) | [Main Index](./README.md) | [Behavioral →](./Behavioral.md)

---

## Table of Contents

1. [System Design Framework](#system-design-framework)
2. [Scalability Fundamentals](#scalability-fundamentals)
3. [Availability & Reliability](#availability--reliability)
4. [Microservices Patterns](#microservices-patterns)
5. [Database Design](#database-design)
6. [Caching Strategies](#caching-strategies)
7. [Message Queues](#message-queues)
8. [Design Case Studies](#design-case-studies)
9. [Interview Questions](#interview-questions)
10. [Scenario-Based Design Questions](#scenario-based-design-questions)

---

## System Design Framework

### Interview Approach (45 minutes)

```
1. Clarify Requirements (5 min)
   - Functional: what must the system do?
   - Non-functional: scale, latency, availability, consistency?
   - Constraints: team size, budget, tech stack?
   - Scope: what's in/out for today's design?

2. Estimate Scale (5 min)
   - Users: DAU, peak concurrent
   - Data: write rate, read rate, storage growth
   - Bandwidth: inbound, outbound
   - QPS: queries per second at peak

3. High-Level Architecture (10 min)
   - Draw major components
   - Data flow between components
   - Client → LB → Services → DB

4. Deep Dive (20 min)
   - Focus on 2-3 critical components
   - Data model, APIs, algorithms
   - Trade-offs: consistency vs availability
   - Bottlenecks and how to address them

5. Identify Bottlenecks & Scale (5 min)
   - Single points of failure
   - How does it scale to 10x, 100x?
   - Failure scenarios
```

### Estimation Numbers to Know

```
1 KB = 1,000 bytes
1 MB = 10^6 bytes
1 GB = 10^9 bytes
1 TB = 10^12 bytes

Time units:
1 ms  = 10^-3 seconds
1 μs  = 10^-6 seconds
1 ns  = 10^-9 seconds

Latency reference points:
L1 cache ref:        0.5 ns
L2 cache ref:        7 ns
RAM access:          100 ns
SSD random read:     16 μs
Network (same DC):   0.5 ms
Network (cross-AZ):  1-2 ms
Network (CA→EU):     150 ms
HDD seek:            10 ms

Traffic estimation:
1M users × 5 requests/day / 86,400 sec/day ≈ 58 QPS (average)
Peak is typically 2-10x average → ~580 QPS peak
```

---

## Scalability Fundamentals

### Horizontal vs Vertical Scaling

```
Vertical Scaling (Scale Up):
- Bigger machine (more CPU, RAM)
- Simple to implement
- Limited ceiling
- Single point of failure
- Good for: databases (early stage), simple architectures

Horizontal Scaling (Scale Out):
- More machines
- Requires load balancing
- Theoretically unlimited
- Resilient (no SPOF)
- Good for: stateless services, read replicas

Hybrid: vertical until pain, then horizontal
```

### Load Balancing

```
L4 Load Balancer (TCP/IP):
  + Faster, less overhead
  + Works for any TCP protocol
  - No application-layer routing
  Use: high-performance TCP, database proxying

L7 Load Balancer (HTTP):
  + Content-based routing (URL, headers, cookies)
  + SSL termination
  + Health checks at app level
  + Session affinity (sticky sessions)
  - Slower than L4 (inspects content)
  Use: web applications, microservices, APIs

Algorithms:
- Round Robin: even distribution
- Weighted RR: proportional to capacity
- Least Connections: route to least busy server
- IP Hash: sticky routing by client IP
- Consistent Hashing: used for cache routing, minimizes remapping
```

### Consistent Hashing

```
Problem: Adding/removing servers in simple hash causes mass remapping
  server = hash(key) % num_servers
  Add server: almost all keys map to different server → cache miss storm

Solution: Consistent hashing
- Hash servers onto a ring: hash(server_id) → position on 0..2^32 ring
- Hash keys onto the same ring
- Key served by first server clockwise on ring
- Add server: only keys between new server and previous server remapped
- Remove server: only that server's keys remapped to next server

Virtual nodes: each server has multiple positions on ring
  → More even distribution
  → Better handles heterogeneous servers (powerful = more vnodes)

Used by: Cassandra, DynamoDB, memcached, load balancers
```

---

## Availability & Reliability

### Availability Numbers

| Availability | Downtime/year | Downtime/month |
|-------------|---------------|----------------|
| 90% (one nine) | 36.5 days | 72 hours |
| 99% (two nines) | 3.65 days | 7.2 hours |
| 99.9% (three nines) | 8.77 hours | 43.8 minutes |
| 99.99% (four nines) | 52.6 minutes | 4.4 minutes |
| 99.999% (five nines) | 5.26 minutes | 26.3 seconds |

### Redundancy Patterns

```
Active-Passive (Hot Standby):
  Primary handles all traffic
  Standby replicates data, ready to take over
  Failover: DNS change / VIP switch (30s-2min)
  Use: databases, simple services

Active-Active:
  Both instances handle traffic simultaneously
  Need: session sharing, consistent data
  Use: stateless services, multi-region

N+1 Redundancy:
  Minimum: N instances needed, deploy N+1
  Any single failure: no impact

N+2 Redundancy:
  Two simultaneous failures tolerated
  Required for: multi-AZ deployments (AZ failure + rolling deployment)
```

### CAP Theorem

```
In a distributed system, you can only guarantee 2 of 3:

Consistency (C): All nodes see the same data at the same time
Availability (A): Every request receives a response
Partition Tolerance (P): System continues operating despite network partitions

Network partitions ALWAYS happen in distributed systems,
so the real choice is:

CP (Consistency + Partition Tolerance):
  During partition, refuse some requests to maintain consistency
  Examples: ZooKeeper, HBase, etcd
  Use: financial transactions, coordination services

AP (Availability + Partition Tolerance):
  During partition, continue serving (possibly stale) data
  Eventually consistent when partition heals
  Examples: CouchDB, DynamoDB, Cassandra
  Use: shopping carts, social feeds, DNS

CA (Consistency + Availability):
  Only works without partitions (single datacenter)
  Examples: Traditional RDBMS (PostgreSQL, MySQL)
  Not truly distributed
```

---

## Microservices Patterns

### Service Communication

```
Synchronous (request-response):
  REST over HTTP/HTTPS
  gRPC (binary, HTTP/2, streaming, code generation)
  GraphQL (flexible querying)
  
  Pros: simple, familiar, immediate feedback
  Cons: tight coupling, cascading failures, blocking

Asynchronous (event-driven):
  Message queues (RabbitMQ, SQS, Kafka)
  Pub/Sub patterns
  
  Pros: loose coupling, resilient, scalable
  Cons: eventual consistency, harder to debug
```

### Circuit Breaker Pattern

```
States:
  Closed → Normal operation, requests pass through
  Open   → Failure threshold exceeded, requests fail immediately
  Half-Open → Test requests allowed to check if service recovered

Implementation (pseudocode):
  class CircuitBreaker:
    state = CLOSED
    failures = 0
    threshold = 5      # Open after 5 failures
    timeout = 30s      # Try again after 30s
    
    def call(fn):
      if state == OPEN:
        if time_since_opened > timeout:
          state = HALF_OPEN
        else:
          raise CircuitOpenException()
      
      try:
        result = fn()
        if state == HALF_OPEN:
          reset()
        return result
      except:
        failures++
        if failures >= threshold or state == HALF_OPEN:
          open_circuit()
        raise

Tools: Hystrix (Java), Resilience4j, Istio (service mesh level)
```

### API Gateway Pattern

```
Client → API Gateway → Microservices

API Gateway handles:
- Authentication/Authorization (verify JWT, API keys)
- Rate limiting (per user, per IP, global)
- Request routing (path-based to services)
- SSL termination
- Request/response transformation
- Aggregation (combine multiple service responses)
- Caching
- Observability (logging, tracing headers)

Options:
- Kong, Nginx, Envoy (self-managed)
- AWS API Gateway, Azure API Management
- Traefik (Kubernetes-native)
```

### Saga Pattern (Distributed Transactions)

```
Problem: Microservices transactions can't use ACID across services

Saga: sequence of local transactions, each publishing events/messages
  On failure: compensating transactions undo previous steps

Example: Order Service Saga
  1. Create Order (Order Service)
  2. Reserve Inventory (Inventory Service)
  3. Process Payment (Payment Service)
  4. Ship Order (Shipping Service)

If Payment fails at step 3:
  Compensate step 2: Release inventory reservation
  Compensate step 1: Cancel order

Choreography: each service reacts to events (decentralized)
Orchestration: central saga orchestrator tells each service what to do
```

---

## Database Design

### SQL vs NoSQL

| Aspect | SQL (Relational) | NoSQL |
|--------|-----------------|-------|
| **Schema** | Fixed, enforced | Flexible, dynamic |
| **Transactions** | ACID guarantees | Eventual consistency (mostly) |
| **Scaling** | Vertical primary; horizontal complex | Horizontal native |
| **Query** | Complex JOINs, aggregations | Limited, simple lookups |
| **Use cases** | Financial, e-commerce, ERP | Social feeds, catalogs, analytics |
| **Examples** | PostgreSQL, MySQL, SQL Server | MongoDB, Cassandra, DynamoDB |

### Database Scaling Patterns

```
Read Replicas:
  Write to primary → replicate to N read replicas
  Direct reads to replicas
  Lag: typically <1 second (sometimes 10s of seconds)
  Use for: read-heavy workloads, analytics queries

Sharding (Horizontal Partitioning):
  Partition data across multiple databases
  Shard key: user_id, tenant_id, geographic region
  Range-based: users 1-1M → shard 1; 1M-2M → shard 2
  Hash-based: hash(user_id) % num_shards
  Challenges: cross-shard queries, rebalancing, hotspots

Connection Pooling:
  Database connections are expensive (100-200ms to establish)
  Pool maintains warm connections (PgBouncer, HikariCP)
  Limits: max_connections = 100; pool size = 20 per app instance

CQRS (Command Query Responsibility Segregation):
  Separate read and write models
  Writes: normalized, consistent, optimized for writes
  Reads: denormalized, fast, optimized for specific queries
  Event sourcing often paired with CQRS
```

---

## Caching Strategies

### Cache Patterns

```
Cache-Aside (Lazy Loading):
  1. Check cache
  2. Cache miss: read from DB, write to cache
  3. Return data
  
  Pros: only cache what's needed; cache failures don't break app
  Cons: cache miss penalty; potential stale data

Write-Through:
  1. Write to cache AND DB synchronously
  2. Always consistent
  
  Pros: cache always fresh
  Cons: write penalty; cache populated even for infrequently read data

Write-Behind (Write-Back):
  1. Write to cache immediately (return to caller)
  2. Asynchronously write to DB
  
  Pros: fast writes
  Cons: risk of data loss if cache fails before DB write

Read-Through:
  Cache sits in front of DB
  Cache automatically loads from DB on miss
  App only talks to cache
  
  Pros: transparent to application
  Cons: cold start problem
```

### Cache Eviction Policies

| Policy | Description | Use Case |
|--------|-------------|---------|
| **LRU** (Least Recently Used) | Evict oldest accessed item | General purpose |
| **LFU** (Least Frequently Used) | Evict least accessed item | Long-running caches |
| **FIFO** | First in, first out | Simple queue-like caches |
| **TTL** (Time-To-Live) | Evict after expiry | Session data, API responses |
| **Random** | Evict random item | Simple, low overhead |

---

## Message Queues

### Queue vs Pub/Sub

```
Message Queue (SQS, RabbitMQ):
  Producer → Queue → Consumer (exactly one consumer per message)
  Point-to-point
  Use: task distribution, work queues, async processing

Pub/Sub (SNS, Kafka, Pub/Sub):
  Publisher → Topic → Multiple Subscribers (fanout)
  Each subscriber gets a copy of each message
  Use: notifications, event streaming, decoupled microservices

Kafka:
  Distributed log — messages persisted, replayable
  Consumer groups: each message delivered once per group
  Topics → Partitions → Offsets
  Retention: configurable (days, weeks, forever)
  Use: event streaming, audit log, stream processing
```

---

## Design Case Studies

### Design a URL Shortener

```
Requirements:
  - Shorten URL: http://long-url.com → http://short.ly/abc123
  - Redirect: short URL → original URL
  - Scale: 100M URLs created/day, 10B redirects/day

Estimation:
  Creates: 100M/day = 1,160 writes/sec
  Redirects: 10B/day = 116,000 reads/sec (100:1 read:write ratio)
  Storage: 100M × 0.5KB = 50GB/day

Architecture:
  Client → CDN (cache redirects) → API Gateway → 
  Redirect Service → Cache (Redis) → DB (read replicas)
  Write Service → DB primary

Short code generation:
  Option 1: hash(long_url + salt) → take first 7 chars (base62)
  Option 2: global counter → encode in base62 (guarantees uniqueness)
  Option 3: random + check uniqueness in DB

Data model:
  CREATE TABLE urls (
    short_code VARCHAR(10) PRIMARY KEY,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP,
    user_id INT,
    click_count INT
  );

Scaling:
  - CDN caches popular redirects (90% cache hit rate)
  - Redis cache with TTL for hot short codes
  - DB read replicas for redirect lookups
  - Partitioned by short_code (consistent hashing)
```

### Design a CI/CD System

```
Requirements:
  - Trigger builds from Git events (push, PR)
  - Run tests in parallel
  - Store artifacts
  - Deploy to multiple environments
  - Scale to 10,000 builds/day

Architecture:
  Git (webhook) → API Gateway → Build Queue (Kafka) →
  Build Workers (ephemeral K8s pods) →
  Artifact Storage (S3/GCS) →
  CD Controller (ArgoCD/Flux) → Target Clusters

Build Worker lifecycle:
  1. Dequeue build job from Kafka
  2. Spin up isolated pod/container
  3. Clone repo (shallow, sparse)
  4. Run build stages (parallel via DAG)
  5. Publish artifacts
  6. Report status back via API
  7. Pod destroyed

Artifact storage:
  - Docker images → Container Registry (ECR/ACR/GCR)
  - Binaries → S3 with lifecycle policy
  - Test results → S3 + dashboard

Key design decisions:
  - Ephemeral workers: no state, security isolation, auto-scale
  - DAG execution: stages with dependencies, max parallelism
  - Build caching: layer cache, dependency cache (Redis/S3)
  - Secrets: vault injection per build, scoped to pipeline
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is horizontal scaling and when do you use it?**

> **Horizontal scaling** (scaling out) means adding more instances of a service behind a load balancer. You use it when you need to handle more traffic beyond what a single machine can handle, when you need high availability (no single point of failure), or when you need to handle traffic spikes. Requires stateless services (sessions stored externally, in Redis or a database).

**Q2: What is a CDN and why do DevOps engineers care about it?**

> A CDN (Content Delivery Network) distributes static content (images, CSS, JS, videos) to edge nodes close to users, reducing latency and origin server load. DevOps engineers configure CDN behavior: cache TTLs, cache invalidation on deployments, origin failover, SSL certificates, and WAF rules. AWS CloudFront, Fastly, and Cloudflare are common options.

---

### 🟡 Intermediate

**Q3: Explain the Circuit Breaker pattern.**

> Circuit Breaker prevents cascading failures. In the **closed** state, requests flow normally. After N consecutive failures, it **opens** — all requests fail immediately without trying the downstream service (fast fail). After a timeout, it enters **half-open** — allows a test request to see if the service recovered. Success → close; failure → open again. Used with Hystrix, Resilience4j, or at the service mesh layer (Istio).

**Q4: What is eventual consistency and when is it acceptable?**

> Eventual consistency means all nodes will converge to the same value, but at any given moment, different nodes may return different values. Acceptable when: reading from social feeds (slightly stale OK), shopping cart (reads from nearest replica), DNS propagation, product catalog. Not acceptable for: bank transactions, inventory decrement (can't oversell), authentication tokens.

---

### 🔴 Advanced

**Q5: Design the data architecture for a multi-region active-active system.**

> - **Application tier**: stateless services behind global load balancer (AWS Global Accelerator / CloudFlare)
> - **Session data**: geo-distributed Redis (Redis Enterprise with Active-Active geo-replication) or JWT tokens (stateless)
> - **Database**: Global DynamoDB (multi-master, auto-replication) or Spanner (global ACID) or CockroachDB
> - **Conflict resolution**: last-write-wins with vector clocks, or CRDT data structures for merge-friendly types
> - **Write routing**: write to local region; async replicate to others (eventual consistency)
> - **Read routing**: read from local region (may be slightly stale, within seconds)
> - **Failover**: detect region failure via health checks; reroute DNS/traffic in < 60 seconds

---

## Scenario-Based Design Questions

### 🔵 Design a Kubernetes-based Deployment Platform

*"Design a platform where 200 engineering teams can deploy microservices to Kubernetes without managing infrastructure."*

```
Requirements:
  - Self-service: teams create namespaces, deploy apps
  - Multi-tenancy: teams isolated from each other
  - Guardrails: resource limits, security policies
  - GitOps: Git as source of truth
  - Observability: logs, metrics, traces per team

Architecture:

1. Cluster topology
   - Multiple clusters: production (dedicated), non-prod (shared)
   - Teams: separate namespaces with resource quotas

2. Onboarding (self-service)
   - Team creates PR to gitops repo → automated namespace creation
   - Terraform/Crossplane creates: namespace, RBAC roles, 
     resource quota, network policies, service account

3. Deployment workflow
   - Team writes Helm chart + values
   - PR review → CI validates → ArgoCD syncs
   - Deployment notifications → Slack

4. Policy enforcement (OPA Gatekeeper)
   - All images must be from approved registry
   - All containers must have resource limits
   - No privileged containers
   - Required labels on all resources

5. Observability
   - Namespace-scoped Grafana dashboards per team
   - Loki log filtering by namespace labels
   - Jaeger traces with service-level access

6. Cost allocation
   - Resource tags per team/namespace
   - Kubecost or custom tooling for cost reports
```

---

> **Cross-links:** [← SRE](./SRE.md) | [Behavioral →](./Behavioral.md) | [Kubernetes (platform design) →](./Kubernetes.md) | [AWS (cloud architecture) →](./AWS.md)
