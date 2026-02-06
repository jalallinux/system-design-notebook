# Command-side Replica

## 1. Introduction

**Command-side Replica** is a data management pattern used in distributed systems and [microservices architectures](../architecture/microservices.md). It involves maintaining a **local, read-only copy** of data owned by another service, specifically to support **command processing** -- that is, validation, authorization, and business logic execution on the write side.

### Origin and Motivation

In a [microservices architecture](../architecture/microservices.md) that follows the Database per Service principle, each service owns its data exclusively. However, processing a command often requires data that lives in another service. The Command-side Replica pattern solves this by keeping a **local projection** of the remote data, updated asynchronously through domain events.

### How It Differs from a CQRS Read Model

A common source of confusion is the relationship between a command-side replica and a [CQRS](./cqrs.md) read model. While both are derived projections of data, they serve fundamentally different purposes:

| Aspect | CQRS Read Model | Command-side Replica |
|--------|-----------------|----------------------|
| **Purpose** | Serve queries (read side) | Support command validation (write side) |
| **Consumer** | Query handlers, API responses | Command handlers, domain logic |
| **Data source** | Same service's write model | Another service's domain events |
| **Consistency need** | Eventual (for display) | Eventual (for validation) |
| **Location** | Read side of the same service | Write side of a *different* service |

**Key insight**: A command-side replica lives on the **write path**, not the read path. It enables a service to validate and process commands without making synchronous calls to external services.

---

## 2. Context and Problem

### Context

Consider a system built with [microservices](../architecture/microservices.md), where each service has its own database (Database per Service pattern). Services communicate asynchronously through events or synchronously through APIs.

```
┌──────────────────┐          ┌──────────────────┐
│  Order Service   │          │ Product Service   │
│                  │          │                   │
│  Orders DB       │          │  Products DB      │
│  ┌────────────┐  │          │  ┌─────────────┐  │
│  │ orders     │  │          │  │ products    │  │
│  │ order_items│  │          │  │ inventory   │  │
│  └────────────┘  │          │  │ pricing     │  │
│                  │          │  └─────────────┘  │
└──────────────────┘          └──────────────────┘
```

### The Problem

When the Order Service receives a `PlaceOrder` command, it needs to:

1. **Validate** that the requested products exist
2. **Check** that the products are in stock
3. **Verify** the current prices to calculate the order total
4. **Apply** business rules (e.g., maximum order quantity, customer eligibility)

All of this data lives in the **Product Service**. How should the Order Service access it?

### The Naive Approach: Synchronous API Calls

```
┌──────────┐     PlaceOrder      ┌──────────────┐
│  Client   │ ──────────────────▶│ Order Service │
└──────────┘                     └──────┬───────┘
                                        │
                          GET /products/123
                          GET /products/456
                          GET /inventory/123
                          GET /inventory/456
                                        │
                                        ▼
                                 ┌──────────────┐
                                 │Product Service│
                                 └──────────────┘
```

**Problems with this approach:**

- **Latency**: Multiple synchronous HTTP calls add up, especially under load
- **Coupling**: Order Service depends on Product Service being available at command time
- **Fragility**: If Product Service is down, Order Service cannot process any commands
- **Throughput**: Network round-trips become a bottleneck for high-volume command processing
- **Cascading failures**: Slow responses from Product Service can cause timeouts and failures to cascade (see [Circuit Breaker](../resilience/circuit-breaker.md))

---

## 3. Forces

The following forces push toward adopting the Command-side Replica pattern:

| Force | Description |
|-------|-------------|
| **Low-latency command processing** | Commands must be validated quickly; network calls to other services add unacceptable latency |
| **Autonomy and resilience** | A service should be able to process commands even when other services are temporarily unavailable |
| **Reduced coupling** | Synchronous inter-service calls create runtime dependencies that reduce system resilience |
| **Acceptable eventual consistency** | The replicated data does not need to be perfectly up-to-date; slight staleness is tolerable for validation purposes |
| **High command throughput** | The service handles a large volume of commands and cannot afford network round-trips for each one |
| **Data ownership boundaries** | The data is owned by another service and should not be directly accessed from its database |

### Tensions

- **Freshness vs. performance**: Local data is faster but may be stale
- **Simplicity vs. resilience**: Synchronous calls are simpler but create runtime dependencies
- **Duplication vs. autonomy**: Replicating data means maintaining it in two places, but enables independent operation

---

## 4. Solution

### Core Idea

Subscribe to **domain events** published by the service that owns the data. Maintain a **local, read-only replica** of the relevant subset of that data. Use this replica during command processing for validation and business logic.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Product Service (Data Owner)                 │
│                                                                  │
│  ┌─────────────┐     Publishes Events     ┌──────────────────┐  │
│  │ Products DB │ ────────────────────────▶ │  Event Bus /     │  │
│  └─────────────┘                           │  Message Broker  │  │
│                                            └────────┬─────────┘  │
└─────────────────────────────────────────────────────┼────────────┘
                                                      │
                    ProductCreated                     │
                    ProductPriceChanged                │
                    InventoryUpdated                   │
                    ProductDiscontinued                │
                                                      │
┌─────────────────────────────────────────────────────┼────────────┐
│                     Order Service (Consumer)         │            │
│                                                      ▼            │
│  ┌──────────────────┐    ┌──────────────────────────────────┐    │
│  │  Command Handler │    │  Event Handler                    │    │
│  │  (PlaceOrder)    │    │  (Subscribes to Product events)  │    │
│  │                  │    │                                    │    │
│  │  Uses replica ──▶│    │  Updates replica                  │    │
│  │  for validation  │    │  ┌────────────────────────────┐   │    │
│  └──────────────────┘    │  │ product_replica table      │   │    │
│                          │  │ ─────────────────────────  │   │    │
│                          │  │ product_id  (PK)           │   │    │
│                          │  │ name                       │   │    │
│                          │  │ price                      │   │    │
│                          │  │ stock_quantity              │   │    │
│                          │  │ is_available                │   │    │
│                          │  │ last_updated_at             │   │    │
│                          │  └────────────────────────────┘   │    │
│                          └──────────────────────────────────┘    │
│                                                                  │
│  ┌──────────────┐                                                │
│  │  Orders DB   │   (Orders + Product Replica in same DB)        │
│  └──────────────┘                                                │
└──────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Flow

```
Step 1: Product Service publishes domain events
─────────────────────────────────────────────────

Product Service ──▶ ProductCreated {
                        productId: "P-001",
                        name: "Wireless Keyboard",
                        price: 49.99,
                        stockQuantity: 500
                    }

Step 2: Order Service subscribes and updates local replica
──────────────────────────────────────────────────────────

Event Handler receives ProductCreated
  ──▶ INSERT INTO product_replica
       (product_id, name, price, stock_quantity, is_available, last_updated_at)
       VALUES ('P-001', 'Wireless Keyboard', 49.99, 500, true, NOW())

Step 3: Client sends PlaceOrder command
───────────────────────────────────────

Client ──▶ PlaceOrder {
               customerId: "C-100",
               items: [
                   { productId: "P-001", quantity: 2 },
                   { productId: "P-042", quantity: 1 }
               ]
           }

Step 4: Command handler validates using local replica
─────────────────────────────────────────────────────

PlaceOrderHandler:
  1. Load products from product_replica        ← LOCAL query, no network call
     SELECT * FROM product_replica
     WHERE product_id IN ('P-001', 'P-042')

  2. Validate all products exist               ← LOCAL check
  3. Validate all products are available        ← LOCAL check
  4. Validate sufficient stock                  ← LOCAL check
  5. Calculate total using replica prices       ← LOCAL calculation
  6. Create order                               ← Write to orders table
  7. Publish OrderPlaced event                  ← For downstream consumers

Step 5: Order created successfully
──────────────────────────────────

No synchronous calls to Product Service were needed.
```

### What Data to Replicate

You should only replicate the **minimal subset** of data needed for command validation:

```
Product Service owns:                Order Service replicates:
───────────────────                  ─────────────────────────
product_id              ──────────▶  product_id
name                    ──────────▶  name
description             ✗ (not needed for validation)
price                   ──────────▶  price
stock_quantity          ──────────▶  stock_quantity
is_available            ──────────▶  is_available
images[]                ✗ (not needed for validation)
detailed_specs          ✗ (not needed for validation)
reviews[]               ✗ (not needed for validation)
created_at              ✗ (not needed for validation)
last_updated_at         ──────────▶  last_updated_at
```

**Principle**: Replicate the minimum data needed. Less data means fewer events to process, less storage, and a smaller consistency window.

### Event Handling: Keeping the Replica Current

```
┌─────────────────────────────────────────────────────┐
│               Event Handlers in Order Service        │
├─────────────────────────────────────────────────────┤
│                                                      │
│  on ProductCreated(event):                           │
│      INSERT INTO product_replica                     │
│      VALUES (event.productId, event.name,            │
│              event.price, event.stock, true, NOW())  │
│                                                      │
│  on ProductPriceChanged(event):                      │
│      UPDATE product_replica                          │
│      SET price = event.newPrice,                     │
│          last_updated_at = NOW()                     │
│      WHERE product_id = event.productId              │
│                                                      │
│  on InventoryUpdated(event):                         │
│      UPDATE product_replica                          │
│      SET stock_quantity = event.newQuantity,          │
│          last_updated_at = NOW()                     │
│      WHERE product_id = event.productId              │
│                                                      │
│  on ProductDiscontinued(event):                      │
│      UPDATE product_replica                          │
│      SET is_available = false,                       │
│          last_updated_at = NOW()                     │
│      WHERE product_id = event.productId              │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Idempotent Event Handling

Since events may be delivered more than once (at-least-once delivery), handlers must be **idempotent**:

```
on ProductPriceChanged(event):
    // Check if we have already processed this event
    if (event.eventId already in processed_events table):
        return   // Skip duplicate

    UPDATE product_replica
    SET price = event.newPrice,
        last_updated_at = NOW()
    WHERE product_id = event.productId

    INSERT INTO processed_events (event_id, processed_at)
    VALUES (event.eventId, NOW())
```

For more on event delivery guarantees, see [Event-Driven Architecture](../event-driven/event-driven-architecture.md).

---

## 5. Example

### Scenario: E-commerce Order Processing

An e-commerce platform has the following services:

```
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│ Product Service│  │Customer Service│  │  Order Service  │
│                │  │                │  │                 │
│ - Products     │  │ - Customers    │  │ - Orders        │
│ - Inventory    │  │ - Memberships  │  │ - Order Items   │
│ - Pricing      │  │ - Credit Limits│  │                 │
│                │  │                │  │ Replicas:       │
│                │  │                │  │ - Products (R/O)│
│                │  │                │  │ - Customers(R/O)│
└───────┬────────┘  └───────┬────────┘  └────────────────┘
        │                   │
        │   Domain Events   │
        └───────────────────┘
                │
                ▼
        ┌───────────────┐
        │  Message Broker│
        │  (e.g., Kafka) │
        └───────┬───────┘
                │
                ▼
        Order Service subscribes
        and maintains replicas
```

### Command: PlaceOrder

```
PlaceOrderCommand {
    customerId: "C-100"
    items: [
        { productId: "P-001", quantity: 2 },
        { productId: "P-042", quantity: 1 }
    ]
    shippingAddress: "123 Main St"
}
```

### Validation Using Command-side Replicas

```
PlaceOrderHandler.handle(command):

    // Step 1: Validate customer using CUSTOMER replica
    customer = customerReplica.findById(command.customerId)
    if (customer == null)
        throw CustomerNotFoundException

    if (customer.status != ACTIVE)
        throw CustomerNotActiveException

    // Step 2: Validate products using PRODUCT replica
    products = productReplica.findByIds(command.items.map(i => i.productId))

    if (products.size != command.items.size)
        throw ProductNotFoundException

    for each item in command.items:
        product = products.get(item.productId)

        if (!product.isAvailable)
            throw ProductNotAvailableException

        if (product.stockQuantity < item.quantity)
            throw InsufficientStockException

    // Step 3: Calculate total using replica prices
    total = 0
    for each item in command.items:
        product = products.get(item.productId)
        total += product.price * item.quantity

    // Step 4: Check customer credit limit
    if (customer.membershipType == CREDIT)
        if (total > customer.creditLimit)
            throw CreditLimitExceededException

    // Step 5: Create order (write to Order Service's own DB)
    order = Order.create(command.customerId, command.items, total)
    orderRepository.save(order)

    // Step 6: Publish domain event
    eventBus.publish(OrderPlaced {
        orderId: order.id,
        customerId: command.customerId,
        items: command.items,
        total: total
    })

    return order.id
```

### Complete Data Flow Diagram

```
     Product Service              Message Broker              Order Service
     ────────────────             ──────────────              ─────────────

     Admin updates price
     for product P-001
            │
            ▼
     ProductPriceChanged ────────▶  Topic:          ────────▶  Event Handler
     {                              product.events              │
       productId: "P-001",                                      ▼
       oldPrice: 49.99,                                  UPDATE product_replica
       newPrice: 44.99                                   SET price = 44.99
     }                                                   WHERE product_id = 'P-001'
                                                                │
                                                                ▼
                                                         Replica is now current
                                                                │
     ───────── Time passes ─────────                            │
                                                                │
     Client sends PlaceOrder ───────────────────────────▶ PlaceOrderHandler
     for product P-001                                          │
                                                                ▼
                                                         SELECT * FROM
                                                         product_replica
                                                         WHERE product_id = 'P-001'
                                                                │
                                                                ▼
                                                         price = 44.99 (current!)
                                                         Validates and creates order
```

---

## 6. Benefits and Drawbacks

### Benefits

| Benefit | Description |
|---------|-------------|
| **Low-latency command processing** | All validation data is local -- no network round-trips during command handling |
| **Service autonomy** | The service can process commands even when the source service is temporarily down |
| **Reduced runtime coupling** | No synchronous dependencies on other services at command time |
| **Resilience** | Temporary unavailability of the data-owning service does not block command processing |
| **Predictable performance** | Local database queries have consistent, predictable latency |
| **Scalability** | Command throughput is not limited by external service capacity |
| **Natural fit with event-driven systems** | Leverages existing [event-driven architecture](../event-driven/event-driven-architecture.md) infrastructure |

### Drawbacks

| Drawback | Description | Mitigation |
|----------|-------------|------------|
| **Eventual consistency** | Replica data may be stale; a product price might change between replica update and command processing | Accept as trade-off; use compensating transactions ([Saga Pattern](../distributed-transactions/saga.md)) for critical cases |
| **Data duplication** | Same data stored in multiple services, increasing storage and maintenance cost | Replicate only the minimum necessary fields |
| **Event handling complexity** | Must handle event ordering, idempotency, and failures | Use idempotent handlers, event deduplication, and dead-letter queues |
| **Schema evolution** | Changes to the source service's event schema require updates in all consuming services | Use event versioning and backward-compatible schemas |
| **Initial data loading** | When a service starts fresh, it needs to populate the replica from scratch | Use snapshot events or batch APIs for initial bootstrap |
| **Stale data risk** | If events are delayed or lost, the replica may have outdated data | Monitor replica freshness; add staleness checks in critical validations |
| **Debugging complexity** | Tracing data flow across services and events is harder than a simple API call | Use correlation IDs, distributed tracing, and event logs |

### When Stale Data Matters

Not all staleness is equal. Consider these scenarios:

```
Scenario 1: Price changed 2 seconds ago, replica not yet updated
─────────────────────────────────────────────────────────────────
Risk Level: LOW
Impact: Customer charged old price ($49.99 instead of $44.99)
Mitigation: Saga compensates or business absorbs small difference

Scenario 2: Product discontinued, replica not yet updated
─────────────────────────────────────────────────────────────────
Risk Level: MEDIUM
Impact: Order placed for unavailable product
Mitigation: Downstream inventory check fails, Saga triggers compensation

Scenario 3: Stock depleted, replica shows stock available
─────────────────────────────────────────────────────────────────
Risk Level: MEDIUM
Impact: Overselling
Mitigation: Final stock reservation at inventory service;
            Saga handles out-of-stock compensation
```

**Key principle**: The command-side replica provides a **best-effort validation**. For critical constraints (like actual stock reservation), a downstream step in a [Saga](../distributed-transactions/saga.md) should perform the authoritative check against the source of truth.

---

## 7. Related Patterns

### Directly Related

| Pattern | Relationship |
|---------|-------------|
| [Database per Service](./database-per-service.md) | The constraint that motivates command-side replicas -- each service owns its data |
| [CQRS](./cqrs.md) | Command-side replica is on the write side; CQRS read models are on the read side. Both are projections but for different purposes |
| [Domain Event](../event-driven/domain-event.md) | The mechanism used to keep the replica synchronized with the source of truth |
| [Saga Pattern](../distributed-transactions/saga.md) | Used to handle cases where the replica's eventual consistency leads to invalid commands that need compensation |

### Complementary Patterns

| Pattern | How It Complements |
|---------|-------------------|
| [Event-Driven Architecture](../event-driven/event-driven-architecture.md) | Provides the infrastructure (event bus, topics, subscriptions) for event propagation |
| [Circuit Breaker](../resilience/circuit-breaker.md) | Protects against cascading failures if a fallback to synchronous calls is needed |
| [API Gateway](../architecture/api-gateway.md) | Routes commands to the appropriate service where command-side replicas are used |
| [CAP Theorem](../fundamentals/cap-theorem.md) | Explains why eventual consistency is an acceptable trade-off in distributed systems |

### Pattern Comparison

```
┌───────────────────────────────────────────────────────────────┐
│          How services get data they don't own                  │
├────────────────────┬──────────────────────────────────────────┤
│                    │                                           │
│  1. Sync API Call  │  Service A ──HTTP──▶ Service B           │
│     (Request/      │  + Simple                                │
│      Response)     │  - Coupled, fragile, slow                │
│                    │                                           │
├────────────────────┼──────────────────────────────────────────┤
│                    │                                           │
│  2. Command-side   │  Service B ──Events──▶ Service A         │
│     Replica        │                        (local replica)   │
│     (This pattern) │  + Fast, decoupled, resilient            │
│                    │  - Eventually consistent, data duplication│
│                    │                                           │
├────────────────────┼──────────────────────────────────────────┤
│                    │                                           │
│  3. API Composition│  Gateway ──▶ Service A + Service B       │
│     (at gateway)   │              (aggregate responses)       │
│                    │  + No duplication                         │
│                    │  - Gateway complexity, still coupled      │
│                    │                                           │
├────────────────────┼──────────────────────────────────────────┤
│                    │                                           │
│  4. Shared Database│  Service A and B share same DB           │
│     (anti-pattern) │  + Simple                                │
│                    │  - Defeats microservices purpose          │
│                    │                                           │
└────────────────────┴──────────────────────────────────────────┘
```

---

## 8. Real-World Usage

### When to Use Command-side Replica

| Scenario | Example |
|----------|---------|
| **Command validation needing external data** | Order Service validating product prices and stock from Product Service |
| **High-throughput command processing** | Payment Service checking account balances from Account Service |
| **Resilience-critical commands** | Booking Service validating room availability from Hotel Service, even if Hotel Service is briefly down |
| **Cross-service business rules** | Lending Service checking customer credit scores from Credit Service for loan approval |
| **Multi-service aggregation for commands** | Shipping Service needing customer address (from Customer Service) and package dimensions (from Product Service) to calculate rates |

### When to Avoid

| Scenario | Why | Alternative |
|----------|-----|-------------|
| **Data changes very frequently** | Replica constantly stale; events overwhelm the system | Use synchronous API with caching and [Circuit Breaker](../resilience/circuit-breaker.md) |
| **Perfect consistency required** | Cannot tolerate any staleness in validation | Use [Two-Phase Commit](../distributed-transactions/two-phase-commit.md) or synchronous call |
| **Very large datasets** | Replicating millions of records is impractical | Use selective replication (hot data only) or API calls |
| **Simple, low-traffic systems** | Overhead of events and replicas not justified | Direct synchronous API calls suffice |

### Industry Examples

**1. Amazon -- Order Processing**

Amazon's order service maintains local replicas of product catalog data (pricing, availability) to validate orders without calling the catalog service synchronously. Final price verification happens downstream as part of a [Saga](../distributed-transactions/saga.md) before charging the customer.

**2. Uber -- Trip Pricing**

The trip service maintains a local replica of surge pricing data and driver availability data. When a rider requests a trip, the local replica provides an instant price estimate. The actual fare is confirmed later using the authoritative pricing service.

**3. Netflix -- Content Authorization**

When a user presses play, the playback service validates content licensing and user subscription status using local replicas rather than calling the subscription service and licensing service synchronously. This enables sub-100ms authorization decisions.

**4. Banking -- Transfer Validation**

A transfer service maintains a replica of account statuses and basic balance information. When a transfer command arrives, it validates the account is active and has an approximate sufficient balance. The actual balance deduction is handled by the account service in a Saga step.

### Implementation Technologies

| Component | Technology Options |
|-----------|--------------------|
| **Event Bus** | Apache Kafka, RabbitMQ, AWS SNS/SQS, Azure Service Bus |
| **Replica Storage** | Same database as service (separate table), Redis, in-memory cache |
| **Event Format** | CloudEvents, Avro, Protobuf, JSON |
| **Initial Load** | Snapshot events, batch REST API, database dump |
| **Monitoring** | Replica lag metrics, freshness checks, event processing dashboards |

For a comparison of event bus technologies, see [RabbitMQ vs Kafka](../messaging/rabbitmq-vs-kafka.md).

---

## 9. Summary

### Key Takeaways

1. **A command-side replica is a local, read-only copy** of data owned by another service, maintained via domain events, and used on the **write side** for command validation and business logic.

2. **It solves the problem** of needing external data during command processing without making synchronous inter-service calls that increase latency, coupling, and fragility.

3. **It is not a CQRS read model**. While both are projections, a command-side replica exists on the write path, not the read path. A [CQRS](./cqrs.md) read model serves queries; a command-side replica serves command validation.

4. **Eventual consistency is the core trade-off**. The replica may be slightly stale, which is acceptable for most validation scenarios. For critical checks, use a downstream [Saga](../distributed-transactions/saga.md) step against the authoritative source.

5. **Replicate only what you need**. Keep the replica minimal -- only the fields required for command validation. This reduces event volume, storage, and the consistency window.

6. **Idempotent event handlers are essential**. Events may be delivered more than once; handlers must gracefully handle duplicates.

7. **Monitor replica freshness**. Track the lag between the source of truth and the local replica. Alert on excessive staleness.

### Decision Checklist

```
Should you use a Command-side Replica?

 [ ] Your service needs data from another service to validate commands
 [ ] The data can be eventually consistent for validation purposes
 [ ] You need low-latency command processing
 [ ] You want your service to be resilient to other services being down
 [ ] You have an event-driven infrastructure in place (or plan to build one)
 [ ] The dataset to replicate is manageable in size
 [ ] Your team is comfortable with eventual consistency and event handling

If most boxes are checked ──▶ Command-side Replica is a good fit.
If few boxes are checked  ──▶ Consider synchronous API calls or other patterns.
```

### Quick Reference

```
┌─────────────────────────────────────────────────────────────────┐
│                    Command-side Replica Pattern                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  WHAT:  Local read-only copy of another service's data           │
│  WHY:   Low-latency command validation without sync calls        │
│  HOW:   Subscribe to domain events, maintain local projection    │
│  WHERE: On the write side of the consuming service               │
│                                                                  │
│  ┌──────────┐   Events    ┌──────────┐   Local     ┌─────────┐ │
│  │ Source   │ ──────────▶ │ Replica  │ ──────────▶ │ Command │ │
│  │ Service  │             │ (R/O)    │   Query     │ Handler │ │
│  └──────────┘             └──────────┘             └─────────┘ │
│                                                                  │
│  Trade-off: Eventual consistency for performance & resilience    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Related Reading

- [CQRS](./cqrs.md) -- Separating read and write models within a single service
- [Saga Pattern](../distributed-transactions/saga.md) -- Coordinating distributed transactions with compensation
- [Event-Driven Architecture](../event-driven/event-driven-architecture.md) -- Event patterns and delivery guarantees
- [CAP Theorem](../fundamentals/cap-theorem.md) -- Understanding consistency trade-offs in distributed systems
- [Microservices Architecture](../architecture/microservices.md) -- The architectural style that motivates this pattern
- [RabbitMQ vs Kafka](../messaging/rabbitmq-vs-kafka.md) -- Choosing a messaging technology for event propagation

---

**Remember**: The Command-side Replica pattern trades data freshness for performance and resilience. It is a pragmatic choice in systems where synchronous inter-service calls at command time are too costly, and where eventual consistency is an acceptable trade-off for command validation.
