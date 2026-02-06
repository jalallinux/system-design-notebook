# Shared Database

## 1. Introduction

The **Shared Database** pattern is one of the most common and traditional approaches to data management in multi-service architectures. In this pattern, multiple services (or modules) share a single, centralized database. Rather than each service owning its private data store, all services read from and write to the same set of tables in a common database server.

This pattern is the default starting point for most organizations, especially those transitioning from a monolithic architecture to [Microservices](../architecture/microservices.md). While it is often considered an **anti-pattern** in mature microservices environments, it remains a pragmatic and valid choice in many real-world scenarios -- particularly during early-stage migrations or when strong data consistency is a hard requirement.

### Key Idea

> Instead of giving each service its own database, let multiple services share a single database so they can leverage ACID transactions and simple joins across service boundaries.

```
┌──────────────────────────────────────────────────────────┐
│                     Shared Database                       │
│                                                          │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │
│   │ Orders  │  │ Users   │  │Products │  │Payments │   │
│   │  Table  │  │  Table  │  │  Table  │  │  Table  │   │
│   └─────────┘  └─────────┘  └─────────┘  └─────────┘   │
│                                                          │
└──────────┬──────────┬──────────┬──────────┬──────────────┘
           │          │          │          │
           ▼          ▼          ▼          ▼
     ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
     │  Order   │ │  User    │ │ Product  │ │ Payment  │
     │ Service  │ │ Service  │ │ Service  │ │ Service  │
     └──────────┘ └──────────┘ └──────────┘ └──────────┘
```

---

## 2. Context and Problem

### Context

Consider an organization with a large monolithic application backed by a single relational database. The team decides to decompose the monolith into microservices to gain independent deployability, better scalability, and clearer team ownership. However, separating the database is one of the hardest parts of the migration.

### The Problem

When moving from a monolith to [Microservices](../architecture/microservices.md), you face a fundamental question:

**How do you manage data across multiple services while maintaining consistency, without introducing the complexity of distributed transactions?**

Splitting the database immediately means:
- You lose **ACID transactions** across service boundaries
- You need patterns like [Saga](../distributed-transactions/saga.md) or [Two-Phase Commit](../distributed-transactions/two-phase-commit.md) for cross-service consistency
- Simple SQL `JOIN` operations across service data are no longer possible
- Data duplication and synchronization become new operational challenges

For many teams, especially at the beginning of a migration, this complexity is not justified.

---

## 3. Forces

Several competing forces push towards or against the Shared Database pattern:

| Force | Pushes Towards Shared DB | Pushes Away from Shared DB |
|-------|--------------------------|----------------------------|
| **Consistency** | ACID transactions across services are trivially easy | Tight coupling limits independent evolution |
| **Simplicity** | Single DB to manage, monitor, and back up | Schema changes require cross-team coordination |
| **Performance** | Joins and aggregations are efficient on a single DB | Database becomes a bottleneck under high load |
| **Team autonomy** | Low initial coordination cost | Teams cannot independently choose their own data technology |
| **Scalability** | Good enough for small/medium scale | Vertical scaling has limits; horizontal is hard with shared state |
| **Migration cost** | Zero migration cost from monolith | Delaying separation accumulates more coupling over time |

### When Forces Favor Shared Database

- Early-stage monolith-to-microservices migration (the "strangler fig" approach)
- Small number of services (2-5) with significant data overlap
- Strong consistency requirements that cannot tolerate eventual consistency
- Small teams that can coordinate schema changes easily
- Low traffic volumes where database bottleneck is not a concern

---

## 4. Solution

### The Pattern

In the Shared Database pattern, multiple services connect to and share a common relational database. There are two variants:

#### Variant A: Shared Tables

Multiple services read from and write to the **same tables**. This is the highest degree of coupling.

```
┌────────────┐     ┌────────────┐
│   Order    │     │  Shipping  │
│  Service   │     │  Service   │
└─────┬──────┘     └─────┬──────┘
      │                  │
      │   Both read/write to orders table
      │                  │
      ▼                  ▼
┌──────────────────────────────┐
│        Shared Database       │
│  ┌────────────────────────┐  │
│  │     orders table       │  │
│  │ ┌────┬───────┬───────┐ │  │
│  │ │ id │ total │status │ │  │
│  │ └────┴───────┴───────┘ │  │
│  └────────────────────────┘  │
└──────────────────────────────┘
```

**Risk:** If the Order Service changes the `orders` table schema (e.g., renames `status` to `order_status`), the Shipping Service breaks immediately.

#### Variant B: Shared Server, Separate Schemas

Each service owns its **own schema** (or set of tables) within the same database server. Services do not directly access each other's tables but can use cross-schema joins or views when needed.

```
┌────────────┐     ┌────────────┐     ┌────────────┐
│   Order    │     │  Shipping  │     │  Payment   │
│  Service   │     │  Service   │     │  Service   │
└─────┬──────┘     └─────┬──────┘     └─────┬──────┘
      │                  │                  │
      ▼                  ▼                  ▼
┌──────────────────────────────────────────────────┐
│              Shared Database Server               │
│                                                  │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐     │
│  │  orders  │   │ shipping │   │ payments │     │
│  │  schema  │   │  schema  │   │  schema  │     │
│  └──────────┘   └──────────┘   └──────────┘     │
│                                                  │
│  (Cross-schema joins possible but discouraged)   │
└──────────────────────────────────────────────────┘
```

**This is a stepping stone** towards [Database per Service](./database-per-service.md), where each service eventually gets its own dedicated database server.

### Guidelines for Using Shared Database

1. **Define clear table ownership** -- Even if the DB is shared, each table should have one owning service. Other services should access that data via the owning service's API, not directly.
2. **Use database views or stored procedures** as a contract layer between services and the underlying schema.
3. **Establish schema change processes** -- Any schema change must be reviewed by all dependent teams.
4. **Monitor for coupling creep** -- Track which services access which tables. Alerting on cross-boundary access helps prevent silent coupling.
5. **Plan your migration path** -- Treat shared database as a temporary state, and have a roadmap to [Database per Service](./database-per-service.md).

---

## 5. Example

### Scenario: E-Commerce Monolith Migration

A mid-size e-commerce company has a monolithic application with a single PostgreSQL database. They decide to extract services incrementally.

#### Phase 1: Monolith (Starting Point)

```
┌──────────────────────────────────────┐
│          Monolith Application        │
│                                      │
│  ┌─────────┐  ┌──────────────────┐   │
│  │ Orders  │  │ User Management  │   │
│  │ Module  │  │     Module       │   │
│  └─────────┘  └──────────────────┘   │
│  ┌─────────┐  ┌──────────────────┐   │
│  │Inventory│  │    Payments      │   │
│  │ Module  │  │     Module       │   │
│  └─────────┘  └──────────────────┘   │
└──────────────────┬───────────────────┘
                   │
                   ▼
         ┌──────────────────┐
         │   PostgreSQL DB  │
         │                  │
         │  users           │
         │  orders          │
         │  order_items     │
         │  products        │
         │  inventory       │
         │  payments        │
         │  shipping        │
         └──────────────────┘
```

#### Phase 2: Extract Services, Share Database

The team extracts the User and Payment modules into separate services. All services still share the same database.

```
                    ┌──────────────┐
                    │   API GW     │
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │  Monolith   │ │    User     │ │   Payment   │
    │ (Orders +   │ │   Service   │ │   Service   │
    │  Inventory) │ │ (extracted) │ │ (extracted) │
    └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
           │               │               │
           └───────────────┼───────────────┘
                           │
                           ▼
                 ┌──────────────────┐
                 │   PostgreSQL DB  │
                 │  (shared by all) │
                 │                  │
                 │  users      ◄──── User Service owns
                 │  orders     ◄──── Monolith owns
                 │  order_items◄──── Monolith owns
                 │  products   ◄──── Monolith owns
                 │  inventory  ◄──── Monolith owns
                 │  payments   ◄──── Payment Service owns
                 │  shipping   ◄──── Monolith owns
                 └──────────────────┘
```

**At this phase:**
- The User Service owns the `users` table and exposes user data via API
- The Payment Service owns the `payments` table
- The remaining monolith still accesses `users` and `payments` tables directly (legacy coupling)
- A `BEGIN ... COMMIT` transaction can still span orders + payments

#### Phase 3: Gradually Decouple (Target State)

Over time, the team replaces direct table access with API calls, and eventually migrates each service to its own database -- achieving [Database per Service](./database-per-service.md).

```
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │   Order     │ │    User     │ │   Payment   │
    │   Service   │ │   Service   │ │   Service   │
    └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
           │               │               │
           ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ Orders   │    │ Users    │    │ Payments │
    │   DB     │    │   DB     │    │   DB     │
    └──────────┘    └──────────┘    └──────────┘
```

Cross-service data consistency is now handled by the [Saga Pattern](../distributed-transactions/saga.md) or event-driven communication.

---

## 6. Benefits and Drawbacks

### Benefits

| Benefit | Description |
|---------|-------------|
| **ACID Transactions** | A single transaction can span data from multiple services. No need for distributed transaction patterns. |
| **Simple Queries** | SQL `JOIN`s across service data are straightforward -- no API composition or data denormalization needed. |
| **No Data Duplication** | A single source of truth for all data. No synchronization or eventual consistency challenges. |
| **Familiar Tooling** | Standard RDBMS tools for backup, monitoring, migration, and BI/reporting work out of the box. |
| **Low Initial Cost** | Zero cost to start -- especially when migrating from a monolith where the shared DB already exists. |
| **Simpler Operations** | One database to monitor, backup, patch, and secure instead of many. |

### Drawbacks

| Drawback | Description |
|----------|-------------|
| **Tight Coupling** | Services are coupled through the shared schema. A column rename or type change can break multiple services simultaneously. |
| **Scaling Bottleneck** | The database becomes a single point of contention. All services compete for the same connection pool, I/O, and CPU resources. |
| **Schema Change Coordination** | Any migration requires careful coordination across all teams that access the affected tables. This slows down development velocity. |
| **Technology Lock-In** | All services must use the same database technology. You cannot use MongoDB for one service and PostgreSQL for another. |
| **Reduced Team Autonomy** | Teams cannot independently evolve their data model, choose their storage technology, or deploy schema changes on their own schedule. |
| **Testing Complexity** | Integration tests require the full shared database, making it harder to test services in isolation. |
| **Single Point of Failure** | A database outage affects all services simultaneously, reducing overall system resilience. |

### Comparison: Shared Database vs Database per Service

| Aspect | Shared Database | [Database per Service](./database-per-service.md) |
|--------|----------------|----------------------|
| **Coupling** | High -- services coupled via schema | Low -- services only coupled via APIs |
| **Consistency** | Strong (ACID) | Eventual (requires [Saga](../distributed-transactions/saga.md) or similar) |
| **Joins** | Native SQL joins | API composition or [CQRS](./cqrs.md) read models |
| **Scalability** | Limited by single DB | Each service scales independently |
| **Technology Freedom** | Locked to one DB technology | Polyglot persistence possible |
| **Operational Cost** | Low (one DB) | Higher (many databases to manage) |
| **Team Autonomy** | Low | High |
| **Migration Complexity** | None (start here) | Requires careful data extraction |
| **Failure Isolation** | Poor -- DB failure affects all | Good -- failure is isolated to one service |
| **Schema Evolution** | Requires cross-team coordination | Independent per service |
| **Best For** | Early migration, small teams, strong consistency | Mature microservices, large teams, high scale |

---

## 7. Related Patterns

### Direct Alternatives

- **[Database per Service](./database-per-service.md)** -- The opposite approach where each service owns its own database. This is the recommended target state for mature microservices architectures. Migrating from Shared Database to Database per Service is a common evolutionary path.

### Complementary Patterns

- **[Saga Pattern](../distributed-transactions/saga.md)** -- When you move away from a shared database, you lose ACID transactions across services. The Saga pattern provides an alternative for managing distributed transactions through a sequence of local transactions with compensating actions.

- **[CQRS](./cqrs.md)** -- When shared database is split, cross-service queries become difficult. CQRS helps by maintaining separate read models that aggregate data from multiple services.

- **[Microservices Architecture](../architecture/microservices.md)** -- The broader architectural context in which the Shared Database pattern (and its alternatives) exists. Microservices principles generally recommend Database per Service, but acknowledge Shared Database as a pragmatic stepping stone.

### Patterns to Consider During Migration

- **Strangler Fig Pattern** -- Incrementally replace parts of the monolith with new services while keeping the shared database, gradually migrating data ownership.
- **Change Data Capture (CDC)** -- Use tools like Debezium to capture database changes and propagate them to other services during the transition from shared to separate databases.
- **API Gateway** -- Route requests to the appropriate service as you extract services from the monolith. See [API Gateway Pattern](../architecture/api-gateway.md).

---

## 8. Real-World Usage

### Enterprises During Migration

Many large enterprises operate with shared databases, particularly during multi-year migration projects:

- **Large banks and financial institutions** often have core banking systems built on a single Oracle or DB2 database, shared by dozens of services. The migration to microservices is ongoing, but ACID consistency requirements for financial transactions make the shared database hard to replace.

- **Legacy ERP systems** (SAP, Oracle EBS) inherently use the shared database model. Modernization efforts wrap these databases with API layers while keeping the shared database underneath.

- **Government and healthcare systems** often have regulatory requirements for strong consistency and auditability that make eventual consistency patterns (like Saga) harder to justify.

### When Shared Database Makes Sense

| Scenario | Why Shared Database Works |
|----------|--------------------------|
| **Early migration phase** | Low risk, allows teams to learn microservices patterns incrementally |
| **2-5 services with heavy data overlap** | The cost of data synchronization outweighs the coupling cost |
| **Strong consistency requirements** | Financial transactions, inventory management, regulatory reporting |
| **Small team (< 10 developers)** | Communication overhead of schema coordination is manageable |
| **Read-heavy reporting workloads** | BI/analytics tools work best against a single database |
| **Short-lived project** | Not worth the investment to split the database |

### When to Migrate Away

Consider migrating to [Database per Service](./database-per-service.md) when:

1. **Schema changes become painful** -- Multiple teams are blocked by a single migration
2. **Database becomes a bottleneck** -- Connection pool exhaustion, lock contention, slow queries
3. **Teams want technology freedom** -- Some data is better served by a document store, graph DB, or time-series DB
4. **Deployment coupling is slowing you down** -- You cannot deploy one service without testing against the full shared schema
5. **Failure isolation is critical** -- A database outage should not take down the entire system

---

## 9. Summary

The Shared Database pattern is the simplest approach to data management in a multi-service architecture. All services share a single database, enabling ACID transactions, simple queries, and zero data duplication.

```
┌──────────────────────────────────────────────────────────────┐
│                     Decision Flowchart                        │
│                                                              │
│   Are you migrating from a monolith?                         │
│     YES ──► Start with Shared Database                       │
│              │                                               │
│              ▼                                               │
│   Do you have < 5 services with strong consistency needs?    │
│     YES ──► Stay with Shared Database (for now)              │
│     NO  ──► Plan migration to Database per Service           │
│              │                                               │
│              ▼                                               │
│   Are schema changes causing cross-team friction?            │
│     YES ──► Migrate to Database per Service                  │
│              Use Saga for distributed transactions            │
│              Use CQRS for cross-service queries              │
│     NO  ──► Keep Shared Database, revisit periodically       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Key Takeaways

- **Shared Database is not inherently bad** -- it is a valid pattern when used intentionally and with awareness of its trade-offs.
- **It is the natural starting point** for monolith-to-microservices migrations.
- **Define clear ownership** -- even in a shared database, each table should have one owning service.
- **Treat it as transitional** -- have a roadmap towards [Database per Service](./database-per-service.md) if you are building a large-scale microservices system.
- **Know when to move on** -- when coupling costs exceed consistency benefits, it is time to migrate.

### Further Reading

- *Building Microservices* by Sam Newman -- Chapter on data management
- *Monolith to Microservices* by Sam Newman -- Migration strategies including shared database
- *Microservices Patterns* by Chris Richardson -- Database per Service and related patterns
- Martin Fowler's articles on [Microservices](https://martinfowler.com/articles/microservices.html)

---

*[Back to Data Patterns](./README.md) | [Back to Main README](../../README.md)*
