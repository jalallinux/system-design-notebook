# Event-Driven Architecture

## 1. Introduction

Event-Driven Architecture (EDA) is a software design pattern where system components communicate by producing and consuming events. An event represents a significant change in state or an occurrence that other parts of the system might be interested in. Unlike traditional request-response patterns, EDA promotes loose coupling and asynchronous communication between services.

### Core Concepts

**Event**: An immutable record of something that happened in the system. Events are facts about the past and cannot be changed.

```
┌─────────────────────────────────────────┐
│           Event Example                 │
├─────────────────────────────────────────┤
│ Event Type: OrderPlaced                 │
│ Timestamp:  2025-01-15T10:30:00Z       │
│ Payload:                                │
│   - orderId: 12345                      │
│   - customerId: 789                     │
│   - total: 150.00                       │
│   - items: [...]                        │
└─────────────────────────────────────────┘
```

**Event Producer**: A component that detects state changes and publishes events. Producers are decoupled from consumers and don't know who will process their events.

**Event Consumer**: A component that subscribes to and processes events. Consumers react to events by performing business logic, updating their own state, or triggering additional events.

**Event Channel/Broker**: The infrastructure that routes events from producers to consumers. This can be a message queue, event stream, or service bus.

### Basic Event Flow

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Producer   │         │    Event     │         │   Consumer   │
│   Service    │────────▶│    Broker    │────────▶│   Service    │
│              │ publish │              │subscribe│              │
└──────────────┘         └──────────────┘         └──────────────┘
                                │
                                │ publish
                                ▼
                         ┌──────────────┐
                         │   Another    │
                         │   Consumer   │
                         └──────────────┘
```

## 2. Four Event-Driven Patterns

Martin Fowler identifies four fundamental patterns in event-driven systems. Each pattern serves different use cases and has distinct characteristics.

### 2.1 Event Notification

The simplest form of EDA. A service sends an event to notify others that something happened, carrying minimal information. Consumers who need more details must call back to the source system.

**Characteristics**:
- Events carry minimal data (usually just IDs)
- Promotes decoupling - producer doesn't know what consumers do
- Consumers make additional queries for full details
- Low network traffic for event distribution
- Simple to implement

**Example Flow**:

```
Customer Service              Event Broker              Email Service
      │                            │                          │
      │ 1. User registers          │                          │
      │────────────────────────────┤                          │
      │                            │                          │
      │ 2. Publish:                │                          │
      │    UserRegistered          │                          │
      │    { userId: 123 }         │                          │
      ├───────────────────────────▶│                          │
      │                            │                          │
      │                            │ 3. Notify subscriber     │
      │                            ├─────────────────────────▶│
      │                            │                          │
      │                            │   4. GET /users/123      │
      │◀──────────────────────────────────────────────────────┤
      │                            │                          │
      │ 5. Return full user data   │                          │
      ├──────────────────────────────────────────────────────▶│
      │                            │                          │
      │                            │   6. Send welcome email  │
```

**When to Use**:
- Notifying multiple services of state changes
- When consumers need different data subsets
- Keeping event payload small
- When source data changes frequently (avoid stale data)

### 2.2 Event-Carried State Transfer

Events carry complete state information, eliminating the need for consumers to call back. This enables greater autonomy for consuming services but increases data duplication.

**Characteristics**:
- Events contain full or substantial state data
- Consumers can operate without calling back
- Reduces coupling and network calls
- Increases data duplication across services
- Risk of stale data in consumer caches

**Example Flow**:

```
Order Service                 Event Broker              Analytics Service
     │                             │                           │
     │ 1. Order placed             │                           │
     │─────────────────────────────┤                           │
     │                             │                           │
     │ 2. Publish:                 │                           │
     │    OrderPlaced {            │                           │
     │      orderId: 456,          │                           │
     │      customerId: 789,       │                           │
     │      items: [...],          │                           │
     │      total: 150.00,         │                           │
     │      shippingAddress: {...},│                           │
     │      paymentMethod: "VISA"  │                           │
     │    }                        │                           │
     ├────────────────────────────▶│                           │
     │                             │                           │
     │                             │ 3. Deliver full event     │
     │                             ├──────────────────────────▶│
     │                             │                           │
     │                             │   4. Store & analyze      │
     │                             │      (no callback needed) │
```

**When to Use**:
- Consumers need most of the data
- Reducing dependencies between services
- Building autonomous microservices
- When eventual consistency is acceptable
- Creating materialized views or read models (links to [CQRS](../data-patterns/cqrs.md))

### 2.3 Event Sourcing

State changes are stored as a sequence of events rather than just the current state. The current state is derived by replaying events from the beginning. This is the foundation of event-sourced systems.

**Characteristics**:
- Events are the source of truth
- State is rebuilt by replaying events
- Complete audit trail by design
- Enables time travel queries
- More complex to implement
- Requires event versioning strategy

**Event Store Structure**:

```
Event Store (Append-Only Log)
┌──────────────────────────────────────────────────────────────────┐
│ Stream: Account-123                                              │
├──────────────────────────────────────────────────────────────────┤
│ [1] AccountCreated      { balance: 0 }                           │
│ [2] MoneyDeposited      { amount: 1000 }                         │
│ [3] MoneyWithdrawn      { amount: 200 }                          │
│ [4] MoneyDeposited      { amount: 500 }                          │
│ [5] MoneyWithdrawn      { amount: 300 }                          │
└──────────────────────────────────────────────────────────────────┘
                              │
                              │ Replay all events
                              ▼
                    Current State: balance = 1000
```

**Rebuilding State**:

```
Event Replay Process

Initial State:     balance = 0

Apply Event [1]:   AccountCreated      → balance = 0
Apply Event [2]:   MoneyDeposited      → balance = 0 + 1000 = 1000
Apply Event [3]:   MoneyWithdrawn      → balance = 1000 - 200 = 800
Apply Event [4]:   MoneyDeposited      → balance = 800 + 500 = 1300
Apply Event [5]:   MoneyWithdrawn      → balance = 1300 - 300 = 1000

Final State:       balance = 1000
```

**Snapshots for Performance**:

```
Event Store with Snapshots
┌──────────────────────────────────────────────────────────────────┐
│ Stream: Account-123                                              │
├──────────────────────────────────────────────────────────────────┤
│ [1] AccountCreated      { balance: 0 }                           │
│ [2] MoneyDeposited      { amount: 1000 }                         │
│ ...                                                              │
│ [100] Snapshot          { balance: 5000, version: 100 }         │
│ [101] MoneyDeposited    { amount: 200 }                          │
│ [102] MoneyWithdrawn    { amount: 300 }                          │
│ ...                                                              │
│ [200] Snapshot          { balance: 7500, version: 200 }         │
│ [201] MoneyDeposited    { amount: 100 }                          │
└──────────────────────────────────────────────────────────────────┘

To rebuild current state:
1. Load latest snapshot (version 200, balance = 7500)
2. Replay events 201-end
3. Much faster than replaying all 200+ events
```

**When to Use**:
- Complete audit trail is critical (finance, healthcare)
- Need to query historical states
- Regulatory compliance requirements
- Complex domains with rich state transitions
- Building temporal reporting systems

### 2.4 CQRS (Command Query Responsibility Segregation)

CQRS separates read and write operations into different models. Commands update state through the write model, while queries retrieve data from optimized read models. This pattern often combines with Event Sourcing.

**Basic CQRS Architecture**:

```
                    Commands                     Events
┌──────────┐      (Updates)      ┌────────┐              ┌──────────┐
│          │─────────────────────▶│ Write  │─────────────▶│  Event   │
│  Client  │                      │ Model  │   Publish    │  Broker  │
│          │◀─────────────────────│        │              └──────────┘
└──────────┘      Ack             └────────┘                    │
     │                                                           │
     │                                                           │ Subscribe
     │            Queries                                        ▼
     │           (Reads)                                  ┌──────────┐
     └─────────────────────▶┌────────┐                   │  Event   │
                             │ Read   │◀──────────────────│ Handler  │
                             │ Model  │    Update Read    └──────────┘
                             └────────┘    Models
```

For a comprehensive guide on CQRS, see [CQRS Pattern](../data-patterns/cqrs.md).

**Key Points**:
- Write model optimized for consistency and business rules
- Read models optimized for queries (denormalized, cached)
- Eventually consistent read models
- Scales reads and writes independently

## 3. Event Notification vs Event-Carried State Transfer

Understanding when to use each pattern is crucial for effective EDA design.

| Aspect | Event Notification | Event-Carried State Transfer |
|--------|-------------------|------------------------------|
| **Event Size** | Small (IDs only) | Large (full state) |
| **Coupling** | Medium (consumers call back) | Low (autonomous consumers) |
| **Network Calls** | More (callbacks needed) | Fewer (self-contained) |
| **Data Freshness** | Always fresh | Potentially stale |
| **Data Duplication** | Minimal | Significant |
| **Consumer Autonomy** | Low | High |
| **Implementation Complexity** | Simple | Moderate |
| **Best For** | Notifications, alerts | Building local views, caching |
| **Bandwidth Usage** | Low for events, high for callbacks | High for events, low for callbacks |
| **Resilience** | Depends on source availability | High (works offline) |

**Decision Matrix**:

```
Use Event Notification when:
├─ Events are frequent
├─ Source data changes often
├─ Consumers need different data subsets
├─ Data freshness is critical
└─ Minimizing data duplication is important

Use Event-Carried State Transfer when:
├─ Consumers need most of the data
├─ Reducing service dependencies is priority
├─ Building microservices with autonomy
├─ Eventual consistency is acceptable
└─ Creating read replicas or projections
```

## 4. Event Sourcing Deep Dive

Event Sourcing is a sophisticated pattern that fundamentally changes how we think about state management.

### 4.1 Event Store Design

An event store is an append-only database optimized for writing events and reading event streams.

**Key Characteristics**:
- Append-only (no updates or deletes)
- Events stored in order
- Indexed by aggregate ID and version
- Supports both stream reads and global reads
- Optimistic concurrency control

**Event Store Schema**:

```
Events Table
┌────────────┬──────────────┬─────────┬───────────────┬───────────┬──────────────┐
│ Stream ID  │   Version    │  Type   │   Timestamp   │  Payload  │   Metadata   │
├────────────┼──────────────┼─────────┼───────────────┼───────────┼──────────────┤
│ Order-123  │      1       │ Created │ 2025-01-15... │  {...}    │   {...}      │
│ Order-123  │      2       │ Updated │ 2025-01-15... │  {...}    │   {...}      │
│ Order-123  │      3       │ Shipped │ 2025-01-16... │  {...}    │   {...}      │
│ Order-456  │      1       │ Created │ 2025-01-16... │  {...}    │   {...}      │
└────────────┴──────────────┴─────────┴───────────────┴───────────┴──────────────┘

Indexes:
- PRIMARY: (Stream ID, Version)
- INDEX: Timestamp (for global ordering)
- INDEX: Type (for event type queries)
```

### 4.2 Aggregate Patterns

```
┌─────────────────────────────────────────────────────────────┐
│                    Order Aggregate                          │
├─────────────────────────────────────────────────────────────┤
│  State:                                                     │
│    - orderId                                                │
│    - status: PENDING | PAID | SHIPPED | DELIVERED          │
│    - items: [...]                                           │
│    - totalAmount                                            │
│                                                             │
│  Commands:                                                  │
│    - createOrder()      → OrderCreated event                │
│    - payOrder()         → OrderPaid event                   │
│    - shipOrder()        → OrderShipped event                │
│    - deliverOrder()     → OrderDelivered event              │
│                                                             │
│  Business Rules:                                            │
│    - Can only pay pending orders                            │
│    - Can only ship paid orders                              │
│    - Cannot cancel shipped orders                           │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 Handling Commands

```
Command Processing Flow

Client                 Command Handler           Event Store
  │                           │                        │
  │  1. PayOrder(123)         │                        │
  ├──────────────────────────▶│                        │
  │                           │                        │
  │                           │ 2. Load events         │
  │                           │    for Order-123       │
  │                           ├───────────────────────▶│
  │                           │                        │
  │                           │ 3. Return events       │
  │                           │    [Created, ...]      │
  │                           │◀───────────────────────┤
  │                           │                        │
  │                           │ 4. Replay events       │
  │                           │    to rebuild state    │
  │                           │                        │
  │                           │ 5. Execute command     │
  │                           │    (business logic)    │
  │                           │                        │
  │                           │ 6. Generate new event  │
  │                           │    OrderPaid           │
  │                           │                        │
  │                           │ 7. Append event        │
  │                           │    (with version check)│
  │                           ├───────────────────────▶│
  │                           │                        │
  │                           │ 8. Success             │
  │                           │◀───────────────────────┤
  │                           │                        │
  │ 9. Acknowledge            │                        │
  │◀──────────────────────────┤                        │
```

### 4.4 Projections and Read Models

Events are replayed to build various read models optimized for queries.

```
Event Store                   Projection Handler           Read Database
    │                               │                            │
    │ OrderCreated                  │                            │
    ├──────────────────────────────▶│                            │
    │                               │ Create order record        │
    │                               ├───────────────────────────▶│
    │                               │                            │
    │ OrderPaid                     │                            │
    ├──────────────────────────────▶│                            │
    │                               │ Update status = PAID       │
    │                               ├───────────────────────────▶│
    │                               │                            │
    │ OrderShipped                  │                            │
    ├──────────────────────────────▶│                            │
    │                               │ Update status = SHIPPED    │
    │                               ├───────────────────────────▶│
```

**Multiple Projections**:

```
                        Event Stream
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │   Current    │  │   Analytics  │  │    Audit     │
    │    Orders    │  │     View     │  │     Log      │
    │   (SQL DB)   │  │  (Elastic)   │  │  (Archive)   │
    └──────────────┘  └──────────────┘  └──────────────┘
```

### 4.5 Temporal Queries

One of the most powerful features of event sourcing is querying state at any point in time.

```
Query: "What was the account balance on 2025-01-10?"

Event Store: Account-123
[1] 2025-01-01  AccountCreated    { balance: 0 }
[2] 2025-01-05  MoneyDeposited    { amount: 1000 }
[3] 2025-01-08  MoneyWithdrawn    { amount: 200 }
[4] 2025-01-10  MoneyDeposited    { amount: 500 }        ← Stop here
[5] 2025-01-12  MoneyWithdrawn    { amount: 300 }        ← Ignore future
[6] 2025-01-15  MoneyDeposited    { amount: 100 }        ← Ignore future

Replay events 1-4:
0 → +1000 → -200 → +500 = 1300

Answer: Balance was 1300 on 2025-01-10
```

### 4.6 Event Versioning

As systems evolve, event schemas change. Handle this carefully.

**Versioning Strategies**:

```
Strategy 1: Version Field
{
  "eventType": "OrderCreated",
  "version": 2,
  "data": {
    "orderId": "123",
    "items": [...],
    "shippingAddress": {...}  // Added in v2
  }
}

Strategy 2: Separate Event Types
OrderCreatedV1
OrderCreatedV2

Strategy 3: Upcasting
Read old event → Transform to current schema → Process
```

## 5. Event Delivery Guarantees

Understanding delivery semantics is critical for building reliable event-driven systems.

### 5.1 At-Most-Once Delivery

Events are delivered zero or one time. May be lost but never duplicated.

```
Producer              Broker              Consumer
   │                    │                    │
   │ 1. Send Event      │                    │
   ├───────────────────▶│                    │
   │                    │ 2. Deliver         │
   │                    ├───────────────────▶│
   │                    │                    │ 3. Process
   │                    │                    │    (Crash!)
   │                    │                    │
   │                    │ 4. Event lost      │
```

**Characteristics**:
- Lowest latency and overhead
- Risk of data loss
- No duplicate processing

**Use Cases**:
- Metrics and monitoring
- Non-critical notifications
- High-throughput scenarios where occasional loss is acceptable

### 5.2 At-Least-Once Delivery

Events are delivered one or more times. May be duplicated but never lost.

```
Producer              Broker              Consumer
   │                    │                    │
   │ 1. Send Event      │                    │
   ├───────────────────▶│                    │
   │                    │ 2. Deliver         │
   │                    ├───────────────────▶│
   │                    │                    │ 3. Process
   │                    │                    │    (Crash before ACK!)
   │                    │                    │
   │                    │ 4. Redeliver       │
   │                    │    (Broker timeout)│
   │                    ├───────────────────▶│
   │                    │                    │ 5. Process again
   │                    │                    │    (Duplicate!)
   │                    │ 6. ACK             │
   │                    │◀───────────────────┤
```

**Characteristics**:
- No data loss
- Possible duplicates
- Requires idempotent consumers

**Use Cases**:
- Most business events
- Financial transactions (with idempotency)
- Order processing

**Implementing Idempotency**:

```
Idempotent Consumer Pattern

┌─────────────────────────────────────────┐
│  Event: OrderCreated { orderId: 123 }   │
└─────────────────────────────────────────┘
                  │
                  ▼
          ┌──────────────┐
          │   Consumer   │
          └──────────────┘
                  │
                  │ 1. Check processed table
                  ▼
    ┌───────────────────────────┐
    │ Already processed 123?    │
    └───────────────────────────┘
           │              │
           │ No           │ Yes
           ▼              ▼
    ┌──────────┐    ┌──────────┐
    │ Process  │    │  Skip    │
    │  Event   │    │  (Safe)  │
    └──────────┘    └──────────┘
           │
           │ 2. Store event ID
           ▼
    ┌──────────────────┐
    │ Processed Events │
    │  - 123           │
    └──────────────────┘
```

### 5.3 Exactly-Once Delivery

Events are delivered exactly one time. No loss, no duplicates.

**Note**: True exactly-once is theoretically impossible in distributed systems. What we actually achieve is "effectively once" or "exactly-once processing semantics."

**Approaches**:

```
Approach 1: Transactional Outbox Pattern

Application DB              Message Broker
┌────────────────┐          ┌────────────┐
│ Business Data  │          │            │
│                │          │            │
│ Outbox Table   │          │            │
│  - event_id    │          │            │
│  - payload     │          │            │
│  - published   │          │            │
└────────────────┘          └────────────┘
        │                          ▲
        │ 1. Insert both           │
        │    in transaction        │ 3. Publish & mark
        │                          │
        │ 2. Committed?            │
        └──────────────────────────┘

Approach 2: Idempotent Producer + Idempotent Consumer
- Producer deduplicates sends
- Consumer deduplicates processing
- Together achieve exactly-once semantics
```

**Comparison**:

| Guarantee | Latency | Complexity | Data Loss | Duplicates | Use Case |
|-----------|---------|------------|-----------|------------|----------|
| At-Most-Once | Low | Simple | Possible | No | Metrics, logs |
| At-Least-Once | Medium | Moderate | No | Possible | Most events |
| Exactly-Once | High | Complex | No | No | Critical transactions |

## 6. Message Brokers vs Event Streams

Two primary infrastructure choices for EDA. For detailed comparison, see [RabbitMQ vs Kafka](../messaging/rabbitmq-vs-kafka.md).

### 6.1 Message Brokers (e.g., RabbitMQ, ActiveMQ)

**Architecture**:

```
┌──────────┐         ┌─────────────┐         ┌──────────┐
│ Producer │────────▶│   Exchange  │────────▶│  Queue   │
└──────────┘         └─────────────┘         └──────────┘
                            │                      │
                            │                      │
                            ▼                      ▼
                     ┌─────────────┐         ┌──────────┐
                     │   Queue     │         │ Consumer │
                     └─────────────┘         └──────────┘
                            │
                            ▼
                     ┌──────────┐
                     │ Consumer │
                     └──────────┘
```

**Characteristics**:
- Messages deleted after consumption
- Push-based delivery
- Complex routing (topics, exchanges)
- Message-oriented middleware
- Traditional queue semantics

### 6.2 Event Streams (e.g., Apache Kafka, AWS Kinesis)

**Architecture**:

```
Topic: OrderEvents
┌─────────────────────────────────────────────────────┐
│ Partition 0: [e1][e2][e3][e4][e5][e6][e7]          │
│ Partition 1: [e8][e9][e10][e11][e12]               │
│ Partition 2: [e13][e14][e15][e16][e17][e18]        │
└─────────────────────────────────────────────────────┘
                    │           │
        ┌───────────┘           └───────────┐
        ▼                                    ▼
┌──────────────┐                     ┌──────────────┐
│  Consumer A  │                     │  Consumer B  │
│  Offset: 5   │                     │  Offset: 3   │
└──────────────┘                     └──────────────┘
```

**Characteristics**:
- Events retained for configured period
- Pull-based consumption
- Ordered within partitions
- Multiple consumers can replay events
- Log-based storage

**Key Differences**:

| Aspect | Message Broker | Event Stream |
|--------|----------------|--------------|
| **Storage** | Temporary (deleted after consumption) | Persistent (configurable retention) |
| **Consumption** | Push-based | Pull-based |
| **Replay** | Not supported | Supported |
| **Ordering** | Queue-level | Partition-level |
| **Throughput** | Moderate | Very high |
| **Use Case** | Task distribution, RPC | Event sourcing, streaming |

## 7. Choreography vs Orchestration

Two patterns for coordinating multi-step processes. Closely related to the [Saga Pattern](../distributed-transactions/saga.md).

### 7.1 Choreography

Services react to events without central coordination. Each service knows what to do when events occur.

```
Event-Driven Choreography (Saga)

Order Service         Payment Service      Inventory Service     Shipping Service
     │                      │                     │                    │
     │ 1. OrderCreated      │                     │                    │
     ├─────────────────────▶│                     │                    │
     │                      │                     │                    │
     │                      │ 2. PaymentProcessed │                    │
     │                      ├────────────────────▶│                    │
     │                      │                     │                    │
     │                      │                     │ 3. InventoryReserved
     │                      │                     ├────────────────────▶│
     │                      │                     │                    │
     │                      │                     │  4. ShippingScheduled
     │◀──────────────────────────────────────────────────────────────┤
     │ 5. OrderConfirmed    │                     │                    │
```

**Advantages**:
- Loose coupling
- No single point of failure
- Services independently deployable
- Natural fit for event-driven systems

**Disadvantages**:
- Harder to understand end-to-end flow
- Difficult to track saga state
- No central monitoring point
- Complex error handling

### 7.2 Orchestration

A central orchestrator directs the process, calling services in sequence.

```
Orchestrated Saga

                      Order Orchestrator
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
  ┌────────────┐      ┌────────────┐     ┌────────────┐
  │  Payment   │      │ Inventory  │     │  Shipping  │
  │  Service   │      │  Service   │     │  Service   │
  └────────────┘      └────────────┘     └────────────┘

Flow:
1. Orchestrator → Payment Service: Process payment
2. Orchestrator → Inventory Service: Reserve items
3. Orchestrator → Shipping Service: Schedule shipment
4. Orchestrator → Order Service: Mark complete
```

**Advantages**:
- Clear process definition
- Centralized monitoring
- Easier to understand
- Explicit error handling

**Disadvantages**:
- Orchestrator is single point of failure
- Tighter coupling
- Orchestrator can become complex

**When to Choose**:

```
Choose Choreography when:
├─ Services highly autonomous
├─ Process is simple and linear
├─ Loose coupling is priority
└─ Building microservices ecosystem

Choose Orchestration when:
├─ Complex business process
├─ Need central visibility
├─ Explicit error handling required
└─ Transactional guarantees needed
```

For detailed implementation patterns, see [Saga Pattern](../distributed-transactions/saga.md).

## 8. Trade-offs

### 8.1 Benefits of Event-Driven Architecture

**1. Loose Coupling**

Services don't need to know about each other. Producers and consumers are independent.

```
Traditional Coupling:
Order Service ──calls──▶ Inventory Service
             ──calls──▶ Email Service
             ──calls──▶ Analytics Service

Event-Driven Decoupling:
Order Service ──publishes──▶ Event Broker
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
              Inventory       Email       Analytics
               Service       Service       Service
```

**2. Scalability**

Components scale independently. Event brokers handle load buffering.

**3. Extensibility**

Add new consumers without modifying producers.

```
Original System:
Producer ──▶ Broker ──▶ Consumer A
                   └───▶ Consumer B

Add New Feature (no changes to producer):
Producer ──▶ Broker ──▶ Consumer A
                   ├───▶ Consumer B
                   └───▶ Consumer C (NEW!)
```

**4. Resilience**

Temporary failures don't cascade. Asynchronous processing isolates failures.

**5. Audit Trail**

Event logs provide complete history of system changes (especially with Event Sourcing).

**6. Real-time Processing**

Events flow immediately to interested consumers, enabling real-time reactions.

**7. Flexibility**

Same events can power multiple use cases: analytics, notifications, caching, etc.

### 8.2 Drawbacks and Challenges

**1. Complexity**

More moving parts: event brokers, schemas, consumers, error handling.

```
Traditional:
Client → Service → Database
(3 components)

Event-Driven:
Client → Service → Event Broker → Consumer → Database
       → Event Store → Projection Builder → Read DB
(8+ components)
```

**2. Eventual Consistency**

Related to [CAP Theorem](../fundamentals/cap-theorem.md). Events propagate over time, creating temporary inconsistencies.

```
Time: T0
Order Service: Order status = PAID
Email Service: No record yet (event in transit)

Time: T1 (50ms later)
Order Service: Order status = PAID
Email Service: Order status = PAID (event processed)
```

**3. Debugging Difficulty**

Tracing requests across asynchronous boundaries is challenging.

**4. Event Schema Evolution**

Changing event structures requires versioning and backward compatibility.

**5. Event Ordering**

Guaranteeing global order is expensive. Usually only partition-level ordering.

**6. Duplicate Events**

At-least-once delivery requires idempotent consumers.

**7. Operational Overhead**

Managing event brokers, monitoring consumers, handling dead-letter queues.

**8. Testing Complexity**

Asynchronous testing is harder than synchronous testing.

### 8.3 Trade-off Summary

| Aspect | Traditional | Event-Driven |
|--------|-------------|--------------|
| **Coupling** | Tight | Loose |
| **Consistency** | Strong | Eventual |
| **Scalability** | Limited | High |
| **Complexity** | Low | High |
| **Debugging** | Easier | Harder |
| **Latency** | Predictable | Variable |
| **Audit Trail** | Manual | Built-in |
| **Failure Isolation** | Poor | Good |
| **Testing** | Straightforward | Complex |

## 9. When to Use / When to Avoid

### 9.1 Use Event-Driven Architecture When

**1. Building Microservices**

EDA naturally supports [Microservices](../architecture/microservices.md) autonomy and loose coupling.

**2. Real-time Processing**

User activity tracking, fraud detection, recommendation engines.

**3. System Integration**

Connecting multiple systems without tight coupling.

**4. Audit Requirements**

Financial systems, healthcare, compliance-heavy domains.

**5. Asynchronous Workflows**

Order processing, batch jobs, long-running processes.

**6. Event Sourcing Benefits**

Temporal queries, complete history, reproducible state.

**7. Scale Requirements**

High throughput, independent scaling of components.

**8. Multiple Downstream Consumers**

Same data needed by analytics, notifications, caching, etc.

### 9.2 Avoid Event-Driven Architecture When

**1. Simple CRUD Applications**

Traditional request-response is simpler and sufficient.

**2. Strong Consistency Required**

If you cannot tolerate eventual consistency, EDA adds complexity. Consider [Two-Phase Commit](../distributed-transactions/two-phase-commit.md) for distributed transactions.

**3. Low Latency Critical**

Synchronous calls have predictable latency; events add overhead.

**4. Small Teams**

Operational overhead may overwhelm small teams.

**5. Debugging/Testing Priority**

If debuggability is paramount, synchronous flows are easier.

**6. Limited Infrastructure**

Running and managing event brokers requires resources.

**7. Single Consumer**

If only one service needs the data, direct calls may be simpler.

**8. Prototyping Phase**

Start simple; introduce EDA when complexity justifies it.

### 9.3 Hybrid Approach

Many systems use both:

```
Hybrid Architecture

┌────────────────────────────────────────────────────┐
│                     System                         │
├────────────────────────────────────────────────────┤
│                                                    │
│  Synchronous:                                      │
│    ├─ User authentication (immediate response)    │
│    ├─ Payment processing (strong consistency)     │
│    └─ API Gateway routing                         │
│                                                    │
│  Asynchronous (Event-Driven):                     │
│    ├─ Order processing workflow                   │
│    ├─ Analytics and reporting                     │
│    ├─ Email notifications                         │
│    └─ Cache invalidation                          │
│                                                    │
└────────────────────────────────────────────────────┘
```

## 10. Real-World Examples

### 10.1 Event Sourcing in Banking

**Use Case**: Account management with complete audit trail.

```
Bank Account System with Event Sourcing

Event Store: Account-987654
┌─────────────────────────────────────────────────────────────┐
│ [1] 2025-01-01 10:00  AccountOpened                         │
│     { customerId: "C123", initialDeposit: 10000 }           │
│                                                             │
│ [2] 2025-01-05 14:30  InterestCredited                      │
│     { amount: 150, rate: 0.015 }                            │
│                                                             │
│ [3] 2025-01-10 09:15  WithdrawalMade                        │
│     { amount: 2000, atmId: "ATM-456", location: "NYC" }     │
│                                                             │
│ [4] 2025-01-15 16:45  DepositMade                           │
│     { amount: 5000, branch: "Downtown" }                    │
│                                                             │
│ [5] 2025-01-20 11:00  FraudAlertRaised                      │
│     { reason: "Unusual pattern", suspended: true }          │
│                                                             │
│ [6] 2025-01-21 10:30  FraudAlertCleared                     │
│     { verifiedBy: "Agent-789", suspended: false }           │
└─────────────────────────────────────────────────────────────┘

Benefits:
✓ Complete audit trail for regulatory compliance
✓ Reconstruct account state at any point in history
✓ Replay events to test new business rules
✓ Generate compliance reports from event history
✓ Investigate fraud by examining event sequences
```

**Regulatory Query Example**:

```
Query: "Show all transactions over $10,000 in Q1 2025"

Filter events:
- Type: DepositMade OR WithdrawalMade
- Date: 2025-01-01 to 2025-03-31
- Amount > 10000

Results:
[1] 2025-01-01  AccountOpened     $10,000 (initial)
[4] 2025-01-15  DepositMade       $5,000  ← Below threshold
[7] 2025-02-10  DepositMade       $15,000 ← Match!
[9] 2025-03-05  WithdrawalMade    $12,000 ← Match!
```

### 10.2 E-commerce Order Processing

**Use Case**: Order workflow with multiple services coordinated via events.

```
Event-Driven Order Processing Flow

Step 1: Order Created
Customer → Order Service
              │
              │ OrderCreated {
              │   orderId: "O-123",
              │   items: [...],
              │   total: $150
              │ }
              ▼
        Event Broker
              │
              ├────────────────────┬────────────────────┬───────────────┐
              ▼                    ▼                    ▼               ▼
        Payment         Inventory          Email           Analytics
        Service         Service            Service         Service

Step 2: Payment Processed
Payment Service → Event Broker
                      │
                      │ PaymentProcessed {
                      │   orderId: "O-123",
                      │   amount: $150,
                      │   method: "VISA"
                      │ }
                      ▼
              ┌───────┴────────┐
              ▼                ▼
        Inventory          Email
        (Reserve)         (Receipt)

Step 3: Inventory Reserved
Inventory Service → Event Broker
                      │
                      │ InventoryReserved {
                      │   orderId: "O-123",
                      │   warehouse: "NYC"
                      │ }
                      ▼
              ┌───────┴────────┐
              ▼                ▼
        Shipping           Email
        (Schedule)       (Confirmation)

Step 4: Shipped
Shipping Service → Event Broker
                      │
                      │ OrderShipped {
                      │   orderId: "O-123",
                      │   trackingNumber: "TRACK-456",
                      │   estimatedDelivery: "2025-01-20"
                      │ }
                      ▼
              ┌───────┴────────┐
              ▼                ▼
         Order              Email
         (Update)          (Tracking)
```

**Compensation Flow (Payment Failed)**:

```
Compensation via Events (Saga Pattern)

Order Service → PaymentService → Event Broker
                                      │
                                      │ PaymentFailed {
                                      │   orderId: "O-123",
                                      │   reason: "Insufficient funds"
                                      │ }
                                      ▼
                              ┌───────┴────────┐
                              ▼                ▼
                        Order Service      Email Service
                              │                │
                              │ OrderCancelled │ Cancellation
                              │                │ Email
```

For detailed saga implementation, see [Saga Pattern](../distributed-transactions/saga.md).

### 10.3 Netflix - Event-Driven Recommendation Engine

**Use Case**: Real-time recommendations based on user behavior.

```
Netflix Viewing Events

User watches video → Event Stream
                          │
                          │ VideoWatched {
                          │   userId: "U-789",
                          │   videoId: "V-456",
                          │   duration: 3600,
                          │   completionRate: 0.95,
                          │   timestamp: "..."
                          │ }
                          ▼
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
  Recommendation     Analytics       Content
     Engine           Service        Service
        │                              │
        │ Update user                  │ Track
        │ preferences                  │ popularity
        │                              │
        ▼                              ▼
  Personalized                    Trending
  Homepage                        Content
```

**Benefits**:
- Real-time updates to recommendations
- Multiple teams consume same events
- Scale viewing events (billions per day)
- A/B test different recommendation algorithms independently

### 10.4 Uber - Event-Driven Dispatch System

**Use Case**: Real-time matching of riders and drivers.

```
Event-Driven Dispatch

Driver Location Updates (every 4 seconds)
Driver App → Event Stream
                  │
                  │ DriverLocationUpdated {
                  │   driverId: "D-123",
                  │   lat: 40.7589,
                  │   lng: -73.9851,
                  │   status: "AVAILABLE",
                  │   timestamp: "..."
                  │ }
                  ▼
          ┌───────┴────────┐
          ▼                ▼
     Dispatch          Analytics
     Service           Service
          │
          │ Maintain real-time
          │ driver positions
          │
          ▼
     Matching Algorithm

Rider Request:
Rider App → Dispatch Service
                  │
                  │ Find nearby available drivers
                  │ (from cached positions)
                  │
                  ▼
            Match Found!
                  │
                  │ RideMatched {
                  │   riderId: "R-456",
                  │   driverId: "D-123",
                  │   estimatedArrival: 3
                  │ }
                  ▼
          Event Broker
                  │
          ┌───────┼────────┐
          ▼       ▼        ▼
      Driver  Rider   Notification
       App     App     Service
```

## 11. Related Patterns

Event-Driven Architecture connects with many other patterns:

**1. CQRS (Command Query Responsibility Segregation)**

See [CQRS Pattern](../data-patterns/cqrs.md) for separating read and write models using events.

**2. Saga Pattern**

See [Saga Pattern](../distributed-transactions/saga.md) for managing distributed transactions with events.

**3. Two-Phase Commit**

See [Two-Phase Commit](../distributed-transactions/two-phase-commit.md) for an alternative to event-based distributed transactions.

**4. Message Broker Selection**

See [RabbitMQ vs Kafka](../messaging/rabbitmq-vs-kafka.md) for choosing the right event infrastructure.

**5. Microservices Architecture**

EDA is a natural fit for [Microservices](../architecture/microservices.md) communication.

**6. API Gateway**

[API Gateway](../architecture/api-gateway.md) often triggers events for internal processing.

**7. Circuit Breaker**

[Circuit Breaker](../resilience/circuit-breaker.md) helps prevent cascading failures in event consumers.

**8. CAP Theorem**

EDA typically prioritizes availability and partition tolerance. See [CAP Theorem](../fundamentals/cap-theorem.md).

**Pattern Relationships**:

```
                    Event-Driven Architecture
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
         Microservices      CQRS          Saga
              │               │               │
              │               │               │
        ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐
        ▼           ▼   ▼           ▼   ▼           ▼
    API Gateway  Circuit  Event    Read  Choreography Orchestration
                 Breaker  Sourcing Models
```

## 12. Interview Framework / Key Takeaways

When discussing Event-Driven Architecture in interviews, use this framework:

### 12.1 Core Concepts to Cover

**1. Define the Pattern**

"Event-Driven Architecture is a pattern where components communicate through events representing significant state changes. It promotes loose coupling and asynchronous processing."

**2. Four Event Patterns**

- **Event Notification**: Minimal data, callback required
- **Event-Carried State Transfer**: Full data, autonomous consumers
- **Event Sourcing**: Events as source of truth
- **CQRS**: Separate read/write models

**3. Key Trade-offs**

| Benefit | Challenge |
|---------|-----------|
| Loose coupling | Eventual consistency |
| Scalability | Complexity |
| Resilience | Debugging difficulty |
| Audit trail | Event schema evolution |
| Extensibility | Operational overhead |

### 12.2 When Designing an Event-Driven System

**Step 1: Identify Events**

```
Domain: E-commerce Order System

Events:
- OrderCreated
- PaymentProcessed
- PaymentFailed
- InventoryReserved
- InventoryUnavailable
- OrderShipped
- OrderDelivered
- OrderCancelled
```

**Step 2: Choose Event Pattern**

- Event Notification: For simple alerts
- Event-Carried State Transfer: For building local views
- Event Sourcing: For audit trail and temporal queries
- CQRS: For read/write separation

**Step 3: Select Infrastructure**

- Message Broker (RabbitMQ): For task distribution, RPC
- Event Stream (Kafka): For event sourcing, high throughput, replay

**Step 4: Define Delivery Guarantees**

- At-most-once: Metrics, non-critical
- At-least-once: Most business events (with idempotency)
- Exactly-once: Critical financial transactions

**Step 5: Design for Failure**

- Implement idempotent consumers
- Use dead-letter queues
- Add circuit breakers
- Plan compensation logic (sagas)

### 12.3 Common Interview Questions

**Q1: "How do you ensure exactly-once processing?"**

A: "True exactly-once is impossible in distributed systems due to network partitions. We achieve 'effectively once' through:
1. Idempotent consumers (check processed event IDs)
2. Transactional outbox pattern
3. Atomic database writes with event publishing
4. Deduplication at both producer and consumer"

**Q2: "How do you handle event ordering?"**

A: "Global ordering is expensive and usually unnecessary. Instead:
1. Partition events by aggregate ID (e.g., same user's events in same partition)
2. Use sequence numbers within partitions
3. Version events for optimistic concurrency
4. Accept that cross-partition events may arrive out of order"

**Q3: "Event Sourcing vs traditional database?"**

A: "Event Sourcing stores events, not state:
- Pros: Complete audit trail, temporal queries, replay capability
- Cons: Complexity, eventual consistency for reads, event versioning
- Use when: Audit critical, temporal queries needed, compliance requirements
- Avoid when: Simple CRUD, strong consistency required, small team"

**Q4: "How do you debug event-driven systems?"**

A: "Debugging is challenging. Key techniques:
1. Correlation IDs across all events
2. Distributed tracing (Zipkin, Jaeger)
3. Centralized logging with event IDs
4. Event replay in test environments
5. Monitoring dashboards for consumer lag
6. Dead-letter queue analysis"

**Q5: "Choreography vs Orchestration?"**

A: "Both coordinate multi-step workflows:
- Choreography: Services react to events (loose coupling, harder to monitor)
- Orchestration: Central controller (easier to understand, single point of failure)
- Choice depends on: process complexity, team structure, monitoring needs
- Often use both: choreography for simple flows, orchestration for complex sagas"

### 12.4 Design Decision Checklist

When proposing EDA, address:

- [ ] Have you identified all domain events?
- [ ] Which event pattern (notification, state transfer, sourcing)?
- [ ] What delivery guarantee is needed?
- [ ] How will you handle duplicates?
- [ ] What's your event schema evolution strategy?
- [ ] How will you monitor consumer lag?
- [ ] What's your failure recovery plan?
- [ ] Is eventual consistency acceptable?
- [ ] Have you considered operational overhead?
- [ ] What's your testing strategy?

### 12.5 Key Takeaways

1. **EDA promotes loose coupling** through asynchronous event communication
2. **Four patterns** serve different needs: notification, state transfer, sourcing, CQRS
3. **Trade consistency for availability** - accept eventual consistency
4. **Idempotency is critical** for at-least-once delivery
5. **Choose infrastructure carefully** - message brokers vs event streams
6. **Choreography vs orchestration** - different coordination approaches
7. **Monitoring is essential** - distributed tracing, correlation IDs, consumer lag
8. **Start simple** - introduce EDA when complexity justifies it
9. **Real-world success** - Netflix, Uber, banks use EDA at massive scale
10. **Combine with other patterns** - CQRS, Saga, Microservices, Circuit Breaker

### 12.6 Further Reading

**Books**:
- "Designing Data-Intensive Applications" by Martin Kleppmann (Chapter 11: Stream Processing)
- "Enterprise Integration Patterns" by Gregor Hohpe
- "Building Event-Driven Microservices" by Adam Bellemare
- "Implementing Domain-Driven Design" by Vaughn Vernon (Event Sourcing chapter)

**Resources**:
- Martin Fowler: "What do you mean by Event-Driven?"
- Greg Young: "Event Sourcing" talks
- Kafka documentation: Event streaming concepts
- AWS EventBridge / Azure Event Grid documentation

**Company Engineering Blogs**:
- Netflix Tech Blog: Event-driven architecture at scale
- Uber Engineering: Real-time dispatch system
- LinkedIn: Kafka usage patterns
- Amazon: Event-driven microservices

---

**Navigation**:
- Previous: [CQRS Pattern](../data-patterns/cqrs.md)
- Next: [Saga Pattern](../distributed-transactions/saga.md)
- Related: [Microservices](../architecture/microservices.md), [CAP Theorem](../fundamentals/cap-theorem.md)
