# Event Sourcing

## 1. Introduction

**Event Sourcing** is an architectural pattern that persists the state of an aggregate as a sequence of state-changing events, rather than storing only the current state. Instead of updating a record in a database, every change is captured as an immutable event appended to an event store. The current state of any entity is derived by replaying its events from the beginning (or from a snapshot).

### Origin and Evolution

The Event Sourcing pattern was popularized by **Greg Young** in the late 2000s, drawing inspiration from concepts that have existed for centuries in accounting ledgers and transaction logs. In double-entry bookkeeping, every financial transaction is recorded as an immutable entry -- the balance is always the result of summing all entries. Similarly, database transaction logs (Write-Ahead Logs) have long used append-only records to guarantee durability and recovery.

Greg Young formalized these ideas into a software architecture pattern and demonstrated how Event Sourcing pairs naturally with [CQRS](../data-patterns/cqrs.md) to build scalable, auditable, and highly flexible systems.

### Traditional State Storage vs Event Sourcing

```
Traditional State Storage:
┌──────────────────────────────────────────────────┐
│  Account Table                                    │
│──────────────────────────────────────────────────│
│  account_id  │  owner     │  balance  │  status  │
│──────────────┼────────────┼───────────┼──────────│
│  ACC-001     │  Alice     │  1300     │  ACTIVE  │
│  ACC-002     │  Bob       │  750      │  ACTIVE  │
└──────────────────────────────────────────────────┘
  Only current state is stored. History is lost.


Event Sourcing:
┌──────────────────────────────────────────────────────────────┐
│  Event Store (Append-Only Log) — Stream: ACC-001             │
│──────────────────────────────────────────────────────────────│
│  [1] AccountOpened       { owner: "Alice", balance: 0 }      │
│  [2] MoneyDeposited      { amount: 1000 }                    │
│  [3] MoneyWithdrawn      { amount: 200 }                     │
│  [4] MoneyDeposited      { amount: 500 }                     │
│──────────────────────────────────────────────────────────────│
│  Current State = Replay all events                           │
│  → balance: 0 + 1000 - 200 + 500 = 1300                     │
└──────────────────────────────────────────────────────────────┘
  Every change recorded. Full history preserved.
```

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Events are immutable** | Once written, events are never modified or deleted |
| **Append-only storage** | New events are only appended; no updates or deletes |
| **Events are the source of truth** | Current state is derived, not stored directly |
| **Events represent facts** | Each event records something that happened in the past |
| **Replay to reconstruct** | State at any point in time can be reconstructed by replaying events |

---

## 2. Context and Problem

### Context

In modern distributed systems, especially those built around [microservices](../architecture/microservices.md) and [event-driven architectures](./event-driven-architecture.md), several challenges repeatedly arise:

1. **Audit trail requirements** -- Regulatory and business requirements demand a complete, tamper-proof history of all state changes (finance, healthcare, legal).
2. **Temporal queries** -- The need to answer questions like "What was the account balance on January 10th?" or "What was the order status before the last change?"
3. **Reliable event publishing** -- When state changes, other services need to be notified reliably. Traditional approaches risk losing events or creating inconsistencies between state and published events.
4. **Debugging and forensics** -- Understanding how the system arrived at a particular state, especially when investigating bugs or incidents.

### Problem

How do you reliably persist state changes and simultaneously publish events to other services, while maintaining a complete and trustworthy history of all changes?

In a traditional CRUD approach:

```
Traditional CRUD Problem:

1. Update database       ← State changes here
2. Publish event         ← Notification happens here

What if step 2 fails?
 → State changed but no event published
 → Other services are out of sync
 → No reliable history of changes

What if you publish first?
1. Publish event
2. Update database       ← What if this fails?
 → Event published but state not saved
 → Inconsistency!
```

This is fundamentally a **dual-write problem**: you need to atomically update state AND publish an event, but these are two separate operations that cannot easily be wrapped in a single transaction.

---

## 3. Forces

Several competing forces drive the need for Event Sourcing:

| Force | Description |
|-------|-------------|
| **Atomicity** | State change and event publishing must happen atomically -- either both succeed or neither does |
| **Complete audit trail** | Every state change must be recorded permanently for compliance, debugging, or business intelligence |
| **Temporal queries** | The system must support querying the state of an entity at any point in the past |
| **Reliable event publishing** | Events must be published reliably whenever state changes, with no lost or phantom events |
| **Performance** | Reads and writes have different performance characteristics and scaling needs |
| **Schema evolution** | The system must handle changes to event structure over time without losing historical data |
| **Complexity budget** | The solution must be justifiable given the added complexity |

### Force Interactions

```
                    ┌──────────────────────┐
                    │   Event Sourcing     │
                    │   resolves these     │
                    │   competing forces   │
                    └──────────┬───────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
  ┌──────────────┐   ┌──────────────────┐   ┌──────────────┐
  │  Atomicity   │   │  Audit Trail +   │   │  Reliable    │
  │  of state +  │   │  Temporal Queries │   │  Event       │
  │  event       │   │                  │   │  Publishing  │
  └──────────────┘   └──────────────────┘   └──────────────┘
         │                     │                     │
         │                     │                     │
         ▼                     ▼                     ▼
  Events ARE the         History is the        Events are stored
  state change           natural byproduct     first, then
  (single write)         of the event log      projected/published
```

---

## 4. Solution

### Core Idea

Instead of storing the current state of an entity, **store every state change as an immutable event** in an append-only event store. The current state of any entity is derived by loading and replaying its event stream.

### Event Store

The event store is a specialized, append-only database that serves as the single source of truth for the system.

```
Event Store Architecture
┌──────────────────────────────────────────────────────────────────┐
│                         EVENT STORE                              │
│                    (Append-Only Database)                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Stream: ShoppingCart-42                                          │
│  ┌───────┬──────────────────┬───────────────┬──────────────────┐ │
│  │ Ver.  │   Event Type     │   Timestamp   │    Payload       │ │
│  ├───────┼──────────────────┼───────────────┼──────────────────┤ │
│  │  1    │ CartCreated      │ 2025-01-15T09 │ { userId: 42 }   │ │
│  │  2    │ ItemAdded        │ 2025-01-15T09 │ { sku: "A1" }    │ │
│  │  3    │ ItemAdded        │ 2025-01-15T09 │ { sku: "B2" }    │ │
│  │  4    │ ItemRemoved      │ 2025-01-15T10 │ { sku: "A1" }    │ │
│  │  5    │ CartCheckedOut   │ 2025-01-15T10 │ { total: 29.99 } │ │
│  └───────┴──────────────────┴───────────────┴──────────────────┘ │
│                                                                  │
│  Stream: ShoppingCart-43                                          │
│  ┌───────┬──────────────────┬───────────────┬──────────────────┐ │
│  │  1    │ CartCreated      │ 2025-01-15T11 │ { userId: 77 }   │ │
│  │  2    │ ItemAdded        │ 2025-01-15T11 │ { sku: "C3" }    │ │
│  └───────┴──────────────────┴───────────────┴──────────────────┘ │
│                                                                  │
│  Properties:                                                     │
│  ✓ Append-only (no updates, no deletes)                         │
│  ✓ Events are immutable                                         │
│  ✓ Ordered within each stream                                   │
│  ✓ Globally ordered via position/sequence number                │
│  ✓ Optimistic concurrency via expected version                  │
└──────────────────────────────────────────────────────────────────┘
```

**Event Store Schema**:

```
┌────────────┬──────────┬──────────────────┬───────────────┬───────────┬──────────────┬──────────┐
│ Stream ID  │ Version  │   Event Type     │   Timestamp   │  Payload  │   Metadata   │ Position │
├────────────┼──────────┼──────────────────┼───────────────┼───────────┼──────────────┼──────────┤
│ Cart-42    │    1     │ CartCreated      │ 2025-01-15... │  {...}    │  {...}       │  1001    │
│ Cart-42    │    2     │ ItemAdded        │ 2025-01-15... │  {...}    │  {...}       │  1002    │
│ Cart-42    │    3     │ ItemAdded        │ 2025-01-15... │  {...}    │  {...}       │  1003    │
│ Acct-99    │    1     │ AccountOpened    │ 2025-01-15... │  {...}    │  {...}       │  1004    │
│ Cart-42    │    4     │ ItemRemoved      │ 2025-01-15... │  {...}    │  {...}       │  1005    │
└────────────┴──────────┴──────────────────┴───────────────┴───────────┴──────────────┴──────────┘

Indexes:
- PRIMARY:   (Stream ID, Version)         → read a single stream
- INDEX:     Position                      → global ordering / subscriptions
- INDEX:     Event Type                    → filter by event type
- INDEX:     Timestamp                     → time-based queries
```

### Reconstructing State

To get the current state of an aggregate, load all events for that aggregate and apply them in order:

```
Reconstructing State: ShoppingCart-42

Step 1: Load events from event store
┌──────────────────────────────────────┐
│  SELECT * FROM events                │
│  WHERE stream_id = 'Cart-42'        │
│  ORDER BY version ASC               │
└──────────────────────────────────────┘

Step 2: Apply events to empty aggregate

  Initial State:  { items: [], total: 0, status: "empty" }

  Apply [1] CartCreated:
    → { items: [], total: 0, status: "active", userId: 42 }

  Apply [2] ItemAdded { sku: "A1", price: 19.99 }:
    → { items: ["A1"], total: 19.99, status: "active" }

  Apply [3] ItemAdded { sku: "B2", price: 29.99 }:
    → { items: ["A1", "B2"], total: 49.98, status: "active" }

  Apply [4] ItemRemoved { sku: "A1", price: 19.99 }:
    → { items: ["B2"], total: 29.99, status: "active" }

  Apply [5] CartCheckedOut:
    → { items: ["B2"], total: 29.99, status: "checked_out" }

Step 3: Current state is the result of replaying all events
```

### Command Processing Flow

```
Client                  Command Handler            Aggregate              Event Store
  │                           │                        │                       │
  │  1. AddItem(Cart-42,     │                        │                       │
  │     sku="C3")            │                        │                       │
  ├──────────────────────────▶│                        │                       │
  │                           │                        │                       │
  │                           │ 2. Load events         │                       │
  │                           │    for Cart-42         │                       │
  │                           ├────────────────────────────────────────────────▶│
  │                           │                        │                       │
  │                           │ 3. Return events       │                       │
  │                           │    [1..5]              │                       │
  │                           │◀───────────────────────────────────────────────┤
  │                           │                        │                       │
  │                           │ 4. Replay events       │                       │
  │                           │    to build state      │                       │
  │                           ├───────────────────────▶│                       │
  │                           │                        │                       │
  │                           │ 5. Execute command     │                       │
  │                           │    (validate: is cart  │                       │
  │                           │     still active?)     │                       │
  │                           │                        │                       │
  │                           │ 6. REJECTED: cart is   │                       │
  │                           │    already checked out │                       │
  │                           │◀──────────────────────┤│                       │
  │                           │                        │                       │
  │  7. Error: Cart already   │                        │                       │
  │     checked out           │                        │                       │
  │◀──────────────────────────┤                        │                       │
```

### Snapshots

For aggregates with many events, replaying the entire event stream on every command becomes slow. **Snapshots** are an optimization that stores a periodic checkpoint of the aggregate state.

```
Event Store with Snapshots
┌──────────────────────────────────────────────────────────────────┐
│  Stream: Account-001                                             │
├──────────────────────────────────────────────────────────────────┤
│  [1]   AccountOpened        { balance: 0 }                       │
│  [2]   MoneyDeposited       { amount: 1000 }                     │
│  [3]   MoneyWithdrawn       { amount: 200 }                      │
│  ...                                                             │
│  [99]  MoneyDeposited       { amount: 50 }                       │
│  [100] *** SNAPSHOT ***     { balance: 5000, version: 100 }     │
│  [101] MoneyDeposited       { amount: 200 }                      │
│  [102] MoneyWithdrawn       { amount: 300 }                      │
│  ...                                                             │
│  [199] MoneyDeposited       { amount: 75 }                       │
│  [200] *** SNAPSHOT ***     { balance: 7500, version: 200 }     │
│  [201] MoneyDeposited       { amount: 100 }                      │
│  [202] MoneyWithdrawn       { amount: 50 }                       │
└──────────────────────────────────────────────────────────────────┘

Without Snapshots:                  With Snapshots:
─────────────────                   ─────────────────
Load events [1..202]                Load snapshot at [200]
Replay 202 events                   Load events [201..202]
                                    Replay 2 events

Performance:                        Performance:
O(n) where n = all events           O(1) snapshot + O(m) recent events
Slow for long-lived aggregates      Fast regardless of history length
```

**Snapshot Strategy Decision**:

```
When to create snapshots:
├─ Every N events (e.g., every 100 events)
├─ On a time schedule (e.g., daily)
├─ When load time exceeds a threshold
└─ On explicit command (admin/maintenance)

When NOT to bother with snapshots:
├─ Aggregates have few events (< 50)
├─ Aggregates are short-lived
└─ Replay is already fast enough
```

### Event Schema Evolution

As systems evolve, event schemas must change. This is one of the most challenging aspects of Event Sourcing because events are immutable -- you cannot go back and modify historical events.

**Strategy 1: Upcasting**

Transform old events to the current schema at read time:

```
Stored Event (v1):                    After Upcasting (v2):
{                                     {
  "type": "CustomerRegistered",         "type": "CustomerRegistered",
  "version": 1,                         "version": 2,
  "data": {                             "data": {
    "name": "Alice Smith"                 "firstName": "Alice",
  }                                       "lastName": "Smith",
}                                         "email": null
                                        }
                                      }

Upcaster logic:
  if (event.version == 1) {
    split name into firstName + lastName
    add email = null (new field)
    return event with version = 2
  }
```

**Strategy 2: Event Versioning**

```
Version 1 events stay as-is
Version 2 events use new schema

Event handlers understand both versions:

  handle(CustomerRegistered_v1 event) {
    // old logic
  }

  handle(CustomerRegistered_v2 event) {
    // new logic with new fields
  }
```

**Strategy 3: Weak Schema with Optional Fields**

```
{
  "type": "OrderPlaced",
  "data": {
    "orderId": "123",
    "items": [...],
    "total": 99.99,
    "couponCode": null,          // Added later, nullable
    "shippingMethod": "standard" // Added later, with default
  }
}

New fields are always optional or have defaults.
Old events without these fields are handled gracefully.
```

| Strategy | Pros | Cons |
|----------|------|------|
| **Upcasting** | Clean current schema, one handler version | Upcasting logic grows over time |
| **Event Versioning** | Explicit, clear history | Multiple handlers to maintain |
| **Weak Schema** | Simplest approach | Less type safety, implicit contracts |

---

## 5. Example

### Example 1: Banking Account

Every financial transaction is stored as an event. The account balance is always the result of replaying all events.

```
Event Store: Account-5678

[1] 2025-01-01  AccountOpened
    { customerId: "C-100", initialDeposit: 0, accountType: "checking" }

[2] 2025-01-02  MoneyDeposited
    { amount: 5000, source: "wire_transfer", reference: "REF-001" }

[3] 2025-01-05  MoneyWithdrawn
    { amount: 1500, channel: "ATM", location: "NYC-42nd-St" }

[4] 2025-01-10  MoneyDeposited
    { amount: 3000, source: "direct_deposit", employer: "Acme Corp" }

[5] 2025-01-15  TransferSent
    { amount: 500, toAccount: "ACC-9999", memo: "rent" }

[6] 2025-01-20  InterestApplied
    { amount: 12.50, rate: 0.025, period: "monthly" }
```

**Replaying to compute current balance**:

```
Event [1] AccountOpened:     balance = 0
Event [2] MoneyDeposited:    balance = 0 + 5000      = 5000
Event [3] MoneyWithdrawn:    balance = 5000 - 1500    = 3500
Event [4] MoneyDeposited:    balance = 3500 + 3000    = 6500
Event [5] TransferSent:      balance = 6500 - 500     = 6000
Event [6] InterestApplied:   balance = 6000 + 12.50   = 6012.50

Current Balance: $6,012.50
```

**Temporal Query: "What was the balance on January 8th?"**

```
Replay events up to 2025-01-08:
  [1] AccountOpened:       balance = 0
  [2] MoneyDeposited:      balance = 5000
  [3] MoneyWithdrawn:      balance = 3500
  (stop - next event is Jan 10th)

Answer: Balance on January 8th was $3,500.00
```

### Example 2: Shopping Cart

```
Event Store: Cart-2024

[1] CartCreated
    { userId: "U-55", sessionId: "sess-abc" }

[2] ItemAdded
    { productId: "PROD-101", name: "Wireless Mouse", price: 29.99, qty: 1 }

[3] ItemAdded
    { productId: "PROD-205", name: "USB-C Cable", price: 12.99, qty: 2 }

[4] ItemQuantityChanged
    { productId: "PROD-205", oldQty: 2, newQty: 3 }

[5] ItemRemoved
    { productId: "PROD-101", reason: "customer_removed" }

[6] CouponApplied
    { code: "SAVE10", discountPercent: 10 }

[7] CartCheckedOut
    { subtotal: 38.97, discount: 3.90, total: 35.07, paymentMethod: "credit_card" }
```

**Replaying events to build cart state**:

```
After [1]:  { items: [], status: "active" }
After [2]:  { items: [{PROD-101, qty:1, $29.99}], status: "active" }
After [3]:  { items: [{PROD-101, qty:1, $29.99}, {PROD-205, qty:2, $12.99}] }
After [4]:  { items: [{PROD-101, qty:1, $29.99}, {PROD-205, qty:3, $12.99}] }
After [5]:  { items: [{PROD-205, qty:3, $12.99}] }
After [6]:  { items: [{PROD-205, qty:3, $12.99}], coupon: "SAVE10" }
After [7]:  { items: [{PROD-205, qty:3}], total: $35.07, status: "checked_out" }
```

**Analytics derived from events**: Without Event Sourcing, you would only know the final cart state. With Event Sourcing, you can analyze:

- Items added then removed (abandoned interest)
- Time between adding items and checking out
- Coupon usage patterns
- Cart abandonment rates (CartCreated without CartCheckedOut)

---

## 6. Benefits and Drawbacks

### Benefits

| Benefit | Description |
|---------|-------------|
| **Complete audit trail** | Every state change is permanently recorded. Perfect for compliance in finance, healthcare, and legal systems |
| **Temporal queries** | Reconstruct the state of any entity at any point in the past by replaying events up to that timestamp |
| **Reliable event publishing** | Events are the state itself -- there is no separate "publish" step. Subscribers read from the event store |
| **Debugging and forensics** | Replay events to reproduce bugs. Understand exactly how the system reached a given state |
| **Natural fit with CQRS** | Events from the event store drive read-model projections, making [CQRS](../data-patterns/cqrs.md) straightforward to implement |
| **Retroactive new features** | Add a new projection or read model and populate it by replaying historical events |
| **Performance (writes)** | Appending to an append-only log is extremely fast -- no read-before-write, no locking of rows |
| **Domain-driven design** | Forces you to think about domain events explicitly, leading to a richer understanding of the domain |

### Drawbacks

| Drawback | Description |
|----------|-------------|
| **Complexity** | Significantly more complex than traditional CRUD. Requires understanding of event stores, projections, snapshots, and eventual consistency |
| **Event schema evolution** | Changing event schemas is difficult because historical events are immutable. Requires upcasting or versioning strategies |
| **Eventual consistency** | Read models are eventually consistent with the event store. Users may not see their own writes immediately |
| **Learning curve** | Unfamiliar to most developers. The "append-only, replay events" mental model is fundamentally different from CRUD |
| **Querying difficulty** | Cannot directly query current state from the event store -- must use projections. Complex queries require building specialized read models |
| **Storage growth** | Event store grows indefinitely. Requires archival and retention strategies for long-lived systems |
| **Replay time** | Without snapshots, reconstructing state for aggregates with many events is slow |
| **Tooling maturity** | Fewer mature tools and frameworks compared to traditional ORM/CRUD approaches |

### When Event Sourcing Pays Off vs When It Doesn't

```
Event Sourcing makes sense when:
├─ Complete audit trail is required (regulatory, compliance)
├─ Temporal queries are a core business requirement
├─ You need reliable event-based integration between services
├─ Domain is complex with rich state transitions
├─ You are already adopting CQRS
└─ Debugging past behavior is critical

Event Sourcing is overkill when:
├─ Simple CRUD with no audit requirements
├─ Team lacks experience with the pattern
├─ Domain is simple (blog, settings page, basic admin)
├─ Strong consistency for reads is mandatory
├─ Rapid prototyping / early-stage startup
└─ No need for historical queries
```

---

## 7. Related Patterns

Event Sourcing connects deeply with several other patterns in this repository:

### CQRS (Command Query Responsibility Segregation)

[CQRS](../data-patterns/cqrs.md) is the most natural companion to Event Sourcing. Since the event store is append-only and not optimized for queries, CQRS provides the read side by projecting events into denormalized, query-optimized read models.

```
Event Sourcing + CQRS
┌───────────────────────────────────────────────────────────┐
│                       WRITE SIDE                          │
│  Command ──▶ Handler ──▶ Aggregate ──▶ Event Store       │
└──────────────────────────────────┬────────────────────────┘
                                   │
                                   │ Events published
                                   │
         ┌─────────────────────────┼─────────────────────┐
         │                         │                     │
         ▼                         ▼                     ▼
   ┌───────────┐            ┌───────────┐         ┌───────────┐
   │  SQL DB   │            │  Search   │         │  Cache    │
   │ (Reports) │            │  Index    │         │ (Redis)   │
   └───────────┘            └───────────┘         └───────────┘
         ▲                         ▲                     ▲
         └─────────────────────────┴─────────────────────┘
                          READ SIDE
```

### Event-Driven Architecture

[Event-Driven Architecture](./event-driven-architecture.md) provides the broader context in which Event Sourcing operates. Event Sourcing is one of the four event-driven patterns described by Martin Fowler, alongside Event Notification, Event-Carried State Transfer, and CQRS.

### Domain Event

Domain Events represent significant occurrences within a bounded context. In Event Sourcing, every state change IS a domain event. The event store is essentially a persistence mechanism for domain events.

### Saga Pattern

The [Saga Pattern](../distributed-transactions/saga.md) coordinates distributed transactions across multiple services. When using Event Sourcing, saga state transitions are naturally modeled as events, and compensating actions produce compensating events.

### Two-Phase Commit

[Two-Phase Commit](../distributed-transactions/two-phase-commit.md) is an alternative approach to achieving atomicity across distributed systems. Event Sourcing eliminates the dual-write problem by making events the single source of truth, removing the need for 2PC in many cases.

### Pattern Relationship Map

```
                        Event Sourcing
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
         CQRS            Event-Driven       Saga Pattern
     (read models        Architecture      (distributed
      from events)       (broader context)  transactions
            │                 │              via events)
            │                 │                 │
            ▼                 ▼                 ▼
      Projections        Domain Events     Compensating
      Snapshots          Event Broker      Events
      Read DBs           Pub/Sub
```

---

## 8. Real-World Usage

### Financial Systems

Event Sourcing originated from accounting principles and remains heavily used in financial systems.

**Banking ledgers**: Every transaction (deposit, withdrawal, transfer, fee) is an event. Account balances are computed from the event stream. Regulators can audit every change. Banks can reconstruct account state at any historical date.

**Trading platforms**: The **LMAX Exchange** is one of the most famous Event Sourcing implementations. It processes 6 million transactions per second using a single-threaded event-sourced architecture. Events are the canonical record of all trades, and the order book state is rebuilt by replaying events.

```
LMAX Architecture (Simplified)
┌──────────┐    ┌──────────────────────┐    ┌──────────────┐
│  Orders  │───▶│  Business Logic     │───▶│  Event       │
│  (Input) │    │  Processor          │    │  Journal     │
└──────────┘    │  (Single Thread)    │    │  (Append-    │
                │  - Match orders     │    │   Only Disk) │
                │  - Update positions │    └──────────────┘
                │  - Risk checks      │           │
                └──────────────────────┘           │
                                                   ▼
                                            ┌──────────────┐
                                            │ Replicate to │
                                            │ other nodes  │
                                            └──────────────┘
```

### Healthcare

Patient medical records benefit enormously from Event Sourcing:

- Every diagnosis, prescription, lab result, and procedure is an event
- Complete, immutable medical history for each patient
- Temporal queries: "What medications was the patient on in March?"
- Regulatory compliance (HIPAA) with tamper-proof audit trail
- Ability to retroactively generate reports from historical data

### Betting Platforms

Sports betting platforms like **Betfair** and **William Hill** use Event Sourcing for:

- Recording every bet placement, settlement, and cash-out as an event
- Audit trail for regulatory compliance (gambling regulations)
- Real-time odds calculation by projecting bet events
- Historical analysis of betting patterns

### E-commerce and Inventory

Large e-commerce platforms use Event Sourcing for:

- Inventory management (every stock change is an event)
- Order lifecycle tracking (created, paid, shipped, delivered, returned)
- Shopping cart behavior analysis
- Price history and change tracking

### Event Store Implementations

| Technology | Type | Language / Platform | Key Features |
|------------|------|---------------------|--------------|
| **EventStoreDB** | Purpose-built event store | Multi-platform | Built by Greg Young; native projections, subscriptions, clustering |
| **Axon Framework** | Framework + Server | Java / JVM | Full CQRS+ES framework; Axon Server for event storage and routing |
| **Marten** | Library on PostgreSQL | .NET / C# | Uses PostgreSQL as event store; integrates with .NET ecosystem |
| **Apache Kafka** | Event streaming platform | Multi-platform | Not a true event store but widely used; requires careful stream design |
| **AWS EventBridge** | Managed event bus | AWS | Serverless event routing; integrates with AWS services |
| **Eventuous** | Library | .NET / C# | Lightweight ES library supporting multiple storage backends |

### When to Use Which Event Store

```
Choose EventStoreDB when:
├─ Event Sourcing is the primary pattern
├─ Need built-in projections and subscriptions
└─ Want a purpose-built solution

Choose Axon Framework when:
├─ Java / Spring Boot ecosystem
├─ Need full CQRS + Event Sourcing framework
└─ Want built-in saga support

Choose Marten when:
├─ .NET ecosystem
├─ Want to use PostgreSQL as event store
└─ Prefer a library over a separate server

Choose Kafka when:
├─ Already have Kafka infrastructure
├─ High-throughput event streaming is primary need
├─ Event Sourcing is secondary to event-driven messaging
└─ Willing to build event-sourcing semantics on top
```

---

## 9. Summary

Event Sourcing is a powerful architectural pattern that fundamentally changes how you think about state management. Instead of storing the current state, you store the complete history of state changes as immutable events. The current state is always derivable by replaying events.

### Key Takeaways

1. **Events are the source of truth** -- the event store is the canonical record; all other representations are derived.

2. **Append-only and immutable** -- events are never updated or deleted, providing a natural audit trail and enabling temporal queries.

3. **Reconstruct state by replay** -- load an aggregate's event stream and apply events in order to build current state. Use snapshots to optimize this for long-lived aggregates.

4. **Solves the dual-write problem** -- since events ARE the state change, there is no separate "update state then publish event" problem. This pairs naturally with [CQRS](../data-patterns/cqrs.md) for read-model projections.

5. **Event schema evolution is hard** -- plan for it from the start using upcasting, versioning, or weak-schema strategies.

6. **Not for every system** -- the added complexity is only justified when you genuinely need audit trails, temporal queries, or reliable event-based integration. Simple CRUD systems should stay simple.

7. **Real-world proven** -- financial systems (LMAX Exchange), healthcare, betting platforms, and large-scale e-commerce systems rely on Event Sourcing for its auditability and reliability.

### Decision Checklist

Before adopting Event Sourcing, verify that:

- [ ] You have a genuine need for a complete audit trail
- [ ] Temporal queries are a business requirement
- [ ] Eventual consistency for reads is acceptable
- [ ] Your team understands the pattern and its implications
- [ ] You have a plan for event schema evolution
- [ ] You have considered snapshot strategy for long-lived aggregates
- [ ] You understand how this fits with your broader architecture (CQRS, event-driven)
- [ ] The complexity is justified by the business requirements

### Further Reading

**Books**:
- "Implementing Domain-Driven Design" by Vaughn Vernon (Event Sourcing chapter)
- "Designing Data-Intensive Applications" by Martin Kleppmann (Chapter 11: Stream Processing)
- "Building Event-Driven Microservices" by Adam Bellemare
- "Versioning in an Event Sourced System" by Greg Young

**Resources**:
- Greg Young's talks on Event Sourcing and CQRS
- EventStoreDB documentation and tutorials
- Martin Fowler: "Event Sourcing" pattern description
- microservices.io: Event Sourcing pattern

---

**Navigation**:
- Previous: [Event-Driven Architecture](./event-driven-architecture.md)
- Next: [CQRS Pattern](../data-patterns/cqrs.md)
- Related: [Saga Pattern](../distributed-transactions/saga.md), [Two-Phase Commit](../distributed-transactions/two-phase-commit.md), [CAP Theorem](../fundamentals/cap-theorem.md)
