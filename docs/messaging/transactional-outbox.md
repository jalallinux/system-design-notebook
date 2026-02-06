# Transactional Outbox

## 1. Introduction

In a microservices architecture where each service owns its own database (the **Database per Service** pattern), one of the most critical challenges is ensuring that a service can **atomically update its database and publish an event or message**. The **Transactional Outbox** pattern solves this problem by leveraging the local database transaction to guarantee reliable message delivery without requiring distributed transactions.

Instead of publishing events directly to a message broker (such as [RabbitMQ or Kafka](./rabbitmq-vs-kafka.md)), the service writes the event into a special **outbox table** within the same database transaction that persists the business state change. A separate process then reads from the outbox table and publishes the events to the message broker.

This pattern is one of the cornerstones of reliable [Event-Driven Architecture](../event-driven/event-driven-architecture.md) and is widely adopted in production systems.

---

## 2. Context and Problem

### Context

Consider a microservices system where:

- Each service has its own database (**Database per Service** pattern).
- Services need to communicate state changes to other services via events or messages.
- The system relies on a message broker (Kafka, RabbitMQ, etc.) for asynchronous communication.

### The Dual Write Problem

When a service needs to update its database **and** publish a message, it performs two distinct operations against two different systems. This is known as the **dual write problem**.

```
                         Dual Write Problem
  ┌─────────────┐
  │             │   1. Write to DB        ┌──────────────┐
  │   Order     │ ───────────────────────>│   Database    │  ✅ Success
  │   Service   │                         └──────────────┘
  │             │   2. Publish Event       ┌──────────────┐
  │             │ ────────────────────────>│ Message       │  ❌ Failure!
  └─────────────┘                         │ Broker        │
                                          └──────────────┘

  Result: Database is updated, but no event is published.
          Other services never learn about the change.
          System is now INCONSISTENT.
```

**What can go wrong:**

| Scenario | DB Write | Message Publish | Result |
|----------|----------|-----------------|--------|
| Happy path | Success | Success | Consistent |
| Broker down | Success | Failure | DB updated, no event published |
| Service crash after DB write | Success | Never attempted | DB updated, no event published |
| DB down | Failure | Success | Event published, but no DB change |
| Network partition | Success | Timeout / Unknown | Unknown state |

The fundamental issue is that **updating a database and publishing a message are two separate operations that cannot be made atomic** without a distributed transaction protocol. Using [Two-Phase Commit (2PC)](../distributed-transactions/two-phase-commit.md) between a database and a message broker is impractical because most message brokers do not support XA/2PC, and it would introduce significant performance and availability penalties.

---

## 3. Forces

The following forces shape the need for this pattern:

- **Atomicity**: The business state change and the event publication must either both succeed or both fail. Partial completion leads to system inconsistency.
- **No distributed transactions**: 2PC between a database and a message broker is impractical, unsupported by many brokers, and harms availability (violating the [CAP Theorem](../fundamentals/cap-theorem.md) trade-offs).
- **Reliability**: Events must not be lost. Other services depend on receiving events to maintain their own state (e.g., in [Saga](../distributed-transactions/saga.md) workflows or [CQRS](../data-patterns/cqrs.md) read model updates).
- **Ordering**: Events should ideally be published in the order they were created.
- **Performance**: The solution should not significantly degrade the performance of the primary business operation.
- **Simplicity**: The solution should be understandable and maintainable by the development team.

---

## 4. Solution

### Core Idea

Instead of publishing events directly to the message broker, the service writes the event to an **outbox table** in the **same local database transaction** as the business data change. A separate component then reads the outbox table and publishes the events to the message broker.

```
                    Transactional Outbox Pattern
  ┌─────────────┐
  │             │      BEGIN TRANSACTION
  │   Order     │   1. INSERT INTO orders (...)        ┌──────────────┐
  │   Service   │ ────────────────────────────────────>│              │
  │             │   2. INSERT INTO outbox (...)         │   Database   │
  │             │ ────────────────────────────────────>│              │
  │             │      COMMIT                          └──────┬───────┘
  └─────────────┘                                             │
                                                              │ Reads
                                                              │ outbox
  ┌─────────────┐                                             │
  │  Message    │      3. Publish event        ┌──────────────▼───────┐
  │  Relay      │ <───────────────────────────│   Outbox Table       │
  │  Process    │                              └──────────────────────┘
  │             │      4. Publish to broker     ┌──────────────┐
  │             │ ────────────────────────────>│ Message       │
  └─────────────┘                              │ Broker        │
                                               └──────────────┘
```

Because both the business data write and the outbox entry are within the **same ACID transaction**, they are guaranteed to either both succeed or both fail. The message relay process is responsible for reading unpublished events from the outbox and forwarding them to the message broker.

### Outbox Table Schema

The outbox table stores events that need to be published. A typical schema:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         outbox Table                                │
├───────────────┬──────────────┬──────────────────────────────────────┤
│ Column        │ Type         │ Description                          │
├───────────────┼──────────────┼──────────────────────────────────────┤
│ id            │ UUID / BIGINT│ Unique identifier for the event      │
│ aggregate_type│ VARCHAR(255) │ Type of the aggregate (e.g., "Order")│
│ aggregate_id  │ VARCHAR(255) │ ID of the aggregate instance         │
│ event_type    │ VARCHAR(255) │ Type of event (e.g., "OrderCreated") │
│ payload       │ JSON / TEXT  │ Serialized event data                │
│ created_at    │ TIMESTAMP    │ When the event was created           │
│ published_at  │ TIMESTAMP    │ When the event was published (NULL   │
│               │              │ if not yet published)                │
└───────────────┴──────────────┴──────────────────────────────────────┘
```

**SQL DDL example:**

```sql
CREATE TABLE outbox (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(255) NOT NULL,
    aggregate_id   VARCHAR(255) NOT NULL,
    event_type     VARCHAR(255) NOT NULL,
    payload        JSONB NOT NULL,
    created_at     TIMESTAMP NOT NULL DEFAULT NOW(),
    published_at   TIMESTAMP NULL
);

CREATE INDEX idx_outbox_unpublished ON outbox (created_at)
    WHERE published_at IS NULL;
```

### Two Delivery Mechanisms

Once events are in the outbox table, they need to be delivered to the message broker. There are two primary approaches:

#### 1. Polling Publisher

A background process periodically **polls** the outbox table for unpublished events, publishes them, and marks them as published.

```
┌────────────────┐    SELECT * FROM outbox     ┌────────────┐
│   Polling      │    WHERE published_at IS NULL│            │
│   Publisher    │ <──────────────────────────  │  Database  │
│   (cron/worker)│                              │            │
│                │    UPDATE outbox SET          └────────────┘
│                │    published_at = NOW()
│                │
│                │    Publish message             ┌────────────┐
│                │ ─────────────────────────────>│  Message   │
└────────────────┘                               │  Broker    │
                                                 └────────────┘
```

**Pros:** Simple to implement, works with any database.
**Cons:** Introduces polling delay, additional database load, harder to scale.

> For a detailed exploration, see [Polling Publisher](./polling-publisher.md).

#### 2. Transaction Log Tailing (CDC)

Instead of polling, a **Change Data Capture (CDC)** tool monitors the database's transaction log (WAL in PostgreSQL, binlog in MySQL) and captures new inserts to the outbox table in near real-time.

```
┌────────────────┐    Read transaction log     ┌────────────┐
│   CDC Tool     │ <─────────────────────────  │  Database  │
│   (Debezium)   │    (WAL / binlog)           │            │
│                │                              └────────────┘
│                │    Publish message            ┌────────────┐
│                │ ────────────────────────────>│  Message   │
└────────────────┘                              │  Broker    │
                                                └────────────┘
```

**Pros:** Near real-time delivery, no polling overhead, no additional queries on the database.
**Cons:** More complex infrastructure (requires CDC tooling), database-specific implementation.

> For a detailed exploration, see [Transaction Log Tailing](./transaction-log-tailing.md).

### Exactly-Once Delivery and Idempotency

The outbox pattern guarantees **at-least-once** delivery to the message broker. However, in failure scenarios (e.g., the relay process crashes after publishing but before marking the event as published), the same event may be published more than once.

To achieve **effectively exactly-once** processing, consumers must be **idempotent**:

- Use the event `id` as a deduplication key.
- Track processed event IDs in the consumer's database.
- Design operations to be naturally idempotent (e.g., "set balance to $100" rather than "add $10 to balance").

```
  Idempotent Consumer Pattern:

  ┌───────────────┐         ┌──────────────────────────┐
  │               │  event  │ 1. Check: Have I seen    │
  │  Message      │────────>│    this event ID before? │
  │  Broker       │         │                          │
  └───────────────┘         │ 2a. YES -> Skip (ack)    │
                            │ 2b. NO  -> Process event │
                            │     + Store event ID     │
                            └──────────────────────────┘
```

---

## 5. Example

### Order Service: Creating an Order

Consider an **Order Service** that needs to create an order and notify other services (Inventory, Payment, Notification) about the new order.

#### Without Transactional Outbox (Dual Write Problem)

```python
def create_order(order_data):
    # Step 1: Save order to database
    order = db.orders.insert(order_data)     # ✅ Succeeds

    # Step 2: Publish event to message broker
    broker.publish("OrderCreated", order)     # ❌ Could fail!

    return order
```

If the broker is unavailable after the database write, the order is created but no event is published. The Inventory Service never reserves stock, the Payment Service never charges the customer, and the system is inconsistent.

#### With Transactional Outbox

```python
def create_order(order_data):
    with db.transaction() as txn:
        # Step 1: Save order to database
        order = txn.execute(
            "INSERT INTO orders (customer_id, total, status) "
            "VALUES (%s, %s, 'CREATED') RETURNING id",
            (order_data.customer_id, order_data.total)
        )

        # Step 2: Write event to outbox table (SAME transaction)
        txn.execute(
            "INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload) "
            "VALUES (%s, %s, %s, %s)",
            (
                "Order",
                order.id,
                "OrderCreated",
                json.dumps({
                    "order_id": order.id,
                    "customer_id": order_data.customer_id,
                    "total": order_data.total,
                    "items": order_data.items,
                    "created_at": datetime.utcnow().isoformat()
                })
            )
        )

    # Transaction committed: both order AND outbox entry are persisted
    return order
```

#### Message Relay Process (Polling Publisher)

```python
def relay_outbox_events():
    while True:
        with db.transaction() as txn:
            # Fetch unpublished events (with row lock to prevent duplicates)
            events = txn.execute(
                "SELECT id, aggregate_type, aggregate_id, event_type, payload "
                "FROM outbox "
                "WHERE published_at IS NULL "
                "ORDER BY created_at "
                "LIMIT 100 "
                "FOR UPDATE SKIP LOCKED"
            )

            for event in events:
                # Publish to message broker
                broker.publish(
                    topic=f"{event.aggregate_type}.{event.event_type}",
                    key=event.aggregate_id,
                    value=event.payload,
                    headers={"event-id": str(event.id)}
                )

                # Mark as published
                txn.execute(
                    "UPDATE outbox SET published_at = NOW() WHERE id = %s",
                    (event.id,)
                )

        time.sleep(1)  # Poll every second
```

#### End-to-End Flow

```
  ┌──────────┐    POST /orders     ┌──────────────┐
  │  Client  │ ──────────────────>│              │
  └──────────┘                     │  Order       │
                                   │  Service     │
                                   │              │
                                   │  BEGIN TXN   │
                                   │  ┌────────┐  │     ┌────────────────┐
                                   │  │ INSERT │  │────>│ orders table   │
                                   │  │ order  │  │     └────────────────┘
                                   │  ├────────┤  │     ┌────────────────┐
                                   │  │ INSERT │  │────>│ outbox table   │
                                   │  │ event  │  │     └────────────────┘
                                   │  └────────┘  │
                                   │  COMMIT TXN  │
                                   └──────────────┘
                                          │
              ┌───────────────────────────┘
              │ (Asynchronous - relay process)
              ▼
  ┌──────────────────┐   Read outbox   ┌────────────────┐
  │  Message Relay   │ <─────────────  │ outbox table   │
  │  (Poller / CDC)  │                 └────────────────┘
  │                  │   Publish
  │                  │ ──────────────> ┌────────────────┐
  └──────────────────┘                 │ Message Broker │
                                       │ (Kafka/RMQ)   │
                                       └───────┬────────┘
                           ┌───────────────────┼────────────────┐
                           ▼                   ▼                ▼
                    ┌────────────┐     ┌────────────┐   ┌────────────┐
                    │ Inventory  │     │  Payment   │   │Notification│
                    │ Service    │     │  Service   │   │  Service   │
                    └────────────┘     └────────────┘   └────────────┘
```

---

## 6. Benefits and Drawbacks

### Benefits

| Benefit | Description |
|---------|-------------|
| **Guaranteed delivery** | Events are persisted in the same transaction as the business data. If the transaction commits, the event is guaranteed to eventually be published. |
| **Atomicity without 2PC** | No need for distributed transactions between the database and the message broker. The local ACID transaction provides the atomicity guarantee. |
| **Simplicity** | The pattern is straightforward to understand and implement. It uses standard database capabilities (transactions, tables). |
| **Broker independence** | The business logic does not depend on the message broker being available at the time of the operation. The broker can be temporarily down. |
| **Auditability** | The outbox table serves as a log of all events produced by the service. It can be queried for debugging and auditing. |
| **Ordering** | Events in the outbox table have a natural ordering (by `created_at`), which can be preserved during publishing. |

### Drawbacks

| Drawback | Description |
|----------|-------------|
| **Eventual consistency** | There is a delay between the database commit and the event being published to the broker. Other services will not see the change immediately. |
| **At-least-once delivery** | The relay process may publish the same event more than once in failure scenarios. Consumers must be idempotent. |
| **Additional table** | The outbox table needs to be managed: schema, indexes, and cleanup of old published events. |
| **Polling overhead** | If using the Polling Publisher approach, frequent polling adds load to the database. If polling is infrequent, latency increases. |
| **CDC complexity** | If using Transaction Log Tailing, you need additional infrastructure (e.g., Debezium, Kafka Connect) and database-specific configuration. |
| **Storage growth** | Without periodic cleanup, the outbox table grows indefinitely. A retention/archival strategy is needed. |

### When to Use

- You need **reliable event publishing** from a service that owns a relational database.
- You are building an [Event-Driven Architecture](../event-driven/event-driven-architecture.md) and cannot afford to lose events.
- You are implementing the [Saga Pattern](../distributed-transactions/saga.md) and need guaranteed delivery of saga commands/events.
- Your [CQRS](../data-patterns/cqrs.md) read models must be kept in sync with the write model.

### When NOT to Use

- Your database does not support ACID transactions (e.g., some NoSQL databases without transaction support).
- You can tolerate occasional event loss (e.g., non-critical analytics events).
- You are using an event store that natively combines data storage and event publishing (e.g., EventStoreDB).

---

## 7. Related Patterns

| Pattern | Relationship |
|---------|-------------|
| [Saga Pattern](../distributed-transactions/saga.md) | Sagas rely on reliable event/command delivery. The Transactional Outbox ensures that saga steps are not lost. |
| [Event-Driven Architecture](../event-driven/event-driven-architecture.md) | The Outbox pattern is a key enabler of reliable event-driven communication between services. |
| [CQRS](../data-patterns/cqrs.md) | CQRS read model updates often rely on events published via the Outbox pattern. |
| [Polling Publisher](./polling-publisher.md) | One of the two mechanisms for delivering outbox events to the message broker. Uses periodic database polling. |
| [Transaction Log Tailing](./transaction-log-tailing.md) | The other delivery mechanism. Uses CDC to capture outbox inserts from the database transaction log. |
| [Domain Event](../event-driven/domain-event.md) | Domain Events are the typical content stored in the outbox table. The outbox ensures their reliable publication. |
| [Two-Phase Commit (2PC)](../distributed-transactions/two-phase-commit.md) | The alternative (impractical) approach for atomicity across DB and broker. The Outbox pattern avoids 2PC. |

---

## 8. Real-World Usage

### Debezium + Kafka (CDC Approach)

[Debezium](https://debezium.io) is the most popular open-source CDC platform for implementing the Transactional Outbox pattern. It provides a dedicated **Outbox Event Router** that:

- Monitors the outbox table via the database's transaction log.
- Automatically routes events to the correct Kafka topic based on `aggregate_type`.
- Deletes processed outbox records (optional) to prevent table growth.
- Supports PostgreSQL, MySQL, SQL Server, MongoDB, and more.

```
┌─────────────┐     WAL/binlog     ┌──────────────┐    Kafka    ┌──────────┐
│  PostgreSQL │ ─────────────────> │   Debezium   │ ──────────> │  Kafka   │
│  (outbox    │                    │   Connector  │             │  Topics  │
│   table)    │                    └──────────────┘             └──────────┘
└─────────────┘
```

### Industry Adoption

| Company / Project | Approach | Notes |
|-------------------|----------|-------|
| **Debezium** | CDC (log tailing) | Dedicated outbox event router for Kafka Connect |
| **MassTransit** (.NET) | Polling + CDC | Built-in transactional outbox support |
| **Axon Framework** (Java) | Polling | Native outbox support for event-driven CQRS apps |
| **NServiceBus** (.NET) | Polling | Outbox feature to deduplicate and guarantee delivery |
| **Eventuate Tram** (Java) | CDC (Debezium) | Microservices framework with built-in outbox pattern |
| **Amazon EventBridge** | Managed | AWS service that can act as a managed outbox relay |

### Cleanup Strategies

Over time, the outbox table will accumulate published events. Common cleanup strategies include:

1. **Periodic deletion**: A scheduled job deletes rows where `published_at` is older than a retention period (e.g., 7 days).
2. **Partitioned table**: Use time-based table partitioning to efficiently drop old partitions.
3. **Delete on publish**: The relay process deletes the outbox row immediately after successful publishing (trades auditability for simplicity).

---

## 9. Summary

The **Transactional Outbox** pattern is a fundamental technique for ensuring reliable event publication in microservices architectures. It solves the **dual write problem** by writing events to an outbox table within the same database transaction as the business data change, and then using a separate relay process to publish those events to the message broker.

```
  ┌─────────────────────────────────────────────────────────┐
  │               Transactional Outbox Summary              │
  ├─────────────────────────────────────────────────────────┤
  │                                                         │
  │  Problem:  Dual write — DB + Broker can't be atomic     │
  │                                                         │
  │  Solution: Write event to outbox table in same DB       │
  │            transaction as business data                  │
  │                                                         │
  │  Delivery: Polling Publisher  OR  CDC (Log Tailing)     │
  │                                                         │
  │  Guarantee: At-least-once (consumers must be idempotent)│
  │                                                         │
  │  Key Tools: Debezium, Kafka Connect, MassTransit,       │
  │             Axon Framework, Eventuate Tram               │
  │                                                         │
  └─────────────────────────────────────────────────────────┘
```

**Key takeaways:**

1. Never perform dual writes (DB update + message publish) without a coordination mechanism.
2. The outbox table acts as a reliable buffer between your service and the message broker.
3. Choose between Polling Publisher (simpler) and Transaction Log Tailing (lower latency) based on your needs.
4. Always design consumers to be idempotent, as at-least-once delivery is inherent to this pattern.
5. Plan for outbox table cleanup to prevent unbounded storage growth.

---

**See also:**
- [Saga Pattern](../distributed-transactions/saga.md) - Coordinating distributed transactions using the outbox for reliable messaging
- [Event-Driven Architecture](../event-driven/event-driven-architecture.md) - The broader architectural style enabled by reliable event publishing
- [RabbitMQ vs Kafka](./rabbitmq-vs-kafka.md) - Choosing the right message broker for your outbox relay
- [Two-Phase Commit (2PC)](../distributed-transactions/two-phase-commit.md) - The alternative approach that the outbox pattern replaces
