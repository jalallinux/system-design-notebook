# Database per Service

## 1. Introduction

The **Database per Service** pattern is a foundational data management principle in [microservices architecture](../architecture/microservices.md). It dictates that each microservice owns a **private database** (or at minimum a private schema/set of tables) that no other service can access directly. All inter-service data access must go through the owning service's well-defined API.

This pattern is the data-layer counterpart of service encapsulation: just as services hide their implementation details behind APIs, they also hide their data behind APIs. It is one of the most important decisions you make when moving from a monolith to microservices.

```
+---------------------------------------------+
|              Monolithic Application          |
|                                             |
|  +---------+  +---------+  +---------+      |
|  | Module A|  | Module B|  | Module C|      |
|  +----+----+  +----+----+  +----+----+      |
|       |            |            |            |
|       v            v            v            |
|  +-------------------------------------+    |
|  |         Shared Database              |    |
|  |  (All modules read/write freely)     |    |
|  +-------------------------------------+    |
+---------------------------------------------+

          vs.

+-------------+   +-------------+   +-------------+
| Service A   |   | Service B   |   | Service C   |
|  (Order)    |   | (Product)   |   |  (Search)   |
+------+------+   +------+------+   +------+------+
       |                 |                 |
       v                 v                 v
  +---------+      +-----------+    +-------------+
  |  SQL DB |      |  MongoDB  |    |Elasticsearch|
  | (Private)|     | (Private) |    |  (Private)  |
  +---------+      +-----------+    +-------------+
```

---

## 2. Context and Problem

When building a system using [microservices architecture](../architecture/microservices.md), one of the first and most impactful decisions is **how to organize the data layer**.

In a traditional monolithic application, all modules share a single relational database. This makes cross-module queries trivial (just write a JOIN) and transactions simple (just use a single ACID transaction). However, this tight coupling at the data layer creates serious problems as the system grows:

- **Schema changes** in one module can break other modules.
- **Performance bottlenecks** in one module's queries affect all modules.
- **Independent deployment** is impossible if multiple services share the same schema.
- **Technology lock-in** forces every module to use the same database technology, even when a different store would be far better suited.
- **Scaling** the database becomes an all-or-nothing operation.

The core question becomes:

> **How do we define the database architecture for a microservices-based system so that services remain loosely coupled, independently deployable, and independently scalable?**

---

## 3. Forces

Several competing forces influence the design decision:

| Force | Description |
|-------|-------------|
| **Loose coupling** | Services must be able to evolve independently. A schema change in Service A should never break Service B. |
| **Independent deployability** | Each service should be deployable on its own timeline, without coordinating database migrations with other teams. |
| **Different data storage needs** | An order service may need ACID transactions (SQL), while a product catalog may benefit from a document store (MongoDB), and a search service needs a full-text index (Elasticsearch). |
| **Cross-service queries** | Some business operations naturally require data from multiple services (e.g., "show me the order with product details and shipping status"). |
| **Data consistency** | Some operations span multiple services and need some form of transactional guarantee. |
| **Redundancy and availability** | Databases must sometimes be replicated for failover, but replication across service boundaries adds complexity. |
| **Operational simplicity** | Running many databases is harder than running one. More databases mean more backups, more monitoring, and more expertise required. |

---

## 4. Solution

### Core Principle

**Give each microservice its own private data store.** No other service may access that data store directly -- not via direct SQL queries, not via shared database links, not via database-level replication. The only way for Service B to read or modify Service A's data is through Service A's published API (REST, gRPC, async messaging, etc.).

### Levels of Isolation

The "private database" concept can be implemented at different levels of strictness:

```
Level 1: Private Tables          Level 2: Private Schema
(weakest isolation)              (moderate isolation)

+---------------------------+    +---------------------------+
|    Shared DB Server       |    |    Shared DB Server       |
|                           |    |                           |
| +--------+ +--------+    |    | +----------+ +----------+ |
| |Table A1| |Table B1|    |    | | Schema A | | Schema B | |
| |Table A2| |Table B2|    |    | | (Tables) | | (Tables) | |
| +--------+ +--------+    |    | +----------+ +----------+ |
|  Service A  Service B     |    |  Service A    Service B   |
+---------------------------+    +---------------------------+

Level 3: Private DB Server       Level 4: Polyglot Persistence
(strong isolation)               (strongest isolation)

+------------+ +------------+    +------------+ +------------+
| DB Server  | | DB Server  |    | PostgreSQL | |  MongoDB   |
| (Service A)| | (Service B)|    | (Service A)| | (Service B)|
+------------+ +------------+    +------------+ +------------+
                                 +------------+
                                 |Elasticsearch|
                                 | (Service C) |
                                 +------------+
```

| Level | Isolation | Pros | Cons |
|-------|-----------|------|------|
| **Private Tables** | Weakest | Low operational cost, easy to set up | Accidental cross-access possible, shared resource contention |
| **Private Schema** | Moderate | Namespace separation, still one DB to manage | Shared server resources, coordinated upgrades needed |
| **Private DB Server** | Strong | Full resource isolation, independent scaling | Higher operational cost, more infrastructure |
| **Polyglot Persistence** | Strongest | Best-fit technology per service | Highest operational complexity, diverse expertise needed |

### How Services Access Each Other's Data

Since direct database access is forbidden, services must use one of these approaches:

```
+-------------+         API Call           +-------------+
|  Service A  | -------------------------> |  Service B  |
| (needs B's  |    GET /api/products/42    | (owns the   |
|   data)     | <------------------------- |   data)     |
|             |    { "name": "Widget" }    |             |
+------+------+                            +------+------+
       |                                          |
       v                                          v
  +---------+                                +---------+
  | DB (A)  |                                | DB (B)  |
  +---------+                                +---------+
```

**Synchronous approaches:**
- **REST/HTTP APIs** -- Service A calls Service B's REST endpoint.
- **gRPC** -- More efficient for internal service-to-service calls.

**Asynchronous approaches:**
- **Event-driven** -- Service B publishes events when its data changes; Service A subscribes and maintains a local read-optimized copy.
- **Messaging** -- Service A sends a request message; Service B processes it and sends a reply.

### Handling Data Consistency

Without a shared database, you lose the ability to use a single ACID transaction across services. Two primary strategies address this:

1. **[Saga Pattern](../distributed-transactions/saga.md)** -- A sequence of local transactions where each service performs its own transaction and publishes an event. If any step fails, compensating transactions undo the previous steps. This achieves **eventual consistency**.

2. **Eventual Consistency with Domain Events** -- Services publish events when their data changes. Other services consume these events to update their own state. The system is temporarily inconsistent but converges to consistency.

```
     Saga: Order Creation Flow

  Order Service     Payment Service    Inventory Service
       |                  |                   |
  1. Create Order         |                   |
  (status=PENDING)        |                   |
       |                  |                   |
       +--- OrderCreated -->                  |
       |                  |                   |
       |           2. Process Payment         |
       |           (charge card)              |
       |                  |                   |
       |                  +-- PaymentDone ---->
       |                  |                   |
       |                  |            3. Reserve Stock
       |                  |            (decrement qty)
       |                  |                   |
       <-------------- StockReserved ---------+
       |                  |                   |
  4. Confirm Order        |                   |
  (status=CONFIRMED)      |                   |
```

### Querying Across Services

When a client needs data from multiple services, two patterns help:

1. **API Composition** -- An [API Composition](./api-composition.md) layer (often an [API Gateway](../architecture/api-gateway.md)) calls multiple service APIs and merges the results in memory.

2. **[CQRS](./cqrs.md)** -- Maintain a dedicated read-optimized data store that aggregates data from multiple services via events. This is especially useful for complex queries, dashboards, and reporting.

```
  API Composition Pattern:

  +--------+       +-------------+
  | Client | ----> | API Gateway |
  +--------+       |  /Composer  |
                   +------+------+
                          |
              +-----------+-----------+
              |           |           |
              v           v           v
         +---------+ +---------+ +---------+
         | Order   | | Product | | User    |
         | Service | | Service | | Service |
         +---------+ +---------+ +---------+


  CQRS Pattern:

  +----------+    events    +----------------+
  | Order    | ----------> | Read Store     |
  | Service  |             | (Denormalized  |
  +----------+             |  view joining  |
  +----------+    events   |  orders,       |
  | Product  | ----------> |  products,     |
  | Service  |             |  users)        |
  +----------+             +-------+--------+
  +----------+    events           |
  | User     | ---------->        |
  | Service  |                    |
  +----------+             +------v------+
                           |   Client    |
                           +-------------+
```

---

## 5. Example

Consider an **e-commerce platform** built with microservices. Each service chooses the best database for its specific workload:

```
+------------------------------------------------------+
|                  E-Commerce Platform                  |
+------------------------------------------------------+
|                                                      |
|  +----------------+  +----------------+  +----------+|
|  | Order Service  |  | Product Service|  |  Search  ||
|  |                |  |                |  | Service  ||
|  | - Create order |  | - Manage       |  |          ||
|  | - Track status |  |   catalog      |  | - Full   ||
|  | - Order history|  | - Pricing      |  |   text   ||
|  |                |  | - Categories   |  |   search ||
|  +-------+--------+  +-------+--------+  +----+-----+|
|          |                   |                 |      |
|          v                   v                 v      |
|   +------------+     +------------+    +------------+ |
|   | PostgreSQL |     |  MongoDB   |    |Elasticsearch||
|   |            |     |            |    |            | |
|   | - orders   |     | - products |    | - product  | |
|   | - payments |     | - reviews  |    |   index    | |
|   | - line     |     | - variants |    | - search   | |
|   |   items    |     |            |    |   suggest  | |
|   +------------+     +------------+    +------------+ |
+------------------------------------------------------+
```

**Why each database was chosen:**

| Service | Database | Reason |
|---------|----------|--------|
| **Order Service** | PostgreSQL | Needs ACID transactions for financial data, complex relational queries for order history, strong consistency guarantees |
| **Product Service** | MongoDB | Flexible schema for diverse product attributes (clothing vs electronics vs food), nested variants, easy horizontal scaling |
| **Search Service** | Elasticsearch | Full-text search, fuzzy matching, faceted navigation, autocomplete, relevance scoring |

**Data flow example -- "Customer places an order":**

1. **Order Service** creates the order in PostgreSQL (ACID transaction).
2. **Order Service** publishes `OrderCreated` event.
3. **Product Service** receives the event and decrements stock in MongoDB.
4. **Search Service** receives the event and updates product availability in Elasticsearch.
5. If the stock decrement fails, a compensating transaction (via the [Saga Pattern](../distributed-transactions/saga.md)) cancels the order.

---

## 6. Benefits and Drawbacks

### Benefits

| Benefit | Explanation |
|---------|-------------|
| **Loose coupling** | Services are not coupled through a shared data store. Schema changes are local to one service. |
| **Independent scaling** | Each database can be scaled independently based on its service's load (e.g., read replicas for Product Service, sharding for Order Service). |
| **Polyglot persistence** | Each service uses the best-fit database technology for its specific workload. |
| **Independent deployments** | Database migrations are scoped to a single service. No cross-team coordination needed. |
| **Fault isolation** | If Order Service's database goes down, Product Service and Search Service continue to operate. |
| **Team autonomy** | Each team fully owns their data model, schema design, and database operations. |

### Drawbacks

| Drawback | Explanation | Mitigation |
|----------|-------------|------------|
| **Distributed transactions** | No single ACID transaction across services. | Use the [Saga Pattern](../distributed-transactions/saga.md) for eventual consistency. |
| **Cross-service query complexity** | JOINs across service boundaries are impossible at the database level. | Use [API Composition](./api-composition.md) or [CQRS](./cqrs.md) read stores. |
| **Data duplication** | Services may need local copies of other services' data for performance. | Accept controlled duplication; use events to keep copies synchronized. |
| **Operational overhead** | More databases to provision, monitor, back up, and upgrade. | Use managed database services (RDS, Atlas, etc.) and infrastructure-as-code. |
| **Eventual consistency** | The system may be temporarily inconsistent between services. | Design UIs to accommodate eventual consistency; use idempotent operations. |
| **Debugging complexity** | Tracing data issues across multiple databases is harder than querying one shared DB. | Invest in distributed tracing (Jaeger, Zipkin) and centralized logging. |

---

## 7. Related Patterns

| Pattern | Relationship |
|---------|-------------|
| [Saga Pattern](../distributed-transactions/saga.md) | Manages distributed transactions across multiple services, each with their own database. Essential companion to Database per Service. |
| [CQRS](./cqrs.md) | Addresses the cross-service query challenge by maintaining denormalized read models aggregated from multiple services. |
| [API Composition](./api-composition.md) | A simpler alternative to CQRS for aggregating data from multiple services at query time. |
| [Shared Database](./shared-database.md) | The anti-pattern / alternative: multiple services share a single database. Simpler but creates tight coupling. |
| [Microservices Architecture](../architecture/microservices.md) | The broader architectural style that Database per Service supports. |
| [Event-Driven Architecture](../event-driven/event-driven-architecture.md) | Provides the mechanism (events) for keeping data synchronized across service boundaries. |
| [API Gateway](../architecture/api-gateway.md) | Often serves as the API Composition layer that aggregates data from multiple services. |

---

## 8. Real-World Usage

### Amazon

Amazon famously transitioned from a monolithic Oracle database to a microservices architecture where each service owns its data. Their e-commerce platform uses hundreds of services, each with its own database. This was a key enabler of their two-pizza team structure, where small autonomous teams own and operate their services end-to-end.

### Netflix

Netflix operates hundreds of microservices, each with its own data store. They use a mix of Cassandra (for high-write throughput), MySQL (for transactional data), Elasticsearch (for search), and EVCache (for caching). Their Zuul [API Gateway](../architecture/api-gateway.md) often acts as the API composition layer.

### Uber

Uber's microservices use a variety of databases chosen for specific workloads:
- **MySQL/PostgreSQL** for transactional trip data
- **Cassandra** for high-volume time-series data (location pings)
- **Elasticsearch** for search and analytics
- **Redis** for caching and real-time geospatial operations

Each service team independently manages their database technology choices, schema, and scaling.

### Industry Adoption Summary

| Company | Approach | Key Technologies |
|---------|----------|-----------------|
| Amazon | Full database-per-service, hundreds of services | DynamoDB, Aurora, RDS, ElastiCache |
| Netflix | Polyglot persistence across hundreds of services | Cassandra, MySQL, Elasticsearch, EVCache |
| Uber | Service-specific database technology selection | MySQL, PostgreSQL, Cassandra, Elasticsearch, Redis |
| Spotify | Squad-based ownership, each squad owns its data | PostgreSQL, Cassandra, Cloud Bigtable |

---

## 9. Summary

The **Database per Service** pattern is a cornerstone of successful [microservices architecture](../architecture/microservices.md). By giving each service its own private data store, you gain the full benefits of microservices: loose coupling, independent deployability, independent scaling, and technology freedom.

However, this independence comes at a cost. You lose simple cross-service JOINs and single ACID transactions, and must instead embrace patterns like the [Saga Pattern](../distributed-transactions/saga.md), [CQRS](./cqrs.md), and [API Composition](./api-composition.md) to manage data consistency and cross-service queries.

**Key decision framework:**

```
Do you need strong ACID transactions across services?
  |
  +-- Yes --> Consider Shared Database (accept the coupling)
  |           or use Saga Pattern (accept eventual consistency)
  |
  +-- No  --> Database per Service (recommended default)
              |
              +-- Same DB technology for all services?
              |     |
              |     +-- Yes --> Private Schema or Private DB Server
              |     +-- No  --> Polyglot Persistence
              |
              +-- Need cross-service queries?
                    |
                    +-- Simple --> API Composition
                    +-- Complex --> CQRS with read store
```

**When to use Database per Service:**
- You are building a microservices system with independent teams.
- Services have different data storage requirements.
- Independent deployment and scaling are priorities.
- You can accept eventual consistency between services.

**When to consider alternatives:**
- Your system has extensive cross-service transactions that require strong consistency.
- Operational overhead of managing multiple databases is too high for your team.
- The system is small enough that a monolithic database is simpler and sufficient.

---

> **Further reading:**
> - [Saga Pattern](../distributed-transactions/saga.md) -- Managing distributed transactions
> - [CQRS](./cqrs.md) -- Optimizing reads across service boundaries
> - [Microservices Architecture](../architecture/microservices.md) -- The broader architectural context
> - [Event-Driven Architecture](../event-driven/event-driven-architecture.md) -- The event backbone for data synchronization
