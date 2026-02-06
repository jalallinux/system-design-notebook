# Domain Event

## 1. Introduction

A **Domain Event** is a record of something significant that happened in the domain. It captures the fact that a meaningful state change occurred within a bounded context. Examples include `OrderPlaced`, `CustomerRegistered`, `PaymentReceived`, and `InventoryReserved`.

The concept originates from **Domain-Driven Design (DDD)**, introduced by Eric Evans in his seminal book and later expanded by Vaughn Vernon in "Implementing Domain-Driven Design." Domain Events bridge the gap between the domain model and the rest of the system, providing a structured way to communicate state changes across service boundaries.

### Core Idea

```
Something important happened in our domain.
We capture it as an immutable fact.
Other parts of the system can react to it.
```

### Domain Event Characteristics

| Characteristic | Description |
|----------------|-------------|
| **Immutable** | Once created, a domain event cannot be changed |
| **Past Tense** | Named in past tense (OrderPlaced, not PlaceOrder) |
| **Significant** | Represents a meaningful business occurrence |
| **Self-Contained** | Carries enough data for consumers to act |
| **Timestamped** | Records when the event occurred |

### Domain Event vs Command

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Command (imperative):   PlaceOrder, RegisterCustomer            │
│  ──────────────────────────────────────────────────────────────  │
│  Intent to do something. May be rejected.                        │
│                                                                  │
│  Domain Event (past tense):   OrderPlaced, CustomerRegistered    │
│  ──────────────────────────────────────────────────────────────  │
│  Fact that already happened. Cannot be undone.                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## 2. Context and Problem

In a [Microservices Architecture](../architecture/microservices.md), each service owns its own data and encapsulates its business logic within a bounded context. This autonomy is a strength, but it creates a fundamental challenge: **how do other services learn about changes that happened in a different service's domain?**

### The Challenge

```
┌───────────────┐          ┌───────────────┐          ┌───────────────┐
│  Order        │          │  Inventory    │          │  Notification  │
│  Service      │          │  Service      │          │  Service       │
│               │          │               │          │                │
│  Order is     │   ???    │  Needs to     │   ???    │  Needs to      │
│  placed       │─────────▶│  reserve      │─────────▶│  send email    │
│               │          │  stock        │          │  to customer   │
└───────────────┘          └───────────────┘          └───────────────┘

Problem: How does Inventory Service know an order was placed?
         How does Notification Service know to send an email?
```

**Direct coupling** (synchronous calls) makes services fragile and tightly bound. If the Notification Service is down, should the Order Service fail? Clearly not. The act of placing an order is a domain fact that other services should be able to react to independently.

## 3. Forces

Several forces shape the need for Domain Events:

- **Loose Coupling**: Services must remain independent. A change in one service should not require changes in others.
- **Reliable Notification**: When something important happens, interested parties must be notified reliably, even if they are temporarily unavailable.
- **Multiple Consumers**: The same event may be relevant to many consumers. Inventory needs to reserve stock, payment needs to charge the customer, and analytics needs to record the sale.
- **Temporal Decoupling**: The producer and consumers do not need to be available at the same time.
- **Audit Trail**: Business-critical changes must be traceable for compliance, debugging, and analytics.
- **Extensibility**: Adding new reactions to domain changes should not require modifying the source service.
- **Domain Clarity**: The domain model should express what is important in the business, including the events that matter.

## 4. Solution

The solution is to **publish Domain Events whenever significant state changes occur** in the domain model. Each event represents a fact about something that happened, and interested consumers subscribe to these events and react accordingly.

### Event Structure

A well-designed Domain Event contains all the information a consumer needs to understand what happened.

```
┌──────────────────────────────────────────────────┐
│                Domain Event                       │
├──────────────────────────────────────────────────┤
│  eventId:       "evt-a1b2c3d4"                   │
│  eventType:     "OrderPlaced"                    │
│  timestamp:     "2026-02-06T14:30:00Z"           │
│  aggregateId:   "order-98765"                    │
│  aggregateType: "Order"                          │
│  version:       1                                │
│  payload: {                                      │
│      orderId:    "order-98765",                  │
│      customerId: "cust-111",                     │
│      items: [                                    │
│          { productId: "P-1", qty: 2 },           │
│          { productId: "P-5", qty: 1 }            │
│      ],                                          │
│      totalAmount: 249.99,                        │
│      currency:    "USD"                          │
│  }                                               │
│  metadata: {                                     │
│      correlationId: "req-xyz789",                │
│      causationId:   "cmd-place-order-123",       │
│      userId:        "user-42"                    │
│  }                                               │
└──────────────────────────────────────────────────┘
```

### Event Field Reference

| Field | Purpose | Example |
|-------|---------|---------|
| `eventId` | Unique identifier for deduplication | `"evt-a1b2c3d4"` |
| `eventType` | What happened (past tense) | `"OrderPlaced"` |
| `timestamp` | When the event occurred | `"2026-02-06T14:30:00Z"` |
| `aggregateId` | Which entity the event belongs to | `"order-98765"` |
| `aggregateType` | Type of the aggregate root | `"Order"` |
| `version` | Schema version for evolution | `1` |
| `payload` | Business data about the event | `{ orderId, items, ... }` |
| `metadata` | Tracing and operational data | `{ correlationId, userId }` |

### Event Naming Conventions

Domain Events should always be named in the **past tense**, reflecting that they describe facts.

| Correct (Past Tense) | Incorrect (Imperative) |
|----------------------|----------------------|
| `OrderPlaced` | `PlaceOrder` |
| `PaymentReceived` | `ReceivePayment` |
| `CustomerRegistered` | `RegisterCustomer` |
| `InventoryReserved` | `ReserveInventory` |
| `ShipmentDispatched` | `DispatchShipment` |

### Event Granularity

Choosing the right level of detail in events is an important design decision.

```
Coarse-Grained Event:
┌───────────────────────────────────────────┐
│  OrderUpdated                             │
│  payload: { entire order object }         │
│                                           │
│  + Simple to publish                      │
│  + Fewer event types                      │
│  - Consumers must diff to find changes    │
│  - Larger payload                         │
└───────────────────────────────────────────┘

Fine-Grained Events:
┌───────────────────────────────────────────┐
│  OrderItemAdded                           │
│  OrderItemRemoved                         │
│  OrderShippingAddressChanged              │
│  OrderDiscountApplied                     │
│                                           │
│  + Consumers know exactly what changed    │
│  + Smaller payload per event              │
│  - More event types to manage             │
│  - More complex publishing logic          │
└───────────────────────────────────────────┘

Recommendation: Prefer fine-grained events that map
to meaningful business actions.
```

### Publishing Domain Events

Domain Events can be published from two locations in the application architecture:

**Option A: From Within Aggregates (DDD Purist Approach)**

```
┌──────────────────────────────────────────────────────────┐
│  Order Aggregate                                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  place(customerId, items):                               │
│      validate business rules                             │
│      update internal state                               │
│      register event: OrderPlaced { ... }                 │
│                                                          │
│  cancel(reason):                                         │
│      validate: can only cancel if not shipped            │
│      update status to CANCELLED                          │
│      register event: OrderCancelled { reason }           │
│                                                          │
│  uncommittedEvents: [ ... ]                              │
│      (collected during the operation, published          │
│       after the aggregate is persisted)                  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Option B: From Application Services**

```
┌──────────────────────────────────────────────────────────┐
│  OrderApplicationService                                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  placeOrder(command):                                    │
│      order = Order.place(command.customerId, ...)        │
│      orderRepository.save(order)                         │
│      eventPublisher.publish(OrderPlaced { ... })         │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Domain Event Flow

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Aggregate  │    │  Application │    │   Event      │
│   (Domain    │───▶│  Service     │───▶│   Publisher  │
│    Model)    │    │              │    │              │
└──────────────┘    └──────────────┘    └──────┬───────┘
                                               │
                                               │ publish
                                               ▼
                                        ┌──────────────┐
                                        │   Message    │
                                        │   Broker     │
                                        └──────┬───────┘
                                               │
                              ┌────────────────┼────────────────┐
                              │                │                │
                              ▼                ▼                ▼
                       ┌────────────┐   ┌────────────┐   ┌────────────┐
                       │ Inventory  │   │  Payment   │   │ Notification│
                       │ Service    │   │  Service   │   │  Service    │
                       └────────────┘   └────────────┘   └────────────┘
```

### Guaranteed Publishing with Transactional Outbox

A critical challenge is ensuring that the domain event is published **if and only if** the state change is committed to the database. If the application saves the order but crashes before publishing the event, the system becomes inconsistent.

The solution is the [Transactional Outbox](../messaging/transactional-outbox.md) pattern: write the event to an outbox table in the **same database transaction** as the state change, then a separate process reads the outbox and publishes events to the message broker.

```
┌────────────────────────────────────────────────────────────┐
│                   Single Database Transaction               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  1. INSERT INTO orders (id, status, ...) VALUES (...)      │
│  2. INSERT INTO outbox (event_id, event_type, payload)     │
│     VALUES ('evt-123', 'OrderPlaced', '{...}')             │
│                                                            │
│  COMMIT;   (both succeed or both fail)                     │
│                                                            │
└────────────────────────────────────────────────────────────┘
         │
         │  Outbox Relay (polls or uses CDC)
         ▼
┌──────────────┐         ┌──────────────┐
│   Outbox     │────────▶│   Message    │
│   Table      │ publish │   Broker     │
└──────────────┘         └──────────────┘
```

### Domain Events vs Integration Events

An important distinction in microservices is between Domain Events (internal) and Integration Events (external).

| Aspect | Domain Event | Integration Event |
|--------|-------------|-------------------|
| **Scope** | Within a bounded context | Across bounded contexts / services |
| **Audience** | Internal handlers in the same service | External microservices |
| **Coupling** | Tightly coupled to internal model | Abstracted, stable contract |
| **Schema** | Can change freely | Must be versioned carefully |
| **Transport** | In-process event bus or outbox | Message broker (Kafka, RabbitMQ) |
| **Example** | `OrderItemAdded` (internal detail) | `OrderPlaced` (public contract) |

```
┌─────────────────────────────────────────────────────────────┐
│  Order Service (Bounded Context)                             │
│                                                             │
│  Domain Events (internal):                                  │
│    OrderItemAdded                                           │
│    OrderItemRemoved                                         │
│    OrderTotalRecalculated                                   │
│    OrderValidated                                           │
│         │                                                   │
│         ▼                                                   │
│  Anti-Corruption Layer / Mapper                             │
│         │                                                   │
│         ▼                                                   │
│  Integration Event (public):                                │
│    OrderPlaced { orderId, customerId, total, items }        │
│                                                             │
└─────────────────────────────────┬───────────────────────────┘
                                  │
                                  ▼
                           Message Broker
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
              Inventory      Payment      Notification
              Service        Service        Service
```

## 5. Example

### E-Commerce: OrderPlaced Event

Consider an e-commerce platform where placing an order triggers multiple downstream reactions.

**Step 1: Customer places an order**

```
Customer ───▶ Order Service
                   │
                   │  1. Validate order
                   │  2. Save to database
                   │  3. Publish OrderPlaced event
                   │
                   ▼
            ┌──────────────┐
            │ OrderPlaced  │
            │ {            │
            │   orderId:   │
            │     "O-500", │
            │   customer:  │
            │     "C-42",  │
            │   items: [   │
            │     {P-1, 2},│
            │     {P-3, 1} │
            │   ],         │
            │   total:     │
            │     $185.00  │
            │ }            │
            └──────┬───────┘
                   │
                   ▼
            Message Broker
```

**Step 2: Multiple consumers react independently**

```
                        Message Broker
                             │
              ┌──────────────┼──────────────┬──────────────┐
              ▼              ▼              ▼              ▼
       ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
       │ Inventory  │ │  Payment   │ │   Email    │ │ Analytics  │
       │ Service    │ │  Service   │ │  Service   │ │  Service   │
       ├────────────┤ ├────────────┤ ├────────────┤ ├────────────┤
       │ Reserve    │ │ Charge     │ │ Send order │ │ Record     │
       │ stock for  │ │ customer's │ │ confirma-  │ │ sale in    │
       │ items P-1  │ │ payment    │ │ tion email │ │ dashboard  │
       │ and P-3    │ │ method     │ │ to C-42    │ │            │
       └────────────┘ └────────────┘ └────────────┘ └────────────┘
              │              │
              ▼              ▼
       InventoryReserved  PaymentProcessed
       (new events)       (new events)
```

**Step 3: Downstream events trigger further reactions**

```
PaymentProcessed ───▶ Order Service ───▶ updates order status to PAID
                 ───▶ Accounting Service ───▶ records revenue

InventoryReserved ───▶ Warehouse Service ───▶ prepares pick list
                  ───▶ Order Service ───▶ updates fulfillment status
```

This chain of events forms a [Saga](../distributed-transactions/saga.md) that coordinates the distributed transaction without tight coupling.

### Compensation on Failure

If the Payment Service fails to charge the customer:

```
PaymentFailed { orderId: "O-500", reason: "Insufficient funds" }
       │
       ├───▶ Inventory Service ───▶ releases reserved stock
       │                           (InventoryReleased event)
       │
       ├───▶ Order Service ───▶ marks order as CANCELLED
       │                        (OrderCancelled event)
       │
       └───▶ Email Service ───▶ sends cancellation notice to customer
```

## 6. Benefits and Drawbacks

### Benefits

| Benefit | Description |
|---------|-------------|
| **Loose Coupling** | Producer does not know or depend on consumers. New consumers can be added without changing the producer. |
| **Extensibility** | React to domain changes by adding new subscribers, not by modifying source code. |
| **Audit Trail** | Every significant state change is recorded as an event, enabling traceability and compliance. |
| **Enables EDA** | Domain Events are the foundation of [Event-Driven Architecture](./event-driven-architecture.md). |
| **Supports CQRS** | Events can feed read-model projections in a [CQRS](../data-patterns/cqrs.md) system. |
| **Supports Event Sourcing** | Domain Events can serve as the source of truth in an [Event Sourcing](./event-sourcing.md) system. |
| **Temporal Decoupling** | Producer and consumers do not need to be online at the same time. |
| **Domain Clarity** | Naming and modeling events makes the domain language explicit. |

### Drawbacks

| Drawback | Description |
|----------|-------------|
| **Eventual Consistency** | Consumers process events asynchronously, so the system is temporarily inconsistent. Related to the [CAP Theorem](../fundamentals/cap-theorem.md). |
| **Event Schema Evolution** | Changing event structure requires careful versioning and backward compatibility. |
| **Debugging Complexity** | Tracing a request across asynchronous event flows is harder than tracing synchronous calls. |
| **Ordering Challenges** | Guaranteeing global event order is expensive; usually only partition-level ordering is feasible. |
| **Duplicate Delivery** | At-least-once delivery means consumers must be idempotent. |
| **Infrastructure Overhead** | Requires a message broker, monitoring, dead-letter queues, and operational tooling. |
| **Over-Engineering Risk** | For simple applications, domain events add unnecessary complexity. |

## 7. Related Patterns

Domain Events connect to many patterns in the system design ecosystem:

**1. Event-Driven Architecture**

Domain Events are the building blocks of [Event-Driven Architecture](./event-driven-architecture.md). EDA provides the infrastructure and patterns for producing, routing, and consuming events at scale.

**2. Event Sourcing**

In [Event Sourcing](./event-sourcing.md), Domain Events become the source of truth. Instead of storing only the current state, every state change is persisted as an event and the current state is rebuilt by replaying events.

**3. CQRS**

[CQRS](../data-patterns/cqrs.md) separates read and write models. Domain Events are often used to synchronize the write model with one or more read-optimized projections.

**4. Saga Pattern**

The [Saga Pattern](../distributed-transactions/saga.md) coordinates distributed transactions through a sequence of Domain Events and compensating actions across multiple services.

**5. Transactional Outbox**

The [Transactional Outbox](../messaging/transactional-outbox.md) pattern guarantees that Domain Events are published reliably by writing them to an outbox table in the same database transaction as the state change.

**Pattern Relationships**:

```
                        Domain Event
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
    Event-Driven        Event Sourcing       CQRS
    Architecture              │                │
           │                  │                │
           │            ┌─────┴─────┐    ┌─────┴─────┐
           ▼            ▼           ▼    ▼           ▼
    Saga Pattern    Snapshots   Temporal  Read      Write
                                Queries  Models    Models
           │
           ▼
    Transactional
    Outbox
```

## 8. Real-World Usage

### Industry Adoption

Domain Events are used extensively in production microservices:

| Company | Usage |
|---------|-------|
| **Amazon** | Order lifecycle events drive warehouse, payment, shipping, and notification services independently. |
| **Netflix** | User activity events (viewed, paused, rated) feed recommendation, analytics, and billing systems. |
| **Uber** | Ride lifecycle events (requested, matched, started, completed) coordinate driver, rider, payment, and mapping services. |
| **Shopify** | Order, product, and inventory events power the merchant ecosystem and third-party integrations via webhooks. |
| **Stripe** | Payment lifecycle events (charge.succeeded, invoice.paid) are the backbone of their webhook system. |

### Common Technology Choices

| Technology | Role |
|------------|------|
| **Apache Kafka** | Distributed event streaming with persistence and replay. Ideal for high-throughput domain events. |
| **RabbitMQ** | General-purpose message broker with flexible routing. Good for task-oriented and notification events. |
| **AWS EventBridge** | Serverless event bus for event routing across AWS services and SaaS integrations. |
| **Azure Event Grid** | Event routing service in the Azure ecosystem. |
| **Axon Framework** | Java framework purpose-built for DDD, CQRS, and Event Sourcing with first-class Domain Event support. |

For a detailed comparison of messaging technologies, see [RabbitMQ vs Kafka](../messaging/rabbitmq-vs-kafka.md).

### Design Considerations in Practice

**1. Event Schemas and Registries**

In large systems, a schema registry (e.g., Confluent Schema Registry for Kafka) enforces event structure and compatibility rules. This prevents producers from publishing breaking changes.

**2. Correlation and Causation IDs**

Every Domain Event should carry a `correlationId` (linking it to the original user request) and a `causationId` (linking it to the command or event that caused it). This enables distributed tracing.

```
Request from User
  correlationId: "req-001"
       │
       ▼
  OrderPlaced
    correlationId: "req-001"
    causationId:   "cmd-place-order"
       │
       ├──▶ PaymentProcessed
       │      correlationId: "req-001"
       │      causationId:   "evt-order-placed"
       │
       └──▶ InventoryReserved
              correlationId: "req-001"
              causationId:   "evt-order-placed"
```

**3. Dead-Letter Queues**

When a consumer fails to process an event after multiple retries, the event is moved to a dead-letter queue for manual inspection and reprocessing.

**4. Idempotent Consumers**

Since at-least-once delivery is the norm, consumers must handle duplicate events gracefully by checking the `eventId` before processing.

## 9. Summary

| Aspect | Detail |
|--------|--------|
| **What** | An immutable record of a significant state change in the domain |
| **Origin** | Domain-Driven Design (Eric Evans, Vaughn Vernon) |
| **Naming** | Past tense: `OrderPlaced`, `PaymentReceived` |
| **Structure** | eventId, eventType, timestamp, aggregateId, payload, metadata |
| **Publishing** | From aggregates or application services, with guaranteed delivery via Transactional Outbox |
| **Consumers** | Other microservices, read-model projections, notification systems, analytics |
| **Key Benefit** | Loose coupling and extensibility across service boundaries |
| **Key Challenge** | Eventual consistency, schema evolution, and debugging distributed flows |
| **Related Patterns** | [Event-Driven Architecture](./event-driven-architecture.md), [Event Sourcing](./event-sourcing.md), [CQRS](../data-patterns/cqrs.md), [Saga](../distributed-transactions/saga.md), [Transactional Outbox](../messaging/transactional-outbox.md) |
| **When to Use** | Microservices needing to react to changes in other services; when audit trail, extensibility, and loose coupling are priorities |
| **When to Avoid** | Simple monolithic CRUD applications where synchronous calls suffice |

### Key Takeaways

1. **Domain Events capture business facts** as immutable records of what happened
2. **Past-tense naming** makes events self-documenting and aligns with ubiquitous language
3. **Guaranteed publishing** via the Transactional Outbox pattern prevents data inconsistency
4. **Domain Events vs Integration Events**: keep internal model details private; publish stable public contracts
5. **Idempotent consumers** are essential because at-least-once delivery is the practical default
6. **Fine-grained events** that map to business actions are preferred over coarse-grained catch-all events
7. **Cross-link with other patterns**: Domain Events are the foundation for Event Sourcing, CQRS, Saga, and Event-Driven Architecture

### Further Reading

**Books**:
- "Domain-Driven Design" by Eric Evans (Chapter 8: Domain Events)
- "Implementing Domain-Driven Design" by Vaughn Vernon
- "Building Event-Driven Microservices" by Adam Bellemare
- "Designing Data-Intensive Applications" by Martin Kleppmann

**Resources**:
- Martin Fowler: "Domain Event" pattern description
- Vaughn Vernon: "Modeling Domain Events" series
- Microsoft: Domain Events design and implementation guide
- Axon Framework documentation: Domain Event handling

---

**Navigation**:
- Previous: [Event-Driven Architecture](./event-driven-architecture.md)
- Related: [CQRS](../data-patterns/cqrs.md), [Saga Pattern](../distributed-transactions/saga.md), [Transactional Outbox](../messaging/transactional-outbox.md), [Event Sourcing](./event-sourcing.md)
