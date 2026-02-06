# Saga Pattern

## 1. Introduction

The **Saga pattern** is a design pattern for managing distributed transactions across multiple microservices. In modern distributed systems where each microservice has its own database (database per service pattern), maintaining data consistency across services becomes challenging. The Saga pattern addresses this by breaking a distributed transaction into a sequence of local transactions, where each transaction updates data within a single service and publishes an event or message to trigger the next transaction step.

Unlike traditional ACID transactions that provide immediate consistency, sagas provide eventual consistency and are particularly useful in microservices architectures where services need to maintain autonomy while coordinating complex business processes.

The Saga pattern serves as a practical alternative to distributed transaction protocols like [Two-Phase Commit (2PC)](two-phase-commit.md), especially in environments where:
- Services use different types of databases (SQL and NoSQL)
- High availability is more important than immediate consistency
- Blocking operations are unacceptable
- Services need to remain loosely coupled

## 2. Problem Statement

In a microservices architecture with the database per service pattern, each service owns its database and no other service can access it directly. This design provides excellent service autonomy and loose coupling but creates a significant challenge: **how do you maintain data consistency across multiple services when executing business transactions that span multiple services?**

Traditional ACID transactions don't work across service boundaries because:
- Each service has its own database
- Services are distributed across different processes or machines
- Database transactions are local to a single database

Using distributed transaction protocols like Two-Phase Commit (2PC) is problematic because:
- **Blocking nature**: 2PC is a blocking protocol where services hold locks during coordination, reducing availability
- **Tight coupling**: Services must be available simultaneously for the transaction to succeed
- **NoSQL incompatibility**: Many modern NoSQL databases don't support 2PC
- **Reduced availability**: If any service is down, the entire transaction fails
- **Performance overhead**: The coordination protocol adds significant latency

Consider an e-commerce system where creating an order involves:
1. Verifying customer credit limit (Customer Service)
2. Charging the payment method (Payment Service)
3. Reserving inventory (Inventory Service)
4. Scheduling delivery (Delivery Service)

Without a pattern like Saga, ensuring all these steps either all succeed or all fail becomes extremely complex.

## 3. Solution Overview

The Saga pattern solves the distributed transaction problem by breaking the overall transaction into a sequence of local transactions. Each local transaction updates data within a single service and publishes an event or message to trigger the next local transaction in the saga.

**Key Characteristics:**

1. **Sequence of Local Transactions**: Each step is a local ACID transaction within a single service
2. **Event/Message Publishing**: After each local transaction, the service publishes an event to trigger the next step
3. **Compensating Transactions**: If a step fails, the saga executes compensating transactions to undo the changes made by preceding transactions
4. **Eventual Consistency**: The saga provides eventual consistency across services rather than immediate consistency

**Two Coordination Approaches:**

The Saga pattern can be implemented using two different coordination mechanisms:

1. **Choreography**: Each service listens to events and decides when to act (decentralized)
2. **Orchestration**: A central coordinator tells services what local transactions to execute (centralized)

Both approaches achieve the same goal but with different trade-offs in complexity, coupling, and testing.

## 4. Saga Coordination Approaches

### Choreography-Based Saga

In choreography, there is no central coordinator. Each service involved in the saga listens to events published by other services and decides whether to take action based on those events.

**How it works:**
1. Service A completes a local transaction and publishes an event
2. Service B listens to Service A's event, performs its local transaction, and publishes its own event
3. Service C listens to Service B's event and continues the chain
4. If any service fails, it publishes a failure event triggering compensating transactions in previous services

```
Order Service    Customer Service    Payment Service    Inventory Service
     |                  |                   |                   |
     |-- Order Created ->|                  |                   |
     |                  |-- Credit Reserved ->                  |
     |                  |                  |-- Payment Charged ->
     |<----------------------- Order Approved --------------------|
```

**Characteristics:**
- Decentralized decision-making
- Services communicate through events
- No single point of failure
- More complex to understand and debug
- Good for simple sagas with few steps

### Orchestration-Based Saga

In orchestration, a saga orchestrator (a separate component or service) coordinates all the saga transactions. The orchestrator tells each service what local transaction to execute and listens for completion events.

**How it works:**
1. Orchestrator sends a command to Service A to execute a local transaction
2. Service A executes its transaction and sends a reply to the orchestrator
3. Orchestrator sends a command to Service B
4. Process continues until all steps complete or a failure occurs
5. On failure, orchestrator sends compensating commands to previous services

```
                   Saga Orchestrator
                         |
         +-------+-------+-------+-------+
         |       |       |       |       |
         v       v       v       v       v
      Order  Customer Payment Inventory Delivery
     Service Service Service  Service   Service
         |       |       |       |       |
         +-------+-------+-------+-------+
                         |
                   (Commands & Replies)
```

**Characteristics:**
- Centralized coordination logic
- Orchestrator knows the saga state
- Easier to understand, test, and monitor
- Single point of coordination (not failure if properly designed)
- Better for complex sagas with many steps

## 5. Practical Example: E-Commerce Order Processing

Let's examine a complete example of creating an order in an e-commerce system using the Saga pattern.

### Business Transaction: Create Order

**Services Involved:**
1. **Order Service**: Creates the order
2. **Customer Service**: Verifies and reserves customer credit
3. **Payment Service**: Processes payment
4. **Kitchen Service**: Creates a ticket for food preparation
5. **Accounting Service**: Records the transaction

### Success Flow (Orchestration-Based)

```
Time    Orchestrator    Order       Customer    Payment     Kitchen     Accounting
 |           |            |            |           |           |            |
 v      CREATE ORDER      |            |           |           |            |
 |           |----------->|            |           |           |            |
 v           |         Created         |           |           |            |
 |           |<-----------|            |           |           |            |
 v      RESERVE CREDIT    |            |           |           |            |
 |           |------------------------>|           |           |            |
 v           |                      Reserved       |           |            |
 |           |<------------------------|           |           |            |
 v      CHARGE PAYMENT    |            |           |           |            |
 |           |--------------------------------->   |           |            |
 v           |                                  Charged        |            |
 |           |<------------------------------------|           |            |
 v      CREATE TICKET     |            |           |           |            |
 |           |------------------------------------------------>|            |
 v           |                                              Created          |
 |           |<------------------------------------------------|            |
 v      RECORD TRANSACTION|            |           |           |            |
 |           |------------------------------------------------------------>|
 v           |                                                           Recorded
 |           |<------------------------------------------------------------|
 v      APPROVE ORDER      |            |           |           |            |
 |           |----------->|            |           |           |            |
 v           |         Approved        |           |           |            |
```

### Failure Scenario with Compensating Transactions

What happens if the Payment Service fails after Customer Service has reserved credit?

```
Time    Orchestrator    Order       Customer    Payment
 |           |            |            |           |
 v      CREATE ORDER      |            |           |
 |           |----------->|            |           |
 v           |         Created         |           |
 |           |<-----------|            |           |
 v      RESERVE CREDIT    |            |           |
 |           |------------------------>|           |
 v           |                      Reserved       |
 |           |<------------------------|           |
 v      CHARGE PAYMENT    |            |           |
 |           |--------------------------------->   |
 v           |                                  FAILED
 |           |<------------------------------------|
 v      UNRESERVE CREDIT  |            |           |
 |           |------------------------>|           |
 |           |                      Unreserved     |
 |           |<------------------------|           |
 v      REJECT ORDER      |            |           |
 |           |----------->|            |           |
 v           |         Rejected        |           |
```

**Compensating Transactions:**
1. **Order Created** → Cancel Order
2. **Credit Reserved** → Unreserve Credit
3. **Payment Charged** → Refund Payment
4. **Ticket Created** → Cancel Ticket
5. **Transaction Recorded** → Reverse Transaction

## 6. Compensating Transactions

Compensating transactions are the key mechanism that allows sagas to provide atomicity without distributed locks. They perform a **semantic rollback** rather than a technical rollback.

### Definition

A compensating transaction is a local transaction that undoes the business effect of a previous local transaction in the saga. It doesn't restore the database to its exact previous state (like a database rollback), but rather applies the inverse business operation.

### Examples

| Original Transaction | Compensating Transaction |
|---------------------|-------------------------|
| Reserve Credit | Unreserve Credit |
| Charge Payment | Refund Payment |
| Reserve Inventory | Release Inventory |
| Create Order | Cancel Order |
| Send Email | Send Cancellation Email |
| Lock Account | Unlock Account |

### Key Requirements for Compensating Transactions

**1. Idempotency**

Compensating transactions must be idempotent because they might be executed multiple times due to failures or retries. For example, if "Refund Payment" is called twice with the same refund ID, it should only refund once.

```
// Idempotent refund operation
refundPayment(orderId, amount) {
    if (refundAlreadyProcessed(orderId)) {
        return SUCCESS; // Already refunded, skip
    }
    processRefund(orderId, amount);
    recordRefund(orderId);
    return SUCCESS;
}
```

**2. Semantic Correctness**

The compensating transaction must correctly undo the business effect. For example, if a customer's credit was reserved for $100, the compensating transaction must unreserve exactly $100, not a different amount.

**3. Eventual Success**

Compensating transactions should be designed to eventually succeed even if they initially fail. This often requires retry mechanisms and proper error handling.

### Handling Compensating Transaction Failures

What if a compensating transaction itself fails? This is a critical scenario that requires careful design:

**Strategies:**
1. **Retry with backoff**: Automatically retry the compensating transaction
2. **Manual intervention**: Alert operations team for manual compensation
3. **Timeout to manual**: After N retries, escalate to manual process
4. **Alternative compensation**: Try an alternative compensating action

**Important Note**: Not all transactions are compensatable. Some operations (like sending an email or publishing a message to external systems) cannot be truly undone. In these cases, you need to either:
- Make them the last step in the saga
- Use semantic compensation (send a cancellation email)
- Design the system to handle duplicate operations

## 7. Choreography vs Orchestration Deep Dive

### Detailed Comparison

| Aspect | Choreography | Orchestration |
|--------|-------------|---------------|
| **Coordination** | Decentralized (services coordinate via events) | Centralized (orchestrator coordinates all services) |
| **Service Dependencies** | Services depend on events from other services | Services depend only on orchestrator commands |
| **Saga Logic Location** | Distributed across all participating services | Centralized in the orchestrator |
| **Coupling** | Loosely coupled between services | Services coupled to orchestrator interface |
| **Complexity** | Complex for sagas with many steps | Simpler for complex sagas |
| **Testing** | Harder to test (need to simulate multiple services) | Easier to test (test orchestrator logic) |
| **Debugging** | Difficult to track saga state | Easy to track saga state (in orchestrator) |
| **Single Point of Failure** | No central point | Orchestrator is coordination point |
| **Scalability** | Each service scales independently | Orchestrator must handle all saga coordination |
| **Understanding** | Hard to understand complete flow | Easy to understand flow (defined in orchestrator) |
| **Evolution** | Hard to change (must update multiple services) | Easier to change (update orchestrator) |
| **Monitoring** | Must monitor events across services | Monitor orchestrator state |

### When to Use Choreography

✓ Use choreography when:
- The saga involves few services (2-4)
- The business logic is simple and stable
- Services already communicate via events
- You want maximum service autonomy
- The organization has strong event-driven culture

Example: A simple saga with Order Service → Payment Service → Notification Service

### When to Use Orchestration

✓ Use orchestration when:
- The saga involves many services (5+)
- Complex business logic with conditional flows
- Saga logic changes frequently
- Need centralized monitoring and testing
- Team is new to saga pattern

Example: A complex order fulfillment process involving 10+ services with conditional logic

### Hybrid Approach

In practice, many systems use a hybrid approach:
- Use choreography for simple, stable sagas
- Use orchestration for complex sagas
- Different sagas in the same system can use different coordination styles

## 8. System Design Trade-offs

### Advantages ✓

**1. High Availability**
- No distributed locks mean services remain available
- Failure of one service doesn't block others
- Better fault tolerance in distributed systems

**2. Loose Coupling**
- Services maintain autonomy
- No need for all services to be available simultaneously
- Services can be developed and deployed independently

**3. NoSQL Database Support**
- Works with databases that don't support 2PC
- Can mix different database types (SQL, NoSQL, key-value stores)
- More flexibility in technology choices

**4. Scalability**
- Each service can scale independently
- No central lock manager to become bottleneck
- Better performance under high load

**5. Enables Microservices Architecture**
- Makes complex business transactions possible across services
- Maintains database per service pattern benefits
- Supports service independence

### Disadvantages ✗

**1. Eventual Consistency Only**
- No immediate consistency guarantee
- Application must handle intermediate states
- Not suitable for use cases requiring strict consistency

**2. Increased Complexity**
- Requires designing compensating transactions
- More complex error handling
- Harder to reason about system state

**3. Lack of Isolation**
- Concurrent sagas can interfere with each other
- Dirty reads and lost updates are possible
- Requires additional countermeasures for isolation

**4. Testing Complexity**
- Must test success paths, failure paths, and compensation paths
- Harder to reproduce failure scenarios
- Integration testing requires multiple services

**5. Debugging Challenges**
- Saga state is distributed (choreography) or centralized (orchestration)
- Must track events/messages across services
- Requires good observability and logging

**6. Operational Complexity**
- Requires monitoring saga state
- Need mechanisms for handling stuck sagas
- Manual intervention may be needed for compensation failures

## 9. Saga vs Two-Phase Commit Comparison

Understanding when to use Saga versus Two-Phase Commit (2PC) is crucial for system design.

### Side-by-Side Comparison

| Aspect | Saga Pattern | Two-Phase Commit (2PC) |
|--------|-------------|------------------------|
| **Consistency Model** | Eventual consistency (BASE) | Immediate consistency (ACID) |
| **Coordination** | Event-driven or orchestrated | Prepare → Vote → Commit phases |
| **Blocking** | Non-blocking (services don't wait) | Blocking (services hold locks) |
| **Availability** | High (services stay available) | Lower (failure blocks all) |
| **Isolation** | No isolation between sagas | Full isolation with locks |
| **Rollback Mechanism** | Compensating transactions (semantic) | Database rollback (technical) |
| **Database Support** | Works with any database | Requires 2PC support (XA transactions) |
| **Complexity** | Higher application complexity | Lower application complexity |
| **Performance** | Better (no locking overhead) | Worse (locking and coordination overhead) |
| **Use Case** | Microservices, distributed systems | Monoliths, tightly coupled systems |
| **NoSQL Support** | Yes | No (most NoSQL databases don't support 2PC) |
| **Single Point of Failure** | No (choreography) or orchestrator (orchestration) | Coordinator (but can be replicated) |
| **Scalability** | High | Limited by coordinator and locks |

### Visual Comparison

```
Two-Phase Commit (2PC)              Saga Pattern
     Coordinator                    Event Bus / Orchestrator
          |                                  |
    +-----+-----+                      +-----+-----+
    |     |     |                      |     |     |
    v     v     v                      v     v     v
   Svc1 Svc2 Svc3                    Svc1 Svc2 Svc3
          |                                  |
    [PREPARE]                          [LOCAL TXN]
    [VOTE YES/NO]                      [PUBLISH EVENT]
    [LOCKS HELD]                       [NO LOCKS]
    [COMMIT/ABORT]                     [COMPENSATE IF NEEDED]
          |                                  |
    Strong Consistency                 Eventual Consistency
    Low Availability                   High Availability
```

### Decision Matrix: When to Use What

**Use Two-Phase Commit when:**
- ✓ You need strong ACID guarantees
- ✓ Immediate consistency is required
- ✓ Working within a monolithic application
- ✓ All databases support XA transactions
- ✓ Lower throughput is acceptable
- ✓ Failures are rare

**Use Saga Pattern when:**
- ✓ Building microservices architecture
- ✓ Need high availability and scalability
- ✓ Using NoSQL databases or mixed database types
- ✓ Eventual consistency is acceptable
- ✓ Services must remain autonomous
- ✓ Long-running transactions

### Real-World Analogy

**Two-Phase Commit** is like a synchronized group dance where everyone must start, move, and stop at exactly the same time. If one dancer falls, the entire performance stops.

**Saga Pattern** is like a relay race where each runner completes their leg independently and passes the baton. If a runner drops the baton, they pick it up and the race continues (or others run backward to compensate).

## 10. Failure Scenarios & Recovery Strategies

Handling failures is central to the Saga pattern. Let's explore common failure scenarios and recovery strategies.

### Scenario 1: Service Failure Mid-Saga

**Problem**: A service fails while executing its local transaction.

```
Order Service → Customer Service → [Payment Service FAILS] → Kitchen Service
```

**Recovery Strategy:**

1. **Detect failure**: Timeout or explicit error response
2. **Trigger compensation**: Orchestrator (or previous service in choreography) initiates compensating transactions
3. **Compensate in reverse order**:
   - Unreserve credit (Customer Service)
   - Cancel order (Order Service)

**Implementation Considerations:**
- Set appropriate timeouts for each step
- Distinguish between transient (retry) and permanent (compensate) failures
- Implement exponential backoff for retries
- Log all failures for debugging

### Scenario 2: Network Partition

**Problem**: Network partition prevents communication between services.

```
Order Service → | NETWORK PARTITION | → Customer Service
```

**Recovery Strategy:**

1. **Timeout detection**: Service doesn't receive response within timeout
2. **Retry with idempotency**: Retry the operation (service must handle duplicate requests)
3. **Eventual compensation**: If retries exhausted, trigger compensating transactions
4. **Reconciliation**: After partition heals, reconcile state

**Implementation Considerations:**
- Use idempotency keys for all operations
- Implement at-least-once delivery semantics
- Use message queues for reliable delivery
- Build reconciliation processes for state verification

### Scenario 3: Compensating Transaction Failure

**Problem**: A compensating transaction fails.

```
Saga Fails → Compensate Step 3 → Compensate Step 2 → [Compensate Step 1 FAILS]
```

**Recovery Strategy:**

1. **Retry with exponential backoff**: Automatically retry the compensating transaction
2. **Circuit breaker**: Stop retrying after threshold, mark saga for manual intervention
3. **Dead letter queue**: Move failed compensation to DLQ for manual processing
4. **Alert operations team**: Human intervention may be needed
5. **Eventual resolution**: Keep retrying until manually resolved

**Implementation:**
```
compensate(saga, step) {
    maxRetries = 5;
    retryCount = 0;

    while (retryCount < maxRetries) {
        try {
            executeCompensatingTransaction(step);
            return SUCCESS;
        } catch (error) {
            retryCount++;
            if (retryCount >= maxRetries) {
                moveToDLQ(saga, step);
                alertOperations(saga, step, error);
                return MANUAL_INTERVENTION_REQUIRED;
            }
            sleep(exponentialBackoff(retryCount));
        }
    }
}
```

### Scenario 4: Timeout Handling

**Problem**: Service doesn't respond within expected time.

**Recovery Strategy:**

1. **Short timeout**: Detect issues quickly
2. **Retry logic**: Retry the operation with idempotency
3. **Fallback to compensation**: If retries exhausted, compensate
4. **Monitoring**: Track timeout rates to identify systemic issues

**Timeout Configuration:**
```
Service Operation Timeouts:
- Database operations: 5 seconds
- External API calls: 10 seconds
- Total saga step timeout: 30 seconds
- Total saga timeout: 5 minutes
```

### Scenario 5: Partial Failure

**Problem**: Operation partially completes (e.g., payment charged but database update fails).

**Recovery Strategy:**

1. **Transactional outbox pattern**: Ensure atomicity of local transaction and event publishing
2. **Polling publisher**: Periodically check for unpublished events
3. **Idempotent compensation**: Design compensations to handle partial states

### General Recovery Principles

**1. Idempotency Everywhere**
- All operations (forward and compensating) must be idempotent
- Use unique transaction IDs to detect duplicates

**2. Observability**
- Log all saga steps with correlation IDs
- Monitor saga completion rates
- Alert on stuck or failing sagas

**3. Timeouts and Retries**
- Set appropriate timeouts for each step
- Implement retry with exponential backoff
- Distinguish between retryable and non-retryable errors

**4. Manual Intervention**
- Build tools for operations team to inspect saga state
- Provide mechanisms to manually complete or compensate sagas
- Maintain audit logs for compliance

## 11. Isolation Challenges

One of the biggest challenges with the Saga pattern is the lack of ACID isolation. Unlike database transactions that provide full isolation, sagas execute a series of local transactions that are individually isolated but not collectively isolated.

### The Problem: Lack of Isolation

Consider two concurrent sagas operating on the same customer account:

```
Time    Saga A (Create Order)         Saga B (Update Credit Limit)
 |      Customer credit: $1000        Customer credit: $1000
 v
 |      Reserve $800
 |      (Balance: $200)
 v                                     Read balance: $200
 |                                     Increase limit: $200 → $500
 |                                     (Balance: $500)
 v      Payment fails
 |      Unreserve $800
 |      (Balance: $1000)
 v                                     Write new limit: $500
 |                                     (Should be $1300, not $500)
 |
Result: Lost update! Saga B's update was based on stale data.
```

### Data Anomalies in Sagas

**1. Lost Updates**

Two sagas read the same data, make changes, and both write back, with one overwriting the other.

**2. Dirty Reads**

Saga A reads data that Saga B has modified but might later compensate (rollback), leading Saga A to make decisions on data that will be undone.

```
Time    Saga A                        Saga B
 |                                    Reserve inventory item X
 v      Read: item X reserved
 |      Skip item X (already reserved)
 v                                    Payment fails
 |                                    Unreserve item X
 v      (Item X was actually available)
```

**3. Non-repeatable Reads**

Saga A reads the same data twice during its execution and gets different values because Saga B modified it in between.

### Countermeasures for Isolation Issues

**1. Semantic Lock**

A saga's compensatable transaction places a flag (semantic lock) on any record it creates or updates. The flag indicates that the record might change if the saga compensates.

```
Order {
    id: "123",
    status: "PENDING",  // Semantic lock
    amount: 100
}

// Other sagas check status before acting
if (order.status == "PENDING") {
    // Order might be compensated, handle carefully
}
```

**2. Commutative Updates**

Design updates to be commutative (order-independent) so concurrent sagas don't interfere.

**Non-commutative (problematic):**
```
balance = balance - 100  // Order matters
```

**Commutative (safe):**
```
transactions.add(new Debit(100))  // Can be applied in any order
```

**3. Pessimistic View**

Reorder saga steps to minimize risk of dirty reads by having potentially compensatable steps execute last.

**Problematic order:**
```
1. Reserve inventory (might be compensated)
2. Validate customer (uses inventory status)
3. Process payment
```

**Better order:**
```
1. Validate customer (no compensation risk)
2. Process payment (low compensation risk)
3. Reserve inventory (might be compensated, but nothing depends on it)
```

**4. Reread Value (Optimistic Locking)**

Before updating, reread the value and verify it hasn't changed. If changed, abort or retry.

```
updateCustomerCredit(customerId, delta) {
    initialCredit = readCredit(customerId);
    newCredit = initialCredit + delta;

    success = updateWithVersionCheck(
        customerId,
        newCredit,
        expectedVersion: initialCredit.version
    );

    if (!success) {
        // Another saga modified the credit, retry or abort
        return CONFLICT;
    }
}
```

**5. Version File**

Use version numbers or timestamps to detect concurrent modifications.

```
Customer {
    id: "456",
    creditLimit: 1000,
    version: 5  // Incremented on each update
}

// Update only if version matches
UPDATE customers
SET creditLimit = 1500, version = 6
WHERE id = '456' AND version = 5
```

**6. By Value**

Use specific values rather than relative changes to avoid lost updates.

**Risky:**
```
incrementBalance(amount)  // Multiple calls can interfere
```

**Safer:**
```
setBalance(newValue)  // Explicit value, less ambiguity
```

### Choosing Countermeasures

The choice depends on your requirements:

| Requirement | Recommended Countermeasure |
|------------|---------------------------|
| Prevent dirty reads | Semantic lock |
| Prevent lost updates | Reread value, version file |
| High concurrency | Commutative updates |
| Simple implementation | Pessimistic view |
| Strong consistency needs | Consider 2PC instead of Saga |

## 12. Real-World Applications

Understanding where the Saga pattern is used (and where it's not) helps guide design decisions.

### Where Saga Pattern is Used ✓

**1. E-Commerce Platforms**

**Use Case**: Order processing across multiple services
- Order Service: Create order
- Payment Service: Process payment
- Inventory Service: Reserve items
- Shipping Service: Schedule delivery
- Notification Service: Send confirmations

**Why Saga**: High availability is critical, eventual consistency is acceptable for order processing.

**Examples**: Amazon, eBay, Shopify

**2. Payment Systems**

**Use Case**: Multi-party payment processing
- User account debit
- Merchant account credit
- Fee calculation
- Transaction recording
- Notification

**Why Saga**: Need to handle failures gracefully, compensate with refunds, maintain audit trails.

**Examples**: PayPal, Stripe (internal processing)

**3. Travel Booking Systems**

**Use Case**: Book complete trip (flight + hotel + car rental)
- Reserve flight
- Reserve hotel
- Reserve car
- Process payment
- Send confirmation

**Why Saga**: Must be able to cancel reservations if any part fails, long-running transactions, high availability needs.

**Examples**: Expedia, Booking.com

**4. Food Delivery Platforms**

**Use Case**: Order fulfillment
- Validate order
- Charge customer
- Notify restaurant
- Assign driver
- Track delivery

**Why Saga**: Real-time coordination across multiple services, need to handle cancellations, high availability requirements.

**Examples**: Uber Eats, DoorDash

**5. Financial Trading Platforms**

**Use Case**: Non-critical trading workflows (not core trade execution)
- Portfolio rebalancing
- Tax lot optimization
- Reporting workflows

**Why Saga**: Long-running workflows with multiple steps, eventual consistency acceptable for these specific use cases.

### Where Saga Pattern is NOT Used ✗

**1. Core Banking Transactions**

**Why Not**: Require strong ACID guarantees, immediate consistency critical, cannot accept eventual consistency.

**Use Instead**: [Two-Phase Commit](two-phase-commit.md), distributed databases with strong consistency.

**2. Real-Time Trading Execution**

**Why Not**: Millisecond latency requirements, need immediate consistency, cannot risk dirty reads.

**Use Instead**: In-memory databases, single-database transactions, specialized trading systems.

**3. Monolithic Applications**

**Why Not**: All data in single database, can use local ACID transactions, saga adds unnecessary complexity.

**Use Instead**: Database transactions (BEGIN/COMMIT/ROLLBACK).

**4. Inventory Management with Strong Consistency Requirements**

**Why Not**: Cannot risk overselling due to eventual consistency, need immediate inventory updates.

**Use Instead**: Single source of truth with ACID transactions, distributed locks if needed across services.

**5. User Authentication/Authorization**

**Why Not**: Security-critical operations requiring immediate consistency, cannot accept eventual consistency for access control.

**Use Instead**: Centralized auth service with strong consistency.

### Industry Adoption

**Technology Companies Using Sagas:**
- **Microsoft**: Azure Durable Functions for saga orchestration
- **Netflix**: Conductor workflow engine for orchestration-based sagas
- **Uber**: Cadence workflow engine for long-running sagas
- **AWS**: Step Functions for saga orchestration
- **Airbnb**: Internal orchestration frameworks

**Frameworks and Tools:**
- Axon Framework (Java)
- NServiceBus (C#)
- MassTransit (.NET)
- Eventuate Tram Saga Framework (Java)
- Temporal (workflow engine supporting sagas)

## 13. Related Patterns

The Saga pattern doesn't exist in isolation. It's part of a broader ecosystem of distributed system patterns.

### Database per Service (Prerequisite)

**Relationship**: The Saga pattern is a solution to the distributed transaction problem created by the database per service pattern.

**Description**: Each microservice has its own database that no other service can access directly.

**Why Related**: Without database per service, you could use local database transactions. Saga becomes necessary when services have separate databases.

### Event Sourcing (Implementation Technique)

**Relationship**: Event Sourcing can be used to implement the Saga pattern.

**Description**: Store all changes to application state as a sequence of events rather than storing just the current state.

**How They Work Together**:
- Saga steps publish domain events
- Event store provides audit trail of saga execution
- Events enable choreography-based sagas
- Event replay helps with debugging and recovery

**Example**: When Order Service creates an order, it publishes an "OrderCreated" event that triggers the next saga step.

### [CQRS](../data-patterns/cqrs.md) (Often Combined)

**Relationship**: [CQRS (Command Query Responsibility Segregation)](../data-patterns/cqrs.md) is often used alongside sagas.

**Description**: Separate read and write models for better scalability and performance.

**How They Work Together**:
- Saga commands update write model
- Saga events update read model
- Query model provides eventual consistent view
- Commands and events align with saga steps

**Example**: Order saga updates order write model, and events populate various read models (order history, customer view, analytics).

### Transactional Outbox (Reliable Event Publishing)

**Relationship**: Solves the problem of reliably publishing events as part of saga steps.

**Description**: Store events in an outbox table in the same database transaction as the business entity, then publish events asynchronously.

**Why Critical for Sagas**: Ensures atomicity of local transaction and event publishing, preventing saga from getting stuck due to unpublished events.

**Example**:
```
BEGIN TRANSACTION
    UPDATE orders SET status = 'APPROVED'
    INSERT INTO outbox (event_type, payload) VALUES ('OrderApproved', {...})
COMMIT

// Separate process polls outbox and publishes events
```

### [Two-Phase Commit](two-phase-commit.md) (Alternative Pattern)

**Relationship**: Saga is an alternative to 2PC for distributed transactions.

**Trade-off**: 2PC provides strong consistency but blocks; Saga provides eventual consistency without blocking.

**Decision Factor**: Use 2PC when you need strong consistency and can tolerate blocking; use Saga when you need high availability and can tolerate eventual consistency.

**See**: Section 9 for detailed comparison.

### [Circuit Breaker](../resilience/circuit-breaker.md) (Failure Handling)

**Relationship**: Used within saga implementations to handle service failures gracefully.

**Description**: Prevent cascading failures by stopping requests to failing services.

**How They Work Together**:
- Saga step calls service through circuit breaker
- Circuit breaker detects repeated failures
- Circuit opens, saga triggers compensation instead of retrying
- Prevents saga from waiting indefinitely

### API Composition (Query Pattern)

**Relationship**: Used to query data across services in saga-based systems.

**Description**: Implement queries by invoking multiple services and combining results.

**How They Work Together**:
- Saga maintains data consistency across services
- API Composition queries that data
- Together they provide complete read/write solution for distributed data

## 14. Design Considerations

### When to Use Saga Pattern

✓ **Use Saga when:**

1. **Microservices Architecture**
   - You have multiple services with separate databases
   - Services need to maintain autonomy
   - Cross-service transactions are required

2. **High Availability Requirements**
   - System must remain available even during partial failures
   - Blocking operations are unacceptable
   - Need to handle network partitions gracefully

3. **Heterogeneous Databases**
   - Using mix of SQL and NoSQL databases
   - Databases don't support distributed transactions (2PC)
   - Different services use different database technologies

4. **Long-Running Transactions**
   - Business processes span minutes or hours
   - Cannot hold database locks for extended periods
   - Need to maintain system responsiveness

5. **Acceptable Eventual Consistency**
   - Business logic can tolerate temporary inconsistencies
   - Final consistency is what matters
   - Intermediate states are acceptable to users

### When NOT to Use Saga Pattern

✗ **Avoid Saga when:**

1. **Strong Consistency Required**
   - Immediate consistency is critical
   - Business cannot tolerate any inconsistency window
   - Examples: financial transactions, inventory with no overselling tolerance

2. **Single Database System**
   - All data in one database
   - Can use local ACID transactions
   - Saga adds unnecessary complexity

3. **Simple Read-Only Operations**
   - No data modifications
   - No transaction coordination needed
   - Simple queries across services

4. **Real-Time Low-Latency Requirements**
   - Sub-millisecond response times required
   - Cannot afford saga coordination overhead
   - Examples: high-frequency trading

5. **Team Lacks Distributed Systems Experience**
   - Team new to microservices
   - Lack of experience with eventual consistency
   - Consider starting with simpler patterns

### Orchestration vs Choreography Decision Factors

**Choose Orchestration when:**
- Saga involves many services (5+)
- Complex conditional logic (if-then-else flows)
- Saga logic changes frequently
- Need centralized monitoring
- Team prefers explicit workflow definitions

**Choose Choreography when:**
- Simple saga with few services (2-4)
- Stable business logic
- Organization has strong event-driven culture
- Want maximum service autonomy
- Services already communicate via events

**Use Hybrid when:**
- Large system with multiple sagas
- Some sagas simple (choreography), others complex (orchestration)
- Want flexibility to choose per saga

### Testing Strategies

**1. Unit Testing**
- Test individual saga steps in isolation
- Mock dependencies
- Test both success and failure paths

**2. Integration Testing**
- Test complete saga flow
- Use test doubles for external services
- Verify compensating transactions execute correctly

**3. Chaos Testing**
- Randomly inject failures during saga execution
- Verify saga handles failures gracefully
- Test partial failure scenarios

**4. End-to-End Testing**
- Test saga in production-like environment
- Use real services (not mocks)
- Verify eventual consistency is achieved

**Testing Checklist:**
- [ ] All success paths tested
- [ ] All failure points tested with compensation
- [ ] Concurrent saga execution tested
- [ ] Idempotency verified for all operations
- [ ] Timeout handling tested
- [ ] Compensating transaction failures tested

### Monitoring and Observability

**Key Metrics to Track:**

1. **Saga Completion Rate**: Percentage of sagas that complete successfully
2. **Saga Duration**: Time from start to completion
3. **Step Failure Rate**: Failure rate per saga step
4. **Compensation Rate**: How often compensations are triggered
5. **Stuck Sagas**: Sagas that haven't progressed in expected time

**Logging Requirements:**
- Use correlation IDs for all saga operations
- Log all saga state transitions
- Log all compensating transactions
- Include timestamps for timeout detection

**Alerting:**
- Alert on high compensation rate
- Alert on stuck sagas
- Alert on compensating transaction failures

## 15. Modern System Design Implications

### Cloud-Native Architecture Fit

The Saga pattern aligns well with cloud-native principles:

**1. Designed for Failure**
- Cloud services can fail unpredictably
- Saga's compensating transactions handle failures gracefully
- Non-blocking nature improves resilience

**2. Independently Deployable**
- Each service in saga can be deployed independently
- Saga coordination doesn't require synchronized deployments
- Supports continuous delivery

**3. Scalability**
- Services scale independently based on load
- No global locks to become bottlenecks
- Choreography enables horizontal scaling

### Kubernetes and Service Mesh Integration

**Kubernetes Benefits:**
- **Health Checks**: Kubernetes health checks help detect saga step failures quickly
- **Service Discovery**: Dynamic service discovery enables saga services to find each other
- **Resource Management**: Each saga service can have resource limits

**Service Mesh (Istio, Linkerd):**
- **Retries**: Automatic retries for transient failures
- **Timeouts**: Consistent timeout policies across saga steps
- **Circuit Breaking**: Prevent cascading failures
- **Observability**: Distributed tracing across saga steps

**Example Istio Configuration:**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - timeout: 10s
    retries:
      attempts: 3
      perTryTimeout: 3s
    route:
    - destination:
        host: payment-service
```

### Event Streaming Platforms

Modern saga implementations often use event streaming platforms:

**Apache Kafka:**
- Durable event log for choreography-based sagas
- Event replay for debugging and recovery
- Partitioning for scalability
- Compacted topics for saga state

**AWS EventBridge:**
- Managed event bus for saga events
- Schema registry for event validation
- Built-in retry and DLQ
- Integration with AWS services

**Benefits:**
- Reliable event delivery (at-least-once)
- Event replay for testing and debugging
- Audit trail of saga execution
- Decouples saga services

### Observability Requirements

Modern saga implementations require comprehensive observability:

**Distributed Tracing (Jaeger, Zipkin):**
```
Trace: Create Order Saga
  Span: Create Order (Order Service) - 50ms
  Span: Reserve Credit (Customer Service) - 30ms
  Span: Charge Payment (Payment Service) - 200ms
  Span: Create Ticket (Kitchen Service) - 40ms
  Span: Approve Order (Order Service) - 20ms
Total: 340ms
```

**Benefits:**
- Visualize complete saga flow
- Identify performance bottlenecks
- Debug failures with full context
- Calculate saga duration

**Metrics (Prometheus, CloudWatch):**
- saga_duration_seconds (histogram)
- saga_completion_rate (gauge)
- saga_step_failures_total (counter)
- saga_compensations_total (counter)

**Logging (ELK, CloudWatch Logs):**
- Structured logging with correlation IDs
- Centralized log aggregation
- Search and filter by saga ID
- Alert on error patterns

### Serverless and Saga Pattern

Serverless platforms offer managed saga orchestration:

**AWS Step Functions:**
- Visual workflow designer
- Automatic retry and error handling
- Built-in compensation logic
- Integration with Lambda, ECS, SNS, SQS

**Azure Durable Functions:**
- Code-based workflow definition
- Automatic state management
- Built-in error handling
- C# and JavaScript support

**Benefits:**
- No infrastructure management
- Pay-per-use pricing
- Automatic scaling
- Built-in durability

**Example AWS Step Functions Saga:**
```json
{
  "StartAt": "CreateOrder",
  "States": {
    "CreateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:CreateOrder",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "CancelOrder"
      }],
      "Next": "ReserveCredit"
    },
    "ReserveCredit": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:ReserveCredit",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "CancelOrder"
      }],
      "Next": "ChargePayment"
    },
    "CancelOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:CancelOrder",
      "End": true
    }
  }
}
```

## 16. Key Takeaways

1. **Saga enables distributed transactions without distributed locks**: By breaking transactions into local steps with compensating transactions, sagas provide atomicity across services without the blocking nature of 2PC.

2. **Choose orchestration for complex sagas, choreography for simple ones**: Orchestration provides better visibility and control for complex multi-step sagas, while choreography offers better decoupling for simple sagas.

3. **Eventual consistency is the trade-off**: Sagas provide high availability and scalability but only guarantee eventual consistency, not immediate consistency. This makes them unsuitable for use cases requiring strong consistency.

4. **Design compensating transactions carefully**: Compensating transactions must be idempotent, semantically correct, and handle their own failures. Not all operations are compensatable.

5. **Lack of isolation requires countermeasures**: Since sagas don't provide ACID isolation, use techniques like semantic locks, optimistic locking, and commutative updates to prevent data anomalies from concurrent sagas.

**Decision Criteria Summary:**

| Factor | Saga | Two-Phase Commit |
|--------|------|------------------|
| Consistency | Eventual | Immediate |
| Availability | High | Lower |
| Scalability | High | Limited |
| Database Types | Any | 2PC-capable only |
| Complexity | Higher | Lower |
| Use Case | Microservices | Monoliths, strong consistency needs |

## 17. Related Topics to Explore

To deepen your understanding of distributed transactions and microservices patterns, explore these related topics:

### Within This Repository

- **[Two-Phase Commit](two-phase-commit.md)**: Understanding the alternative distributed transaction pattern and when to use 2PC vs Saga

### External Patterns to Study

- **Database per Service**: The microservices pattern that necessitates sagas
- **Event Sourcing**: Store application state as sequence of events; often used with sagas
- **[CQRS (Command Query Responsibility Segregation)](../data-patterns/cqrs.md)**: Separate read/write models, commonly combined with sagas
- **Transactional Outbox**: Reliably publish events as part of database transactions
- **[Event-Driven Architecture](../event-driven/event-driven-architecture.md)**: Broader architectural style that enables choreography-based sagas
- **[Circuit Breaker](../resilience/circuit-breaker.md)**: Failure handling pattern used within saga implementations
- **[Microservices Architecture](../architecture/microservices.md)**: The architectural style that necessitates saga patterns
- **Distributed Tracing**: Essential for observability in saga-based systems

### Advanced Topics

- **Long-Running Transactions**: Theory behind sagas and business process management
- **Eventual Consistency Patterns**: Techniques for working with eventual consistency
- **Compensating Transaction Design**: Deep dive into designing effective compensations
- **Saga Isolation Strategies**: Advanced techniques for handling concurrent sagas
- **Workflow Engines**: Tools like Temporal, Cadence, Conductor for implementing sagas

### Further Reading

- Chris Richardson's microservices.io (comprehensive saga documentation)
- Martin Fowler's blog on enterprise patterns
- "Designing Data-Intensive Applications" by Martin Kleppmann (distributed transactions chapter)
- Original Saga paper: "Sagas" by Hector Garcia-Molina and Kenneth Salem (1987)

---

*This documentation is part of the System Design Notebook. For questions, suggestions, or contributions, please refer to the main repository README.*
