# API Composition

## 1. Introduction

**API Composition** is a pattern for implementing queries in a microservices architecture by invoking the APIs of multiple services and combining the results in memory. Rather than performing a single SQL JOIN across tables (as in a monolith), an API Composer fetches data from each owning service separately and merges the responses before returning them to the client.

This pattern arises naturally when you adopt a [Database per Service](./database-per-service.md) strategy. Once each service owns its own data store, cross-service queries can no longer rely on database-level joins. API Composition provides a straightforward way to answer those queries without introducing shared databases or additional infrastructure.

### Why It Matters

In a typical monolithic application, answering a question like "show me the order details page" is a single SQL query that joins orders, customers, products, and shipping tables. In a microservices world, those tables live in different services with different databases. API Composition is the simplest pattern to bridge that gap.

**Key Characteristics:**

- Queries multiple service APIs to gather data
- Combines (joins, aggregates, filters) results in memory
- Can be implemented in the [API Gateway](../architecture/api-gateway.md) or a dedicated service
- Works with any service regardless of its technology stack
- No additional infrastructure required beyond the services themselves

---

## 2. Context and Problem

### The Microservices Data Challenge

When you decompose a monolith into microservices with [Database per Service](./database-per-service.md), each service owns its data exclusively:

```
MONOLITH (Single Database)
┌──────────────────────────────────────────────┐
│                  Database                     │
│                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │  Orders   │ │Customers │ │ Products │    │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘    │
│       │             │            │           │
│       └─────────────┴────────────┘           │
│              SQL JOIN                        │
│  SELECT o.*, c.name, p.title                 │
│  FROM orders o                               │
│  JOIN customers c ON o.customer_id = c.id    │
│  JOIN products p  ON o.product_id  = p.id    │
└──────────────────────────────────────────────┘
        Simple. One query. Done.


MICROSERVICES (Database per Service)
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Order Service│  │Customer Svc  │  │Product Svc   │
│  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │
│  │OrdersDB│  │  │  │CustDB  │  │  │  │ProdDB  │  │
│  └────────┘  │  │  └────────┘  │  │  └────────┘  │
└──────────────┘  └──────────────┘  └──────────────┘
       ?                 ?                 ?
       └─────────────────┴─────────────────┘
         How do we JOIN across these?
```

### The Core Problem

**How do you implement queries that need to retrieve data owned by multiple services when you can no longer perform SQL JOINs across service boundaries?**

For example, an e-commerce order details page needs:

| Data | Owning Service |
|------|---------------|
| Order ID, status, total | Order Service |
| Customer name, email | Customer Service |
| Product names, images | Product Service |
| Tracking number, ETA | Shipping Service |

In a monolith, this is one query. In microservices, the data is scattered across four independent services with four independent databases.

---

## 3. Forces

Several forces shape the solution to this problem:

| Force | Description |
|-------|-------------|
| **Data Ownership** | Each service owns its data; no shared database allowed |
| **Query Complexity** | Some queries require data from two or more services |
| **Performance** | End users expect fast responses (< 500ms) even for composite queries |
| **Simplicity** | The solution should be easy to understand, implement, and maintain |
| **Availability** | Partial service failures should not necessarily fail the entire query |
| **Consistency** | Data from different services may be at slightly different points in time |
| **Service Autonomy** | Services should remain independently deployable and evolvable |
| **Technology Diversity** | Services may use different languages, frameworks, and databases |

The tension is clear: you need to combine data from multiple sources while keeping services decoupled, maintaining reasonable performance, and handling failures gracefully.

---

## 4. Solution

### The API Composition Pattern

An **API Composer** (also called an aggregator) sits between the client and the backend services. It receives a single request from the client, fans out to multiple services, gathers their responses, and merges the results into a single response.

### How It Works Step by Step

```
Step 1: Client sends a single request

┌──────────┐
│  Client  │ ─── GET /order-details/789 ───▶
└──────────┘

Step 2: API Composer receives the request and identifies which services to call

┌────────────────────────────────────────────┐
│            API Composer                     │
│                                            │
│  Request: GET /order-details/789           │
│                                            │
│  Plan:                                     │
│   1. Order Service   → GET /orders/789     │
│   2. Customer Service→ GET /customers/456  │
│   3. Product Service → GET /products/101   │
│   4. Shipping Service→ GET /shipments/789  │
└────────────────────────────────────────────┘

Step 3: API Composer calls services in parallel

┌────────────────────────────────────────────┐
│            API Composer                     │
│                                            │
│  ┌────────────┐ ┌────────────┐            │
│  │  Thread 1  │ │  Thread 2  │            │
│  │Order Svc   │ │Customer Svc│            │
│  │  45ms      │ │  30ms      │            │
│  └────────────┘ └────────────┘            │
│  ┌────────────┐ ┌────────────┐            │
│  │  Thread 3  │ │  Thread 4  │            │
│  │Product Svc │ │Shipping Svc│            │
│  │  55ms      │ │  40ms      │            │
│  └────────────┘ └────────────┘            │
│                                            │
│  Total wall-clock time: ~55ms              │
│  (limited by slowest call)                 │
└────────────────────────────────────────────┘

Step 4: API Composer combines results and returns to client

┌────────────────────────────────────────────┐
│            API Composer                     │
│                                            │
│  Combined Response:                        │
│  {                                         │
│    orderId: 789,                           │
│    status: "shipped",                      │
│    customer: { name: "Alice", ... },       │
│    products: [ { name: "Widget", ... } ],  │
│    shipping: { tracking: "TRK123", ... }   │
│  }                                         │
└────────────────────────────────────────────┘
          │
          ▼
     ┌──────────┐
     │  Client  │  ◀── Single aggregated response
     └──────────┘
```

### High-Level Flow Diagram

```
┌──────────┐         ┌───────────────────────────────────────────────┐
│          │  (1)    │              API Composer                      │
│  Client  │────────▶│                                               │
│          │         │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐     │
│          │         │  │Svc A │  │Svc B │  │Svc C │  │Svc D │     │
│          │◀────────│  │  ▼   │  │  ▼   │  │  ▼   │  │  ▼   │     │
│          │  (6)    │  │Resp A│  │Resp B│  │Resp C│  │Resp D│     │
└──────────┘         │  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘     │
                     │     │         │         │         │          │
                     │     └─────────┴─────────┴─────────┘          │
                     │                    │                          │
                     │              (5) Combine                      │
                     │                    │                          │
                     │              Merged Result                    │
                     └───────────────────────────────────────────────┘

  (1) Client sends request
  (2) Composer fans out to Service A  ─┐
  (3) Composer fans out to Service B  ─┤ In parallel
  (4) Composer fans out to Service C  ─┤
      Composer fans out to Service D  ─┘
  (5) Composer merges all responses
  (6) Composer returns combined result to client
```

### Two Implementation Approaches

There are two main ways to implement the API Composer:

#### Approach 1: API Gateway as Composer

The [API Gateway](../architecture/api-gateway.md) itself handles the composition logic.

```
┌──────────┐       ┌──────────────────────────────┐
│          │       │     API Gateway               │
│  Client  │──────▶│  ┌────────────────────────┐  │
│          │       │  │  Composition Logic      │  │
│          │◀──────│  │  - Fan-out to services  │  │
│          │       │  │  - Merge responses      │  │
└──────────┘       │  │  - Handle errors        │  │
                   │  └────────────────────────┘  │
                   └──────────┬────────────────────┘
                              │
                   ┌──────────┼──────────┐
                   ▼          ▼          ▼
              ┌────────┐ ┌────────┐ ┌────────┐
              │Svc A   │ │Svc B   │ │Svc C   │
              └────────┘ └────────┘ └────────┘
```

**When to use:**

- Simple compositions (2-3 services)
- The gateway already handles routing
- No complex business logic in the merge step

**Drawbacks:**

- Bloats the gateway with composition logic
- Harder to test composition in isolation
- Changes to composition require gateway redeployment

#### Approach 2: Dedicated Composer Service

A standalone service handles composition for a specific domain or set of queries.

```
┌──────────┐       ┌───────────┐       ┌──────────────────────┐
│          │       │           │       │  Order Details       │
│  Client  │──────▶│API Gateway│──────▶│  Composer Service    │
│          │       │ (routing) │       │                      │
│          │◀──────│           │◀──────│  - Fetch order       │
└──────────┘       └───────────┘       │  - Fetch customer    │
                                       │  - Fetch products    │
                                       │  - Merge & return    │
                                       └──────────┬───────────┘
                                                  │
                                       ┌──────────┼──────────┐
                                       ▼          ▼          ▼
                                  ┌────────┐ ┌────────┐ ┌────────┐
                                  │Order   │ │Customer│ │Product │
                                  │Service │ │Service │ │Service │
                                  └────────┘ └────────┘ └────────┘
```

**When to use:**

- Complex compositions with business logic
- Composition logic changes frequently
- Need to test composition independently
- Multiple query types for the same domain

**Drawbacks:**

- Extra service to deploy and maintain
- Additional network hop (gateway -> composer -> services)

### Comparison of Approaches

| Aspect | Gateway as Composer | Dedicated Composer Service |
|--------|-------------------|--------------------------|
| **Simplicity** | Simpler for basic cases | Requires separate service |
| **Flexibility** | Limited by gateway framework | Full programming flexibility |
| **Testing** | Harder to test in isolation | Independently testable |
| **Deployment** | Coupled to gateway releases | Independent deployment |
| **Scalability** | Scales with gateway | Scales independently |
| **Best for** | 2-3 services, simple merge | Complex logic, many services |

### Handling Failures

When one of the downstream services fails, you have several options:

```
Scenario: Shipping Service is down

┌────────────────────────────────────────────┐
│            API Composer                     │
│                                            │
│  Order Service    ── 200 OK  ── ✓         │
│  Customer Service ── 200 OK  ── ✓         │
│  Product Service  ── 200 OK  ── ✓         │
│  Shipping Service ── TIMEOUT ── ✗         │
│                                            │
│  Strategy Options:                         │
│  ┌──────────────────────────────────────┐ │
│  │ 1. PARTIAL RESULTS                   │ │
│  │    Return available data with null   │ │
│  │    for shipping. Let client handle.  │ │
│  ├──────────────────────────────────────┤ │
│  │ 2. FALLBACK / CACHED DATA           │ │
│  │    Return last-known shipping data   │ │
│  │    from cache with a stale flag.     │ │
│  ├──────────────────────────────────────┤ │
│  │ 3. FAIL ENTIRE REQUEST              │ │
│  │    Return 503 to client.            │ │
│  │    Use when all data is essential.   │ │
│  ├──────────────────────────────────────┤ │
│  │ 4. DEFAULT VALUES                    │ │
│  │    Return placeholder: "Tracking    │ │
│  │    info temporarily unavailable."    │ │
│  └──────────────────────────────────────┘ │
└────────────────────────────────────────────┘
```

**Best practice:** Decide per-field whether data is essential or optional. Return partial results for optional data and fail only when essential data is missing.

### Performance Considerations

```
SEQUENTIAL (Bad for independent calls)
──────────────────────────────────────────────────

  Order Svc    ├──── 45ms ────┤
  Customer Svc                 ├──── 30ms ────┤
  Product Svc                                  ├──── 55ms ────┤
  Shipping Svc                                                 ├──── 40ms ────┤

  Total: 170ms


PARALLEL (Preferred for independent calls)
──────────────────────────────────────────────────

  Order Svc    ├──── 45ms ────┤
  Customer Svc ├──── 30ms ───┤
  Product Svc  ├──── 55ms ─────┤
  Shipping Svc ├──── 40ms ────┤

  Total: 55ms  (limited by slowest)


HYBRID (When calls have dependencies)
──────────────────────────────────────────────────

  Step 1: Order Svc  ├──── 45ms ────┤  (need order to get customer_id)
  Step 2 (parallel):
    Customer Svc  ├──── 30ms ───┤
    Product Svc   ├──── 55ms ─────┤
    Shipping Svc  ├──── 40ms ────┤

  Total: 100ms
```

**Optimization techniques:**

| Technique | Description | Impact |
|-----------|-------------|--------|
| **Parallel calls** | Call independent services simultaneously | Major latency reduction |
| **Caching** | Cache frequently accessed data (product info, customer profiles) | Eliminates network calls |
| **Timeouts** | Set aggressive per-service timeouts | Prevents slow services from blocking |
| **Connection pooling** | Reuse HTTP connections to backend services | Reduces connection overhead |
| **Data locality** | Co-locate composer near services | Lower network latency |
| **Batch APIs** | Fetch multiple items in one call (e.g., products by IDs) | Fewer HTTP round trips |

---

## 5. Example

### E-Commerce Order Details Page

A customer visits their order details page on an e-commerce site. The page must show:

```
┌───────────────────────────────────────────────────┐
│  Order #789                    Status: Shipped     │
├───────────────────────────────────────────────────┤
│                                                   │
│  Customer: Alice Johnson                          │
│  Email:    alice@example.com                      │
│                                                   │
│  Items:                                           │
│  ┌─────────────────────────────────────────────┐ │
│  │  1x Widget Pro       $49.99                 │ │
│  │  2x Gadget Mini      $29.99 ea              │ │
│  └─────────────────────────────────────────────┘ │
│                                                   │
│  Subtotal:     $109.97                            │
│  Shipping:      $5.99                             │
│  Total:       $115.96                             │
│                                                   │
│  Shipping:                                        │
│  Carrier:  FedEx                                  │
│  Tracking: TRK-2024-ABC123                        │
│  ETA:      Feb 10, 2026                           │
│                                                   │
└───────────────────────────────────────────────────┘
```

### Service Calls Needed

```
┌───────────────────────────────────────────────────────────────┐
│                  Order Details Composer                        │
│                                                               │
│  Input: orderId = 789                                         │
│                                                               │
│  Step 1: Fetch the order (need customer_id and product_ids)   │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ GET /orders/789                                         │ │
│  │ Response:                                               │ │
│  │ { orderId: 789, customerId: 456,                        │ │
│  │   items: [{productId:101, qty:1}, {productId:102, qty:2}],│ │
│  │   subtotal: 109.97, shipping: 5.99, total: 115.96,     │ │
│  │   status: "shipped" }                                   │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  Step 2: Fetch remaining data in parallel                     │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────┐ │
│  │Customer Service   │ │Product Service   │ │Shipping Svc  │ │
│  │GET /customers/456 │ │GET /products?    │ │GET /shipments│ │
│  │                   │ │  ids=101,102     │ │  ?orderId=789│ │
│  │{ name: "Alice     │ │[{ id:101,        │ │{ carrier:    │ │
│  │  Johnson",        │ │  name:"Widget    │ │  "FedEx",    │ │
│  │  email: "alice@   │ │  Pro",           │ │  tracking:   │ │
│  │  example.com" }   │ │  price: 49.99 }, │ │  "TRK-...",  │ │
│  │                   │ │ { id:102,        │ │  eta: "2026- │ │
│  │                   │ │  name:"Gadget    │ │  02-10" }    │ │
│  │                   │ │  Mini",          │ │              │ │
│  │                   │ │  price: 29.99 }] │ │              │ │
│  └──────────────────┘ └──────────────────┘ └──────────────┘ │
│                                                               │
│  Step 3: Merge into unified response                          │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ {                                                       │ │
│  │   orderId: 789,                                         │ │
│  │   status: "shipped",                                    │ │
│  │   customer: { name: "Alice Johnson",                    │ │
│  │               email: "alice@example.com" },             │ │
│  │   items: [                                              │ │
│  │     { name: "Widget Pro",  qty: 1, price: 49.99 },     │ │
│  │     { name: "Gadget Mini", qty: 2, price: 29.99 }      │ │
│  │   ],                                                    │ │
│  │   subtotal: 109.97, shipping: 5.99, total: 115.96,     │ │
│  │   delivery: { carrier: "FedEx",                         │ │
│  │               tracking: "TRK-2024-ABC123",              │ │
│  │               eta: "2026-02-10" }                       │ │
│  │ }                                                       │ │
│  └─────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

### Comparison: API Composition vs Direct DB Query

```
DIRECT DB QUERY (Monolith)
──────────────────────────────────────────────
  SELECT o.id, o.status, o.total,
         c.name, c.email,
         p.name, oi.qty, p.price,
         s.carrier, s.tracking, s.eta
  FROM orders o
  JOIN customers c  ON o.customer_id = c.id
  JOIN order_items oi ON oi.order_id = o.id
  JOIN products p   ON oi.product_id = p.id
  JOIN shipments s  ON s.order_id = o.id
  WHERE o.id = 789;

  Latency: ~5ms (single DB call)
  Consistency: Strong (single transaction)


API COMPOSITION (Microservices)
──────────────────────────────────────────────
  1. GET /orders/789            → 45ms
  2. GET /customers/456         → 30ms  ─┐
     GET /products?ids=101,102  → 55ms  ─┤ parallel
     GET /shipments?orderId=789 → 40ms  ─┘

  Latency: ~100ms (sequential + parallel)
  Consistency: Eventual (data from different points in time)
```

The trade-off is clear: API Composition is slower and provides weaker consistency, but it preserves service independence and allows each service to scale, deploy, and evolve independently.

---

## 6. Benefits and Drawbacks

### Benefits

| Benefit | Explanation |
|---------|-------------|
| **Simple to implement** | No special infrastructure needed; just HTTP calls and in-memory merging |
| **Works with any service** | Language-agnostic; any service with an API can participate |
| **No shared database** | Preserves [Database per Service](./database-per-service.md) encapsulation |
| **Flexible** | Easy to add or remove services from the composition |
| **Incremental adoption** | Can be applied query-by-query without changing existing services |
| **Familiar programming model** | Developers understand HTTP calls and data transformation |
| **No extra infrastructure** | Unlike [CQRS](./cqrs.md), no event store or read-model databases needed |

### Drawbacks

| Drawback | Explanation | Mitigation |
|----------|-------------|------------|
| **Increased latency** | Multiple network calls add up, even in parallel | Parallel calls, caching, timeouts |
| **In-memory join limitations** | Large datasets cannot be joined efficiently in memory | Pagination, limit result sizes |
| **Data consistency** | Different services may return data from different points in time | Accept eventual consistency or use versioning |
| **Partial failure handling** | One failing service can degrade or break the entire query | Fallbacks, circuit breakers, partial results |
| **Availability reduction** | Overall availability is the product of individual service availabilities | Caching, fallbacks, graceful degradation |
| **No complex aggregations** | Hard to do GROUP BY, SUM across service boundaries in memory | Consider [CQRS](./cqrs.md) for complex analytics |
| **Increased network traffic** | Multiple internal calls per single client request | Batch APIs, caching |
| **Composition logic complexity** | Merging heterogeneous data can become complex over time | Keep compositions focused; split into multiple composers |

### Availability Math

If each service has 99.9% availability and you compose across 4 services:

```
Combined availability = 99.9% x 99.9% x 99.9% x 99.9%
                      = 99.6%

This means ~3.5 hours of additional downtime per year
compared to a single service.

Mitigation: Use caching and fallbacks to decouple
availability from downstream services.
```

---

## 7. Related Patterns

### API Composition vs CQRS

When API Composition becomes too complex or too slow, [CQRS (Command Query Responsibility Segregation)](./cqrs.md) is the typical alternative.

```
API COMPOSITION                          CQRS
─────────────────                        ─────────────────

Query time:                              Write time:
  Composer calls N services              Events update a pre-built
  and merges in memory                   denormalized read model

  ┌──────────┐                           ┌──────────┐
  │  Client  │                           │  Client  │
  └─────┬────┘                           └─────┬────┘
        │ query                                │ query
        ▼                                      ▼
  ┌──────────────┐                       ┌──────────────┐
  │   Composer   │                       │  Read Model  │
  │  (runtime    │                       │  (pre-joined │
  │   merging)   │                       │   database)  │
  └──┬──┬──┬──┬──┘                       └──────────────┘
     │  │  │  │                          Single fast query
     ▼  ▼  ▼  ▼
  Svc A B C D
  Multiple network calls
```

| Aspect | API Composition | CQRS |
|--------|----------------|------|
| **Query latency** | Higher (multiple calls) | Lower (single read from pre-built view) |
| **Implementation effort** | Low | High (event handlers, read model, sync) |
| **Infrastructure** | None extra | Event store, read database |
| **Data freshness** | Real-time from services | Eventually consistent (sync lag) |
| **Complex aggregations** | Difficult (in-memory) | Easy (pre-computed in read model) |
| **Best for** | Simple joins, low-moderate query volume | Complex queries, high query volume, analytics |

**Rule of thumb:** Start with API Composition. If a query becomes too slow, involves too many services, or requires complex aggregations, migrate that specific query to [CQRS](./cqrs.md).

### Related Pattern Summary

| Pattern | Relationship to API Composition |
|---------|-------------------------------|
| [Database per Service](./database-per-service.md) | Creates the need for API Composition |
| [CQRS](./cqrs.md) | Alternative for complex or high-volume queries |
| [API Gateway](../architecture/api-gateway.md) | Common location for implementing the composer |
| [Circuit Breaker](../resilience/circuit-breaker.md) | Protects composer from failing downstream services |
| [Saga Pattern](../distributed-transactions/saga.md) | Handles distributed writes; API Composition handles distributed reads |
| [Event-Driven Architecture](../event-driven/event-driven-architecture.md) | Events can keep local caches fresh to speed up composition |

---

## 8. Real-World Usage

### Netflix: API Gateway Composition

Netflix pioneered API composition at scale through their API Gateway (originally Zuul, now Spring Cloud Gateway). Their gateway composes responses from dozens of microservices to build the data needed for a single screen in their apps.

```
┌────────────────────────────────────────────────────┐
│             Netflix API Gateway                     │
│                                                    │
│  Screen: "Movie Details Page"                      │
│  Composes data from:                               │
│  ┌──────────────────────────────────────────────┐ │
│  │ Movie Metadata Service   → title, year, plot │ │
│  │ Rating Service           → user rating, avg  │ │
│  │ Recommendation Service   → "more like this"  │ │
│  │ Availability Service     → streaming regions │ │
│  │ Image Service            → poster, backdrop  │ │
│  │ Audio/Subtitle Service   → language options  │ │
│  └──────────────────────────────────────────────┘ │
│                                                    │
│  Key decisions:                                    │
│  • Parallel calls with per-service timeouts       │
│  • Fallback to cached data when a service is slow │
│  • Different compositions for TV, mobile, web     │
│    (similar to BFF pattern)                       │
└────────────────────────────────────────────────────┘
```

### GraphQL as a Composition Layer

GraphQL is increasingly used as a declarative API Composition layer. Instead of the server deciding what to compose, the client specifies exactly which fields it needs, and the GraphQL resolver fetches from the relevant services.

```
┌──────────┐       ┌───────────────────────────────────┐
│          │       │        GraphQL Server              │
│  Client  │──────▶│                                   │
│          │       │  query {                           │
│          │       │    order(id: 789) {                │
│          │       │      status          → Order Svc   │
│          │       │      customer {                    │
│          │       │        name          → Customer Svc│
│          │       │      }                             │
│          │       │      items {                       │
│          │       │        product {                   │
│          │       │          name        → Product Svc │
│          │       │          price       → Product Svc │
│          │       │        }                           │
│          │       │      }                             │
│          │       │    }                               │
│          │       │  }                                 │
│          │       │                                   │
│          │◀──────│  Resolvers fetch from each service │
└──────────┘       └───────────────────────────────────┘
```

**Benefits of GraphQL for composition:**

- Client controls exactly which data it needs (no over-fetching)
- Single round trip from client to server
- Strong typing via schema
- Built-in tooling for batching and caching (DataLoader)

### Amazon: Service Mesh Composition

Amazon composes product detail pages from many internal services:

```
Product Detail Page composes from:
├── Catalog Service        → product info, images
├── Pricing Service        → current price, deals
├── Inventory Service      → stock availability
├── Reviews Service        → ratings, user reviews
├── Recommendations Service→ "customers also bought"
├── Shipping Service       → delivery estimates
├── Advertising Service    → sponsored products
└── Seller Service         → seller info, ratings
```

At Amazon's scale, each service handles millions of requests per second. API Composition is the default pattern for read-heavy pages, with aggressive caching and circuit breakers at every layer.

### When Companies Outgrow API Composition

As query complexity grows, many organizations adopt a hybrid approach:

```
┌─────────────────────────────────────────────────┐
│  Query Type          │  Pattern Used             │
├──────────────────────┼──────────────────────────┤
│  Simple page loads   │  API Composition          │
│  Dashboard views     │  API Composition + Cache  │
│  Search              │  CQRS (Elasticsearch)     │
│  Analytics/Reports   │  CQRS (Data Warehouse)    │
│  Real-time feeds     │  Event-Driven + CQRS      │
└─────────────────────────────────────────────────┘
```

---

## 9. Summary

API Composition is the **simplest and most natural pattern** for querying data across microservices. It preserves service autonomy, requires no additional infrastructure, and works with any technology stack.

### Key Takeaways

1. **API Composition answers cross-service queries** by calling multiple service APIs and merging results in memory.

2. **It is the default starting point** when you adopt [Database per Service](./database-per-service.md). Use it until you have evidence that something more complex is needed.

3. **Parallel calls are essential** for acceptable latency. Sequential composition across many services will be too slow.

4. **Handle partial failures gracefully.** Not all data is equally critical. Return partial results with fallbacks for non-essential data.

5. **Two implementation locations:** the [API Gateway](../architecture/api-gateway.md) for simple cases, or a dedicated composer service for complex logic.

6. **Know when to graduate to [CQRS](./cqrs.md).** If a query touches too many services, requires complex aggregations, or must serve very high read volumes, CQRS with a pre-built read model is the next step.

7. **Caching is your best friend.** Cache stable data (product info, customer profiles) aggressively to reduce the number of actual network calls.

8. **Monitor composition latency.** Track P50, P95, and P99 latencies for each composed query and for each downstream service call.

9. **Design service APIs with composition in mind.** Provide batch endpoints (e.g., `GET /products?ids=1,2,3`) to reduce the number of calls.

10. **API Composition is a read pattern.** For distributed writes, look at the [Saga Pattern](../distributed-transactions/saga.md) or [Two-Phase Commit](../distributed-transactions/two-phase-commit.md).

### Decision Framework

```
┌─────────────────────────────────────────┐
│ Do you need data from multiple services?│
└─────────────────┬───────────────────────┘
                  │
             No   │   Yes
                  │
   ┌──────────────▼──────────────┐
   │ Call the single service     │
   │ directly. No composition.   │
   └─────────────────────────────┘
                  │
                  │ Yes
                  ▼
┌─────────────────────────────────────────┐
│ Is the query simple (< 5 services,     │
│ no complex aggregation)?               │
└─────────────────┬───────────────────────┘
                  │
             No   │   Yes
                  │
   ┌──────────────▼──────────────┐
   │ Use API Composition.        │
   │ Start simple, add caching   │
   │ and fallbacks as needed.    │
   └─────────────────────────────┘
                  │
                  │ No (complex query, high volume, analytics)
                  ▼
┌─────────────────────────────────────────┐
│ Consider CQRS with a dedicated         │
│ read model for this specific query.    │
└─────────────────────────────────────────┘
```

---

**Related Topics:**

- [CQRS](./cqrs.md) - Alternative for complex queries with pre-built read models
- [Database per Service](./database-per-service.md) - The pattern that creates the need for API Composition
- [API Gateway](../architecture/api-gateway.md) - Common location for implementing composition
- [Circuit Breaker](../resilience/circuit-breaker.md) - Protecting the composer from downstream failures
- [Saga Pattern](../distributed-transactions/saga.md) - Distributed write operations
- [Event-Driven Architecture](../event-driven/event-driven-architecture.md) - Events for keeping cached data fresh
- [CAP Theorem](../fundamentals/cap-theorem.md) - Understanding consistency trade-offs in distributed queries

**References:**

- "Microservices Patterns" by Chris Richardson (Chapter 7: API Composition)
- microservices.io - API Composition pattern
- Netflix Tech Blog: Optimizing the Netflix API
- "Building Microservices" by Sam Newman (Chapter on Data)
- GraphQL documentation on query composition
