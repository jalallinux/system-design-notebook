# Idempotent Consumer

## 1. Introduction

The Idempotent Consumer pattern ensures that a message handler can safely process the same message multiple times, always producing the same result regardless of how many times the message is delivered. In distributed systems built on [messaging](./messaging.md), achieving exactly-once processing is extremely difficult. Message brokers typically guarantee **at-least-once delivery**, which means consumers must be prepared to handle duplicates gracefully.

**Origin and Context:**

The term "idempotent" comes from mathematics, where an operation is idempotent if applying it multiple times yields the same result as applying it once. In the context of messaging, an idempotent consumer is one that can receive and process the same message two, three, or a hundred times without causing incorrect side effects such as double charges, duplicate orders, or repeated notifications.

**Why It Matters:**

In a [microservices architecture](../architecture/microservices.md), services communicate asynchronously through message brokers like [RabbitMQ or Kafka](../messaging/rabbitmq-vs-kafka.md). Network failures, consumer crashes, broker rebalancing, and retry mechanisms all contribute to the possibility of duplicate message delivery. Without idempotent consumers, these duplicates can lead to data corruption and business logic errors that are difficult to detect and resolve.

**Core Principle:**

Design message handlers so that processing the same message more than once has exactly the same effect as processing it once. This can be achieved through deduplication tracking, naturally idempotent operations, or client-provided idempotency keys.

---

## 2. Context and Problem

### The Duplicate Delivery Scenario

Consider an e-commerce platform where a payment service processes orders through an event-driven pipeline:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Order      │────▶│   Message    │────▶│   Payment    │
│   Service    │     │   Broker     │     │   Service    │
└──────────────┘     └──────────────┘     └──────────────┘
                           │
                    Message: PaymentRequested
                    {orderId: 1234, amount: $99.99}
```

**What can go wrong:**

1. The Payment Service receives the `PaymentRequested` message and begins processing
2. It successfully charges the customer's credit card
3. Before it can acknowledge (ACK) the message, the service crashes or the network connection drops
4. The message broker, having not received an ACK, redelivers the same message
5. The Payment Service processes the message again, resulting in a **double charge**

```
Timeline of Duplicate Delivery:

  Order Service          Message Broker         Payment Service
       │                      │                       │
       │  PaymentRequested    │                       │
       ├─────────────────────▶│                       │
       │                      │  Deliver Message      │
       │                      ├──────────────────────▶│
       │                      │                       │
       │                      │              Process: Charge $99.99
       │                      │                       │
       │                      │                  ╳ CRASH / No ACK
       │                      │                       │
       │                      │  Redeliver Message    │
       │                      ├──────────────────────▶│
       │                      │                       │
       │                      │              Process: Charge $99.99
       │                      │                  (DUPLICATE!)
       │                      │                       │
       │                      │         ACK           │
       │                      │◀──────────────────────┤
       │                      │                       │
```

### Common Causes of Duplicate Messages

| Cause | Description | Frequency |
|-------|-------------|-----------|
| **Consumer crash** | Consumer processes message but crashes before ACK | Occasional |
| **Network timeout** | ACK is sent but lost in transit; broker redelivers | Occasional |
| **Broker rebalancing** | Kafka partition rebalancing causes re-delivery | During scaling |
| **Producer retry** | Producer retries after timeout, sending duplicate | Occasional |
| **Infrastructure restart** | Broker or consumer restarts mid-processing | During deployments |
| **Manual replay** | Operations team replays messages for recovery | Intentional |

### The Business Impact

Without idempotent consumers, duplicate processing can cause:

- **Financial loss**: Double charges, duplicate refunds, incorrect account balances
- **Data corruption**: Duplicate records, incorrect counters, inconsistent state
- **Customer dissatisfaction**: Duplicate orders, repeated notifications, multiple shipments
- **Compliance violations**: Duplicate financial transactions may violate regulatory requirements

---

## 3. Forces

The following forces shape the need for idempotent consumers:

1. **At-least-once delivery is the norm**: Most message brokers (Kafka, RabbitMQ, SQS) guarantee at-least-once delivery. Exactly-once delivery is either impossible or extremely expensive to achieve across distributed systems.

2. **Business operations must not be repeated incorrectly**: Charging a credit card, placing an order, or sending a notification must happen exactly once from the business perspective, even if the underlying message is delivered multiple times.

3. **Network is unreliable**: According to the [CAP theorem](../fundamentals/cap-theorem.md), network partitions are inevitable. Messages, acknowledgements, and responses can be lost or delayed.

4. **Performance matters**: The deduplication mechanism must not become a bottleneck. High-throughput systems process millions of messages per second.

5. **Storage is finite**: Tracking every processed message ID forever is not practical. A cleanup strategy is needed.

6. **Ordering may not be guaranteed**: Messages may arrive out of order, making simple sequence-based deduplication insufficient.

---

## 4. Solution

Implement message handlers that can safely process the same message multiple times without causing incorrect side effects. There are several strategies to achieve this, each suited to different scenarios.

### Strategy 1: Message Deduplication (Processed Message Tracking)

Track processed message IDs in a database. Before processing a new message, check if its ID has already been recorded. If so, skip processing and return the previously computed result.

```
Idempotent Consumer Flow (Deduplication):

  Message Arrives
       │
       ▼
  ┌─────────────────────────┐
  │  Look up message_id in  │
  │  processed_messages DB  │
  └────────────┬────────────┘
               │
        ┌──────┴──────┐
        │  Already    │
        │  processed? │
        └──────┬──────┘
               │
       ┌───────┴───────┐
       │               │
      Yes              No
       │               │
       ▼               ▼
  ┌──────────┐   ┌──────────────────────────┐
  │  Return   │   │  BEGIN TRANSACTION       │
  │  cached   │   │                          │
  │  result   │   │  1. Insert message_id    │
  │  (skip)   │   │     into processed_msgs  │
  └──────────┘   │                          │
                  │  2. Execute business     │
                  │     logic                │
                  │                          │
                  │  3. Store result         │
                  │                          │
                  │  COMMIT TRANSACTION      │
                  └──────────────────────────┘
```

**Message Tracking Table:**

```sql
CREATE TABLE processed_messages (
    message_id   VARCHAR(255) PRIMARY KEY,
    processed_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    result       JSONB,
    expires_at   TIMESTAMP    NOT NULL
);

CREATE INDEX idx_processed_messages_expires
    ON processed_messages (expires_at);
```

**Pseudocode:**

```python
def handle_message(message):
    message_id = message.id

    # Check if already processed
    existing = db.query(
        "SELECT result FROM processed_messages WHERE message_id = ?",
        message_id
    )

    if existing:
        log.info(f"Duplicate message {message_id}, returning cached result")
        return existing.result

    # Process within a transaction
    with db.transaction():
        # Insert message ID first (acts as a lock)
        db.execute(
            "INSERT INTO processed_messages (message_id, result, expires_at) "
            "VALUES (?, NULL, NOW() + INTERVAL '7 days')",
            message_id
        )

        # Execute business logic
        result = process_business_logic(message)

        # Update with result
        db.execute(
            "UPDATE processed_messages SET result = ? WHERE message_id = ?",
            json.dumps(result), message_id
        )

    return result
```

### Strategy 2: Natural Idempotency

Design operations to be inherently idempotent by using absolute values instead of relative changes.

```
NON-IDEMPOTENT (Dangerous):
  UPDATE accounts SET balance = balance + 100 WHERE id = 123;
  -- Executing twice: balance increases by 200 (WRONG!)

IDEMPOTENT (Safe):
  UPDATE accounts SET balance = 500 WHERE id = 123 AND version = 4;
  -- Executing twice: balance is 500 both times (CORRECT)

NON-IDEMPOTENT:
  INSERT INTO orders (customer_id, product_id, quantity) VALUES (1, 42, 2);
  -- Executing twice: two duplicate orders created (WRONG!)

IDEMPOTENT:
  INSERT INTO orders (order_id, customer_id, product_id, quantity)
  VALUES ('ord-abc-123', 1, 42, 2)
  ON CONFLICT (order_id) DO NOTHING;
  -- Executing twice: only one order exists (CORRECT)
```

### Strategy 3: Idempotency Keys

The client generates a unique key for each logical operation. The server uses this key to detect and prevent duplicate processing.

```
Idempotency Key Flow:

  Client                       Server
    │                            │
    │  POST /payments            │
    │  Idempotency-Key: ik_abc1  │
    │  {amount: 99.99}           │
    ├───────────────────────────▶│
    │                            │
    │                   ┌────────┴────────┐
    │                   │ Check ik_abc1   │
    │                   │ in key store    │
    │                   └────────┬────────┘
    │                            │
    │                   Key not found:
    │                   Process payment,
    │                   store result with key
    │                            │
    │  201 Created               │
    │  {payment_id: pay_xyz}     │
    │◀───────────────────────────┤
    │                            │
    │  POST /payments (retry)    │
    │  Idempotency-Key: ik_abc1  │
    │  {amount: 99.99}           │
    ├───────────────────────────▶│
    │                            │
    │                   ┌────────┴────────┐
    │                   │ Check ik_abc1   │
    │                   │ FOUND! Return   │
    │                   │ cached response │
    │                   └────────┬────────┘
    │                            │
    │  201 Created (cached)      │
    │  {payment_id: pay_xyz}     │
    │◀───────────────────────────┤
```

### Strategy 4: Optimistic Locking with Version Numbers

Use version numbers on entities to detect and reject duplicate updates.

```python
def handle_update_message(message):
    entity = db.find(message.entity_id)

    if entity.version != message.expected_version:
        log.info("Version mismatch, message already applied or stale")
        return  # Skip processing

    entity.apply_changes(message.data)
    entity.version += 1
    db.save(entity)  # Fails if version changed concurrently
```

### Strategy 5: Database Constraints

Use unique constraints to prevent duplicate inserts at the database level.

```sql
-- Unique constraint prevents duplicate processing
CREATE TABLE order_events (
    event_id    VARCHAR(255) PRIMARY KEY,  -- message ID as PK
    order_id    VARCHAR(255) NOT NULL,
    event_type  VARCHAR(100) NOT NULL,
    payload     JSONB        NOT NULL,
    created_at  TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uq_order_event UNIQUE (order_id, event_type)
);
```

```python
def handle_order_event(message):
    try:
        db.execute(
            "INSERT INTO order_events (event_id, order_id, event_type, payload) "
            "VALUES (?, ?, ?, ?)",
            message.id, message.order_id, message.event_type, message.payload
        )
        # Process business logic only on successful insert
        process_order_event(message)
    except UniqueConstraintViolation:
        log.info(f"Duplicate event {message.id}, skipping")
```

### Cleanup Strategy for Processed Message IDs

Storing every message ID indefinitely is not practical. Implement a TTL-based cleanup:

```
Cleanup Approaches:

┌────────────────────────────────────────────────────────┐
│               Message ID Cleanup                       │
├────────────────────────────────────────────────────────┤
│                                                        │
│  1. TTL-Based Expiry                                   │
│     - Set expires_at = processed_at + 7 days           │
│     - Cron job deletes expired records                 │
│     - Balance: longer TTL = more storage, safer        │
│                                                        │
│  2. Sliding Window                                     │
│     - Keep only last N hours of message IDs            │
│     - Partition table by time for efficient cleanup    │
│                                                        │
│  3. Redis with TTL                                     │
│     - Store message_id in Redis with auto-expiry       │
│     - SET message:{id} "processed" EX 604800           │
│     - Fast lookups, automatic cleanup                  │
│                                                        │
│  4. Compacted Log (Kafka)                              │
│     - Use Kafka compacted topic for processed IDs      │
│     - Automatic cleanup via log compaction             │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Periodic Cleanup Job:**

```sql
-- Run daily via cron
DELETE FROM processed_messages
WHERE expires_at < CURRENT_TIMESTAMP;
```

**Redis-Based Tracking:**

```python
def is_processed(message_id):
    return redis.exists(f"processed:{message_id}")

def mark_processed(message_id, ttl_seconds=604800):  # 7 days
    redis.set(f"processed:{message_id}", "1", ex=ttl_seconds)
```

---

## 5. Example

### Example 1: Payment Processing with Deduplication

A payment service receives `PaymentRequested` events. The same event can be delivered twice due to broker retries or consumer crashes.

```
Scenario: Duplicate PaymentRequested Event

  Event: PaymentRequested
  {
    "messageId":  "msg-7f3a-4b2c",
    "orderId":    "ord-1234",
    "customerId": "cust-5678",
    "amount":     99.99,
    "currency":   "USD"
  }
```

**First delivery (normal processing):**

```
  Message Broker              Payment Service              Database
       │                           │                          │
       │  msg-7f3a-4b2c            │                          │
       ├──────────────────────────▶│                          │
       │                           │  SELECT * FROM           │
       │                           │  processed_messages      │
       │                           │  WHERE id = 'msg-7f3a'   │
       │                           ├─────────────────────────▶│
       │                           │                          │
       │                           │  Result: NOT FOUND       │
       │                           │◀─────────────────────────┤
       │                           │                          │
       │                           │  BEGIN TRANSACTION       │
       │                           │  1. INSERT processed_msg │
       │                           │  2. Charge credit card   │
       │                           │  3. INSERT payment       │
       │                           │  COMMIT                  │
       │                           ├─────────────────────────▶│
       │                           │                          │
       │         ACK               │  Committed               │
       │◀──────────────────────────┤◀─────────────────────────┤
```

**Second delivery (duplicate, safely handled):**

```
  Message Broker              Payment Service              Database
       │                           │                          │
       │  msg-7f3a-4b2c (retry)    │                          │
       ├──────────────────────────▶│                          │
       │                           │  SELECT * FROM           │
       │                           │  processed_messages      │
       │                           │  WHERE id = 'msg-7f3a'   │
       │                           ├─────────────────────────▶│
       │                           │                          │
       │                           │  Result: FOUND           │
       │                           │  (already processed)     │
       │                           │◀─────────────────────────┤
       │                           │                          │
       │         ACK               │  Skip processing         │
       │◀──────────────────────────┤  Log: "Duplicate msg"    │
       │                           │                          │
```

**Result:** The customer is charged exactly once, even though the message was delivered twice.

### Example 2: Order Creation with Idempotency Key

A client creates an order using an idempotency key to prevent duplicate orders during retries.

```python
# Client side
import uuid

idempotency_key = str(uuid.uuid4())  # "ik-9a8b7c6d"

response = http.post(
    "/api/orders",
    headers={"Idempotency-Key": idempotency_key},
    json={
        "product_id": "prod-42",
        "quantity": 2,
        "shipping_address": "123 Main St"
    }
)

# If network timeout occurs, client retries with SAME key
response = http.post(
    "/api/orders",
    headers={"Idempotency-Key": idempotency_key},  # same key
    json={
        "product_id": "prod-42",
        "quantity": 2,
        "shipping_address": "123 Main St"
    }
)
# Server returns the same order from the first request
```

```python
# Server side
def create_order(request):
    idempotency_key = request.headers.get("Idempotency-Key")

    # Check for existing result
    cached = db.query(
        "SELECT response FROM idempotency_keys WHERE key = ?",
        idempotency_key
    )
    if cached:
        return cached.response  # Return same response

    with db.transaction():
        # Lock the idempotency key
        db.execute(
            "INSERT INTO idempotency_keys (key, status) VALUES (?, 'processing')",
            idempotency_key
        )

        # Create the order
        order = Order.create(
            product_id=request.json["product_id"],
            quantity=request.json["quantity"],
            shipping_address=request.json["shipping_address"]
        )

        # Store the response
        response = {"order_id": order.id, "status": "created"}
        db.execute(
            "UPDATE idempotency_keys SET status = 'completed', response = ? WHERE key = ?",
            json.dumps(response), idempotency_key
        )

    return response
```

---

## 6. Benefits and Drawbacks

### Benefits

| Benefit | Description |
|---------|-------------|
| **Safe duplicate handling** | Consumers process messages exactly once from a business perspective, regardless of delivery count |
| **Enables at-least-once delivery** | Systems can rely on simpler and more robust at-least-once semantics instead of complex exactly-once |
| **Simpler retry logic** | Producers and brokers can freely retry without fear of causing incorrect side effects |
| **Resilience to failures** | Consumer crashes, network timeouts, and broker restarts do not cause data corruption |
| **Operational flexibility** | Operations teams can safely replay messages for recovery or reprocessing |
| **Compatibility with event sourcing** | Works naturally with [event-driven architectures](../event-driven/event-driven-architecture.md) where events may be replayed |

### Drawbacks

| Drawback | Description | Mitigation |
|----------|-------------|------------|
| **Additional storage** | Tracking processed message IDs requires database space | TTL-based cleanup, Redis with auto-expiry |
| **Performance overhead** | Every message requires a lookup before processing | Use fast stores (Redis), indexing, caching |
| **Implementation complexity** | Adds code complexity to every consumer | Abstract into a middleware or framework |
| **Clock synchronization** | TTL-based cleanup depends on consistent timestamps | Use logical clocks or broker-provided timestamps |
| **Transaction scope** | Deduplication check and business logic must be atomic | Use database transactions or two-phase patterns |
| **Key generation burden** | Idempotency keys require client-side coordination | Provide SDK/library for key generation |

### Decision Matrix: Which Strategy to Use

| Strategy | Best For | Complexity | Performance | Storage Cost |
|----------|----------|------------|-------------|--------------|
| **Message deduplication** | General purpose, any consumer | Medium | Medium | Medium |
| **Natural idempotency** | Simple state updates (SET vs INCREMENT) | Low | High | None |
| **Idempotency keys** | API endpoints, client-initiated requests | Medium | Medium | Medium |
| **Optimistic locking** | Entity updates with versioning | Low | High | Low |
| **Database constraints** | Insert-heavy operations | Low | High | Low |

---

## 7. Related Patterns

- **[Messaging](./messaging.md)** -- The core context for this pattern. Idempotent consumers are essential in any messaging-based architecture where at-least-once delivery is used.

- **[Transactional Outbox](../messaging/transactional-outbox.md)** -- A related reliable messaging pattern that ensures messages are published atomically with database changes. Even with the Transactional Outbox ensuring reliable publishing, consumers still need idempotency because the outbox relay process may deliver duplicates.

- **[Event-Driven Architecture](../event-driven/event-driven-architecture.md)** -- Idempotent consumers are critical in event-driven systems where events can be replayed, redelivered, or processed out of order.

- **[Saga Pattern](../distributed-transactions/saga.md)** -- In saga-based workflows, compensating transactions and step handlers must be idempotent because the orchestrator or choreography may retry failed steps.

- **[CQRS](../data-patterns/cqrs.md)** -- In CQRS architectures, the event handlers that update read models must be idempotent since events can be replayed to rebuild projections.

- **[Circuit Breaker](../resilience/circuit-breaker.md)** -- When a circuit breaker opens and later closes, retried messages may arrive as duplicates. Idempotent consumers ensure these retries are safe.

---

## 8. Real-World Usage

### Stripe: Idempotency Keys

Stripe is one of the most well-known implementations of the idempotency key pattern. Every mutating API request accepts an `Idempotency-Key` header.

```
POST /v1/charges
Idempotency-Key: ik_KG5LxwFBepaKHyUD
Content-Type: application/json

{
  "amount": 2000,
  "currency": "usd",
  "source": "tok_visa"
}
```

**Stripe's implementation details:**
- Keys are stored for 24 hours and then automatically expire
- If a request is in progress, a concurrent duplicate returns a `409 Conflict`
- The idempotent response includes the exact same status code and body
- Keys are scoped per API key (different accounts can use the same key)

### AWS SQS: Visibility Timeout and Deduplication

Amazon SQS provides built-in support for idempotent processing:

- **Standard queues**: At-least-once delivery; consumers must implement their own deduplication
- **FIFO queues**: Built-in deduplication using `MessageDeduplicationId` (5-minute window)

```
SQS FIFO Deduplication:

  Producer                    SQS FIFO Queue             Consumer
     │                             │                        │
     │  Send(DeduplicationId=X)    │                        │
     ├────────────────────────────▶│                        │
     │                             │  Store message         │
     │                             │                        │
     │  Send(DeduplicationId=X)    │                        │
     ├────────────────────────────▶│                        │
     │                             │  Reject (duplicate     │
     │                             │  within 5-min window)  │
     │                             │                        │
     │                             │  Deliver message       │
     │                             ├───────────────────────▶│
     │                             │                        │
```

### Kafka Consumer Groups: Offset Management

Apache Kafka consumers track their position (offset) in each partition. Idempotent processing is needed because offsets may not always be committed before a crash.

```
Kafka Offset and Idempotency:

  Partition: [msg-0] [msg-1] [msg-2] [msg-3] [msg-4]
                                 ^
                         committed offset = 2

  Consumer processes msg-2, msg-3
  Consumer crashes before committing offset to 4

  On restart:
  Consumer re-reads from offset 2 (msg-2 is redelivered)
  Idempotent handler detects msg-2 is already processed → skip
  Processes msg-3 (again, idempotent) and msg-4
  Commits offset to 5
```

**Kafka's Idempotent Producer:**

Kafka also supports idempotent producers (via `enable.idempotence=true`), which prevent the producer from sending duplicate messages to the broker. However, this only covers producer-to-broker duplicates; consumer-side idempotency is still necessary.

### PayPal: Request ID Pattern

PayPal uses a `PayPal-Request-Id` header for idempotent requests:

```
POST /v2/checkout/orders
PayPal-Request-Id: req-unique-7a8b9c
Content-Type: application/json

{
  "intent": "CAPTURE",
  "purchase_units": [{
    "amount": {"currency_code": "USD", "value": "100.00"}
  }]
}
```

If the same `PayPal-Request-Id` is sent again, PayPal returns the original response without creating a new order.

---

## 9. Summary

The Idempotent Consumer pattern is essential for building reliable message-driven systems in a [microservices architecture](../architecture/microservices.md). Since message brokers guarantee at-least-once delivery, consumers must be designed to handle duplicates safely.

**Key Takeaways:**

| Aspect | Detail |
|--------|--------|
| **Problem** | Duplicate messages cause incorrect side effects (double charges, duplicate records) |
| **Solution** | Design handlers that produce the same result regardless of delivery count |
| **Strategies** | Message deduplication, natural idempotency, idempotency keys, optimistic locking, DB constraints |
| **Storage** | Track processed message IDs with TTL-based cleanup |
| **Real-world** | Stripe (idempotency keys), AWS SQS (FIFO deduplication), Kafka (offset + idempotent producers) |

**Remember:** Idempotency is not optional in distributed systems. Any consumer that processes messages from a broker should assume duplicates will arrive and handle them correctly. The [Transactional Outbox](../messaging/transactional-outbox.md) pattern ensures reliable publishing, but idempotent consumers ensure reliable processing.

---

**References:**
- Chris Richardson, "Microservices Patterns" (2018) - Chapter on messaging and idempotency
- Stripe API Documentation: Idempotent Requests
- AWS Documentation: Amazon SQS Message Deduplication
- Apache Kafka Documentation: Idempotent Producer and Consumer Design
- Martin Kleppmann, "Designing Data-Intensive Applications" (2017) - Chapter on stream processing
