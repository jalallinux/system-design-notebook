# Polling Publisher

## 1. Introduction

In a microservices architecture, reliably publishing domain events to a message broker is a fundamental challenge. The **Polling Publisher** pattern provides a straightforward mechanism to deliver events from an outbox table to a message broker by periodically querying the database for unpublished events.

This pattern is typically used as the **delivery mechanism** for the [Transactional Outbox](./transactional-outbox.md) pattern. While the Transactional Outbox ensures that events are atomically written alongside business data, the Polling Publisher is responsible for the second step: getting those events from the database into the message broker (e.g., [Kafka or RabbitMQ](./rabbitmq-vs-kafka.md)).

The Polling Publisher is valued for its **simplicity** and **database-agnostic** nature. It requires no special database features or infrastructure beyond a background process and a relational (or NoSQL) database.

```
+------------------+       +------------------+       +------------------+
|                  |       |                  |       |                  |
|   Application    | ----> |   Outbox Table   | ----> | Message Broker   |
|   (writes to     |       |   (in database)  |       | (Kafka, RabbitMQ |
|    outbox)       |       |                  |       |  etc.)           |
|                  |       |                  |       |                  |
+------------------+       +--------+---------+       +------------------+
                                    ^
                                    |
                           +--------+---------+
                           |                  |
                           | Polling Publisher |
                           | (background job) |
                           |                  |
                           +------------------+
```

## 2. Context and Problem

When using the [Transactional Outbox](./transactional-outbox.md) pattern, the application writes domain events to an outbox table in the same database transaction as the business data change. This guarantees atomicity — either both the business data and the event are persisted, or neither is.

However, events sitting in an outbox table are useless until they reach the message broker where downstream consumers can process them. The core problem is:

> **How do we reliably transfer events from the outbox table to the message broker?**

There are two primary approaches to solve this:

1. **Polling Publisher** (this pattern) — a background process periodically queries the outbox table
2. **[Transaction Log Tailing](./transaction-log-tailing.md)** — reads the database transaction log (CDC) to capture new events

Each approach has different trade-offs in terms of complexity, latency, and infrastructure requirements.

## 3. Forces

Several forces influence the decision to use the Polling Publisher pattern:

- **Simplicity over real-time**: The system can tolerate a small delay (tens to hundreds of milliseconds) between writing an event and publishing it to the broker. Absolute real-time delivery is not required.

- **Database-agnostic approach**: The solution must work with any database that supports basic SQL queries (SELECT, UPDATE, DELETE). No dependency on database-specific features like change data capture (CDC), logical replication slots, or transaction log access.

- **No special infrastructure needed**: The team does not want to deploy or manage additional infrastructure like Debezium, Maxwell, or other CDC connectors. A simple background process is sufficient.

- **Operational simplicity**: The team prefers a solution that is easy to understand, debug, and operate. Anyone who can read SQL can understand the Polling Publisher.

- **Consistency requirements**: Events must be delivered at least once. Duplicate delivery is acceptable if consumers are idempotent, but events must never be lost.

- **Ordering constraints**: Events for the same aggregate (e.g., the same order) must be published in the order they were created.

## 4. Solution

The Polling Publisher pattern uses a **background process** (a scheduled job, cron task, or dedicated worker) that runs at regular intervals and performs the following cycle:

### Polling Cycle

```
    +------------------+
    |  Wait for next   |
    |  poll interval   |
    +--------+---------+
             |
             v
    +------------------+
    |  Query outbox    |
    |  for unpublished |
    |  events          |
    +--------+---------+
             |
             v
    +------------------+
    |  Any events      |-----> No -----> (back to wait)
    |  found?          |
    +--------+---------+
             |
            Yes
             |
             v
    +------------------+
    |  Publish batch   |
    |  to message      |
    |  broker          |
    +--------+---------+
             |
             v
    +------------------+
    |  Mark events as  |
    |  published in    |
    |  outbox          |
    +--------+---------+
             |
             v
        (back to wait)
```

### Polling Algorithm

The core algorithm consists of the following steps:

**Step 1: Query unpublished events**

```sql
SELECT id, aggregate_type, aggregate_id, event_type, payload, created_at
FROM outbox
WHERE published = FALSE
ORDER BY created_at ASC
LIMIT 100;
```

The `ORDER BY created_at ASC` ensures events are processed in the order they were created. The `LIMIT` clause controls the batch size to avoid overwhelming the broker or consuming too much memory.

**Step 2: Publish to the message broker**

For each event (or as a batch), publish the event payload to the appropriate topic or queue in the message broker.

```
For each event in batch:
    topic = derive_topic(event.aggregate_type, event.event_type)
    key   = event.aggregate_id
    value = event.payload
    publish(topic, key, value)
```

Using the `aggregate_id` as the message key ensures that events for the same aggregate land on the same partition (in Kafka), preserving ordering.

**Step 3: Mark events as published**

```sql
UPDATE outbox
SET published = TRUE, published_at = NOW()
WHERE id IN (?, ?, ?, ...);
```

### Outbox Table Schema

A typical outbox table looks like this:

```sql
CREATE TABLE outbox (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    aggregate_type  VARCHAR(255) NOT NULL,
    aggregate_id    VARCHAR(255) NOT NULL,
    event_type      VARCHAR(255) NOT NULL,
    payload         TEXT NOT NULL,              -- JSON event payload
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    published       BOOLEAN NOT NULL DEFAULT FALSE,
    published_at    TIMESTAMP NULL,
    retry_count     INT NOT NULL DEFAULT 0,
    idempotency_key VARCHAR(255) UNIQUE NOT NULL
);

CREATE INDEX idx_outbox_unpublished ON outbox (published, created_at);
```

### Event State Machine

Each event in the outbox transitions through the following states:

```
  +-------------+       +-----------+       +-------------+
  |             |       |           |       |             |
  |   PENDING   | ----> | PUBLISHING| ----> |  PUBLISHED  |
  |             |       |           |       |             |
  +------+------+       +-----+-----+       +-------------+
         |                     |
         |                     | (failure)
         |                     v
         |              +------+------+
         |              |             |
         +------------> |   FAILED    | ---> (retry -> PUBLISHING)
                        |             |
                        +------+------+
                               |
                               | (max retries exceeded)
                               v
                        +------+------+
                        |             |
                        | DEAD LETTER |
                        |             |
                        +-------------+
```

| State       | Description                                          |
|-------------|------------------------------------------------------|
| PENDING     | Event written to outbox, not yet picked up           |
| PUBLISHING  | Event picked up by the poller, being sent to broker  |
| PUBLISHED   | Successfully published to the message broker         |
| FAILED      | Publication failed, will be retried                  |
| DEAD LETTER | Max retries exceeded, requires manual intervention   |

### Handling Failures

Failures can occur at multiple points:

1. **Broker unavailable**: The message broker is temporarily down. The poller should catch the exception, increment the `retry_count`, and leave the event as unpublished for the next polling cycle.

2. **Partial batch failure**: Some events in a batch publish successfully, others fail. Mark only the successful ones as published.

3. **Duplicate publishing**: If the poller publishes an event but crashes before marking it as published, the next poll will re-publish the same event. This is why **consumers must be idempotent**. The `idempotency_key` in the outbox table helps downstream consumers detect and skip duplicates.

4. **Dead letter handling**: After a configurable number of retries (e.g., 5), move the event to a dead letter state and alert the operations team for manual investigation.

```
Retry Strategy:
    Attempt 1: immediate
    Attempt 2: wait 1 second
    Attempt 3: wait 5 seconds
    Attempt 4: wait 30 seconds
    Attempt 5: wait 2 minutes
    After 5 attempts: mark as DEAD LETTER, alert ops team
```

### Ordering Guarantees

Events for the **same aggregate** must be published in order. The Polling Publisher ensures this by:

1. Querying events ordered by `created_at ASC`
2. Using `aggregate_id` as the message key (ensures same-partition delivery in Kafka)
3. Processing events sequentially per aggregate within each batch

> **Important**: If an event for aggregate X fails to publish, all subsequent events for aggregate X must be blocked until the failed event is successfully published. Otherwise, consumers would see events out of order.

### Tuning the Poll Interval

The poll interval is the trade-off knob between **latency** and **database load**:

| Poll Interval | Latency         | Database Load | Use Case                        |
|---------------|-----------------|---------------|----------------------------------|
| 10ms          | Very low (~10ms)| Very high     | Near real-time requirements      |
| 100ms         | Low (~100ms)    | Moderate      | Most microservices (recommended) |
| 1 second      | Medium (~1s)    | Low           | Non-critical event delivery      |
| 5 seconds     | High (~5s)      | Very low      | Batch-oriented or low-priority   |

For most systems, a poll interval of **100ms** provides a good balance.

### Cleanup Strategy

Published events should be periodically removed from the outbox to prevent unbounded table growth:

```sql
-- Delete published events older than 7 days
DELETE FROM outbox
WHERE published = TRUE
  AND published_at < NOW() - INTERVAL 7 DAY
LIMIT 10000;
```

Run this cleanup as a separate scheduled job (e.g., once per hour or once per day). Use `LIMIT` to avoid long-running delete operations that could lock the table.

## 5. Example

Consider an e-commerce system where placing an order must publish an `OrderPlaced` event to Kafka so that the payment, inventory, and notification services can react.

### Application Code (Writing to Outbox)

```python
def place_order(order_data):
    with database.transaction():
        order = Order.create(order_data)

        outbox_event = OutboxEvent(
            aggregate_type="Order",
            aggregate_id=str(order.id),
            event_type="OrderPlaced",
            payload=json.dumps({
                "order_id": order.id,
                "customer_id": order.customer_id,
                "total": order.total,
                "items": order.items
            }),
            idempotency_key=f"order-placed-{order.id}"
        )
        outbox_event.save()

    return order
```

### Polling Publisher (Background Job)

```python
class PollingPublisher:
    def __init__(self, db, kafka_producer, batch_size=100, poll_interval_ms=100):
        self.db = db
        self.kafka = kafka_producer
        self.batch_size = batch_size
        self.poll_interval = poll_interval_ms / 1000.0  # convert to seconds
        self.max_retries = 5

    def run(self):
        """Main polling loop."""
        while True:
            events = self.fetch_unpublished_events()

            if events:
                self.publish_batch(events)

            time.sleep(self.poll_interval)

    def fetch_unpublished_events(self):
        return self.db.execute("""
            SELECT id, aggregate_type, aggregate_id, event_type,
                   payload, created_at, idempotency_key
            FROM outbox
            WHERE published = FALSE AND retry_count < %s
            ORDER BY created_at ASC
            LIMIT %s
        """, [self.max_retries, self.batch_size])

    def publish_batch(self, events):
        published_ids = []
        failed_ids = []

        for event in events:
            try:
                topic = f"{event.aggregate_type.lower()}.events"
                self.kafka.produce(
                    topic=topic,
                    key=event.aggregate_id,
                    value=event.payload,
                    headers={"idempotency_key": event.idempotency_key}
                )
                published_ids.append(event.id)
            except Exception as e:
                logger.error(f"Failed to publish event {event.id}: {e}")
                failed_ids.append(event.id)

        # Flush to ensure all messages are delivered
        self.kafka.flush()

        if published_ids:
            self.db.execute("""
                UPDATE outbox
                SET published = TRUE, published_at = NOW()
                WHERE id IN (%s)
            """, published_ids)

        if failed_ids:
            self.db.execute("""
                UPDATE outbox
                SET retry_count = retry_count + 1
                WHERE id IN (%s)
            """, failed_ids)
```

### Execution Timeline

```
Time (ms)  |  Action
-----------+-------------------------------------------------------
  0        |  Poller wakes up
  5        |  SELECT 3 unpublished events from outbox
  10       |  Publish event #1 (OrderPlaced) to Kafka -> success
  15       |  Publish event #2 (OrderUpdated) to Kafka -> success
  20       |  Publish event #3 (OrderCancelled) to Kafka -> FAIL
  25       |  UPDATE events #1, #2 as published
  26       |  UPDATE event #3: retry_count = retry_count + 1
  30       |  Poller sleeps for 100ms
  ...      |  ...
  130      |  Poller wakes up
  135      |  SELECT unpublished events (includes event #3 again)
  140      |  Publish event #3 (OrderCancelled) to Kafka -> success
  145      |  UPDATE event #3 as published
  150      |  Poller sleeps for 100ms
```

## 6. Benefits and Drawbacks

### Benefits

| Benefit                        | Description                                                                                          |
|--------------------------------|------------------------------------------------------------------------------------------------------|
| Simple to implement            | Only requires SQL queries and a background loop. No complex infrastructure.                          |
| Database-agnostic              | Works with any database: PostgreSQL, MySQL, MongoDB, SQL Server, etc.                                |
| No CDC infrastructure needed   | Unlike [Transaction Log Tailing](./transaction-log-tailing.md), no Debezium, Maxwell, or similar tools required. |
| Easy to understand             | The entire mechanism can be explained with a SELECT, a publish call, and an UPDATE.                  |
| Easy to debug                  | Query the outbox table directly to see pending, published, or failed events.                         |
| Portable                       | No vendor lock-in to a specific database or CDC tool.                                                |
| Reliable at-least-once delivery| Combined with consumer idempotency, guarantees no events are lost.                                   |

### Drawbacks

| Drawback                       | Description                                                                                          |
|--------------------------------|------------------------------------------------------------------------------------------------------|
| Polling overhead               | Frequent queries against the database even when there are no new events.                             |
| Higher latency than CDC        | Events are delivered with a delay equal to the poll interval (typically 100ms-5s).                    |
| Database load                  | Frequent polling adds read load to the database, which can be significant at scale.                  |
| Scaling challenges             | Running multiple pollers requires coordination (e.g., locking) to avoid duplicate publishing.        |
| Table growth                   | Without cleanup, the outbox table grows indefinitely, degrading query performance.                   |
| Not truly real-time            | The poll interval introduces inherent latency that CDC-based approaches avoid.                       |

### Polling Publisher vs Transaction Log Tailing

| Aspect                  | Polling Publisher                         | Transaction Log Tailing (CDC)                  |
|-------------------------|-------------------------------------------|------------------------------------------------|
| Latency                 | Poll interval (100ms - seconds)           | Near real-time (milliseconds)                  |
| Infrastructure          | None (just a background process)          | Requires CDC tool (Debezium, Maxwell, etc.)    |
| Database compatibility  | Any SQL/NoSQL database                    | Depends on CDC tool support                    |
| Database load           | Read queries on every poll cycle          | Reads transaction log (minimal table impact)   |
| Complexity              | Very low                                  | Moderate to high                               |
| Scaling                 | Needs coordination for multiple pollers   | CDC connector handles scaling                  |
| Debugging               | Easy (query the outbox table)             | Harder (inspect CDC connector logs/offsets)    |
| Operational overhead    | Low                                       | Medium to high                                 |
| Best for                | Simpler setups, small to medium scale     | High-throughput, low-latency requirements      |

## 7. Related Patterns

- **[Transactional Outbox](./transactional-outbox.md)** — The pattern that the Polling Publisher complements. The outbox ensures atomic event persistence; the Polling Publisher delivers those events to the broker.

- **[Transaction Log Tailing](./transaction-log-tailing.md)** — The alternative delivery mechanism for the Transactional Outbox pattern. Uses CDC (Change Data Capture) to read new events from the database transaction log instead of polling.

- **[Event-Driven Architecture](../event-driven/event-driven-architecture.md)** — The broader architectural style that relies on events being reliably published and consumed across services.

- **[Saga Pattern](../distributed-transactions/saga.md)** — Sagas depend on reliable event publishing to coordinate distributed transactions. The Polling Publisher can serve as the delivery mechanism for saga events.

- **[CQRS](../data-patterns/cqrs.md)** — CQRS read models are often updated by consuming domain events. The Polling Publisher ensures those events reach the message broker for read model synchronization.

- **[RabbitMQ vs Kafka](./rabbitmq-vs-kafka.md)** — Understanding the target message broker helps in configuring the Polling Publisher (batch sizes, acknowledgments, partitioning).

- **[Microservices Architecture](../architecture/microservices.md)** — The Polling Publisher is a key infrastructure pattern in microservices for reliable inter-service communication.

## 8. Real-World Usage

The Polling Publisher pattern is widely used in production systems, especially in organizations that prefer simplicity over the operational complexity of CDC tools:

- **Early-stage startups**: Teams with limited infrastructure expertise often start with a Polling Publisher because it requires no additional tools beyond the application database and a message broker.

- **Monolith-to-microservices migrations**: When extracting services from a monolith, the Polling Publisher provides a low-risk way to start publishing domain events without introducing CDC infrastructure.

- **Enterprise applications with strict database policies**: In organizations where access to database transaction logs is restricted by DBAs, the Polling Publisher works purely through standard SQL queries.

- **Frameworks and libraries**:
  - **Axon Framework** (Java) supports a Polling Publisher as one of its event publishing strategies.
  - **MassTransit** (.NET) provides outbox support with a built-in polling mechanism.
  - **Laravel** (PHP) developers commonly implement polling-based outbox publishing using scheduled commands or queue workers.

- **Common progression**: Many teams start with a Polling Publisher for simplicity and later migrate to [Transaction Log Tailing](./transaction-log-tailing.md) (e.g., Debezium) when they need lower latency or higher throughput.

### Scaling the Polling Publisher

For high-throughput systems, a single poller may become a bottleneck. Common strategies include:

```
Strategy 1: Partitioned Polling (recommended)
+-------------------------------------------+
| Poller 1: WHERE aggregate_id % 3 = 0      |
| Poller 2: WHERE aggregate_id % 3 = 1      |
| Poller 3: WHERE aggregate_id % 3 = 2      |
+-------------------------------------------+
Each poller handles a subset of aggregates.
No coordination needed. Ordering preserved per aggregate.

Strategy 2: Leader Election
+-------------------------------------------+
| Only the leader poller runs at any time.   |
| If leader fails, another takes over.       |
| Uses distributed lock (Redis, ZooKeeper).  |
+-------------------------------------------+
Simple but limits throughput to a single poller.

Strategy 3: Database Row Locking
+-------------------------------------------+
| SELECT ... FOR UPDATE SKIP LOCKED          |
| Multiple pollers compete for rows.         |
| Database handles coordination.             |
+-------------------------------------------+
Good for moderate scale but adds DB lock contention.
```

## 9. Summary

The **Polling Publisher** is a simple, pragmatic pattern for delivering events from an outbox table to a message broker. It serves as the delivery mechanism for the [Transactional Outbox](./transactional-outbox.md) pattern.

**Key takeaways:**

1. A background process periodically polls the outbox table for unpublished events
2. Events are published to the message broker and marked as published in the database
3. The pattern is **database-agnostic** and requires **no special infrastructure**
4. It trades **higher latency** for **simplicity** compared to [Transaction Log Tailing](./transaction-log-tailing.md)
5. Consumers must be **idempotent** because at-least-once delivery can produce duplicates
6. The poll interval is the primary tuning knob: lower interval means lower latency but higher database load
7. Published events should be periodically cleaned up to prevent table bloat
8. For scaling, use partitioned polling or database row locking strategies

The Polling Publisher is an excellent starting point for teams adopting the [Transactional Outbox](./transactional-outbox.md) pattern, and can be replaced with [Transaction Log Tailing](./transaction-log-tailing.md) when lower latency or higher throughput is needed.

---

**See also:** [Transactional Outbox](./transactional-outbox.md) | [Transaction Log Tailing](./transaction-log-tailing.md) | [Event-Driven Architecture](../event-driven/event-driven-architecture.md) | [RabbitMQ vs Kafka](./rabbitmq-vs-kafka.md)
