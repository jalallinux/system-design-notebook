# Two-Phase Commit (2PC) - A System Design Perspective

## Introduction

Two-Phase Commit (2PC) is a distributed algorithm used to ensure atomicity in distributed transactions across multiple nodes or databases. It guarantees that either all participants in a distributed transaction commit their changes, or all of them abort, maintaining the ACID properties across distributed systems.

## Problem Statement

In distributed systems, we often need to perform operations that span multiple databases or services. The challenge is:

- **Distributed Atomicity**: How do we ensure that a transaction either completes successfully on all nodes or fails on all nodes?
- **Partial Failures**: What happens if some nodes succeed and others fail?
- **Consistency**: How do we prevent scenarios where half the data is committed and half is rolled back?

Without coordination, you could end up in an inconsistent state where:
- Money is debited from one bank account but never credited to another
- An order is created but inventory is never decremented
- User data is updated in one system but not in related systems

## Solution Overview

Two-Phase Commit solves this problem by introducing a **coordinator** that orchestrates the transaction across multiple **participants** (nodes/databases) in two distinct phases:

1. **Prepare Phase** (Voting Phase): Ask all participants if they can commit
2. **Commit Phase** (Decision Phase): Tell all participants to commit or abort based on votes

The key insight: **No participant commits until ALL participants agree they can commit**.

## Detailed Phase Breakdown

### Phase 1: Prepare (Voting Phase)

```
Coordinator                          Participants
    |                                     |
    |--- PREPARE (Transaction) --------->| Node 1
    |                                     |
    |--- PREPARE (Transaction) --------->| Node 2
    |                                     |
    |--- PREPARE (Transaction) --------->| Node 3
    |                                     |
    |<-------- YES (Ready) --------------|
    |<-------- YES (Ready) --------------|
    |<-------- YES (Ready) --------------|
    |                                     |
```

**What happens in this phase:**

1. **Coordinator sends PREPARE request** to all participants
   - Includes the transaction details and operations to perform

2. **Each participant:**
   - Attempts to perform the transaction locally
   - Writes changes to a temporary area (NOT committed)
   - Acquires all necessary locks
   - Writes a PREPARE record to its Write-Ahead Log (WAL)
   - Responds with either:
     - **YES** - "I can commit if everyone else can"
     - **NO** - "I cannot commit this transaction"

3. **Coordinator collects all votes**
   - Waits for responses from ALL participants
   - Records the decision in its WAL

### Phase 2: Commit/Abort (Decision Phase)

**Scenario A: All participants voted YES**

```
Coordinator                          Participants
    |                                     |
    |--- COMMIT ----------------------->| Node 1
    |                                     |
    |--- COMMIT ----------------------->| Node 2
    |                                     |
    |--- COMMIT ----------------------->| Node 3
    |                                     |
    |<-------- ACK --------------------|
    |<-------- ACK --------------------|
    |<-------- ACK --------------------|
    |                                     |
```

**Scenario B: Any participant voted NO (or timeout)**

```
Coordinator                          Participants
    |                                     |
    |--- ABORT ------------------------>| Node 1
    |                                     |
    |--- ABORT ------------------------>| Node 2
    |                                     |
    |--- ABORT ------------------------>| Node 3
    |                                     |
    |<-------- ACK --------------------|
    |<-------- ACK --------------------|
    |<-------- ACK --------------------|
    |                                     |
```

**What happens in this phase:**

1. **Coordinator makes a decision:**
   - If ALL votes are YES → Send COMMIT to all participants
   - If ANY vote is NO or timeout → Send ABORT to all participants
   - Writes decision to WAL before sending

2. **Each participant:**
   - Receives COMMIT or ABORT message
   - Executes the command (makes changes permanent or discards them)
   - Releases all locks
   - Writes COMMIT/ABORT record to WAL
   - Sends ACK back to coordinator

3. **Coordinator completes:**
   - Waits for all ACKs
   - Writes completion record to WAL
   - Transaction is complete

## Practical Example: Bank Transfer

**Scenario:** Transfer $100 from Account A (Bank DB 1) to Account B (Bank DB 2)

### Step-by-Step Flow

```
1. Client requests: Transfer $100 from A to B

2. PHASE 1 - PREPARE:

   Coordinator → DB1: "Can you debit $100 from Account A?"
   DB1 actions:
     - Check balance (A has $500) ✓
     - Lock Account A
     - Write "PREPARE: Debit A $100" to WAL
     - Set aside the change (don't commit yet)
   DB1 → Coordinator: "YES, ready to commit"

   Coordinator → DB2: "Can you credit $100 to Account B?"
   DB2 actions:
     - Check Account B exists ✓
     - Lock Account B
     - Write "PREPARE: Credit B $100" to WAL
     - Set aside the change (don't commit yet)
   DB2 → Coordinator: "YES, ready to commit"

3. PHASE 2 - COMMIT:

   Coordinator decision: Both said YES → COMMIT

   Coordinator writes: "DECISION: COMMIT" to WAL

   Coordinator → DB1: "COMMIT the transaction"
   DB1 actions:
     - Make the debit permanent (A: $500 → $400)
     - Write "COMMIT" to WAL
     - Release lock on Account A
   DB1 → Coordinator: "ACK"

   Coordinator → DB2: "COMMIT the transaction"
   DB2 actions:
     - Make the credit permanent (B: $200 → $300)
     - Write "COMMIT" to WAL
     - Release lock on Account B
   DB2 → Coordinator: "ACK"

4. Transaction complete! $100 successfully transferred.
```

## Failure Scenarios & Recovery

### Scenario 1: Participant Fails During Prepare Phase

```
Coordinator sends PREPARE to all participants
  → Node 1: YES
  → Node 2: [CRASHED - no response]
  → Node 3: YES

Coordinator action:
  - Timeout waiting for Node 2
  - Decision: ABORT (not all participants ready)
  - Send ABORT to Node 1 and Node 3
  - Node 2 will abort when it recovers (no PREPARE in log)
```

### Scenario 2: Participant Votes NO

```
Coordinator sends PREPARE to all participants
  → Node 1: YES (has funds)
  → Node 2: NO (insufficient funds)
  → Node 3: YES

Coordinator action:
  - Received a NO vote
  - Decision: ABORT immediately
  - Send ABORT to all participants
  - All participants rollback changes
```

### Scenario 3: Coordinator Crashes After Collecting Votes

**Before writing decision to WAL:**
```
Participants are in "uncertain" state (voted YES, waiting for decision)

Recovery:
  - New coordinator elected or original recovers
  - Reads WAL → no decision recorded
  - Participants still have PREPARE records
  - Options:
    a) Restart the prepare phase
    b) Abort the transaction (safer default)
    c) Query participants to reconstruct state
```

**After writing COMMIT decision to WAL:**
```
Recovery:
  - New coordinator reads WAL → sees COMMIT decision
  - Resends COMMIT to all participants
  - Participants that already committed: idempotent (no-op)
  - Participants that didn't: now commit
  - All participants eventually commit
```

### Scenario 4: Network Partition

```
Coordinator sends COMMIT to participants:
  → Node 1: ACK received
  → Node 2: [NETWORK PARTITION - message lost]
  → Node 3: ACK received

Coordinator action:
  - Retry sending COMMIT to Node 2
  - Node 2 eventually receives and commits
  - If Node 2 was already committed: idempotent operation

Problem: While partitioned, Node 2 holds locks!
  - Blocking problem: other transactions waiting
  - Can lead to timeouts and poor performance
```

## Write-Ahead Log (WAL) Role

The WAL is critical for durability and recovery:

### What Gets Logged

```
Coordinator WAL:
  [T1] START transaction T1
  [T1] PREPARE sent to [Node1, Node2, Node3]
  [T1] VOTE-YES from Node1
  [T1] VOTE-YES from Node2
  [T1] VOTE-YES from Node3
  [T1] DECISION: COMMIT
  [T1] COMMIT sent to all
  [T1] ACK from Node1
  [T1] ACK from Node2
  [T1] ACK from Node3
  [T1] COMPLETE

Participant WAL:
  [T1] PREPARE received
  [T1] Local changes ready (before image saved)
  [T1] VOTE: YES sent
  [T1] COMMIT received
  [T1] Changes applied
  [T1] ACK sent
```

### Recovery Procedure

**Coordinator recovery:**
1. Read WAL to find in-progress transactions
2. For each transaction:
   - If no decision recorded → ABORT
   - If COMMIT decision recorded → Resend COMMIT
   - If ABORT decision recorded → Resend ABORT
   - If COMPLETE recorded → Nothing to do

**Participant recovery:**
1. Read WAL to find in-progress transactions
2. For each transaction:
   - If only PREPARE recorded → Contact coordinator for decision
   - If COMMIT recorded → Ensure changes applied
   - If ABORT recorded → Ensure changes rolled back

## System Design Trade-offs

### Advantages

**✓ Strong Consistency Guarantee**
- Ensures all-or-nothing atomicity across distributed systems
- No partial updates possible

**✓ Simple to Understand**
- Clear two-phase protocol
- Easy to reason about correctness

**✓ ACID Compliance**
- Maintains traditional database guarantees across distribution

**✓ Proven and Mature**
- Well-understood failure modes
- Extensive tooling and support

### Disadvantages

**✗ Blocking Protocol**
- Participants hold locks during both phases
- If coordinator fails, participants are stuck waiting
- **Major impact:** Can block other transactions indefinitely

```
Example:
  Time 0: Transaction T1 locks Account A and B
  Time 1: Coordinator crashes before sending COMMIT
  Time 2: Transaction T2 wants to access Account A
  Result: T2 is BLOCKED until T1 resolves (could be minutes/hours)
```

**✗ Single Point of Failure**
- Coordinator failure halts all transactions
- Requires coordinator failover mechanisms
- Failover itself can be complex

**✗ Performance Overhead**
- Two network round-trips minimum (prepare + commit)
- Synchronous blocking at each phase
- Latency = 2 × (slowest participant) + network delays

```
Example latency calculation:
  Prepare phase: max(Node1: 50ms, Node2: 100ms, Node3: 75ms) = 100ms
  Commit phase:  max(Node1: 50ms, Node2: 100ms, Node3: 75ms) = 100ms
  Total minimum: 200ms per transaction
```

**✗ Scalability Limitations**
- More participants = more coordination overhead
- Performance degrades with distributed scale
- Not suitable for highly distributed systems (100+ nodes)

**✗ Availability Concerns**
- Violates availability in [CAP theorem](../fundamentals/cap-theorem.md)
- Chooses consistency over availability
- During network partitions, system may halt

**✗ Timeout Complexity**
- Need to set appropriate timeouts
- Too short: false negatives
- Too long: resources locked longer
- No perfect timeout value

## Real-World Applications

### Where 2PC is Used

**1. Traditional Distributed Databases**
- **MySQL Cluster (NDB)**: Uses 2PC for cross-shard transactions
- **PostgreSQL**: Supports 2PC via `PREPARE TRANSACTION`
- **Oracle RAC**: Distributed transaction coordination

**2. Enterprise Service Buses (ESB)**
- **IBM WebSphere**: XA transactions across JMS queues
- **Oracle SOA Suite**: Coordinating service calls

**3. Microservices (Limited Use)**
- When strong consistency is absolutely required
- Financial transactions (payments, transfers)
- Inventory reservation systems

**4. Java Transaction API (JTA)**
- Enterprise Java applications using XA protocol
- Application servers coordinating database + message queue

### Where 2PC is NOT Used

**1. Web-Scale Systems**
- **Google, Facebook, Amazon**: Use eventual consistency
- Too slow and blocking for massive scale
- Prefer BASE over ACID

**2. Modern Microservices**
- Netflix, Uber, Airbnb: Use [Saga pattern](saga.md) instead
- Prefer availability and partition tolerance
- Accept eventual consistency

**3. NoSQL Databases**
- MongoDB, Cassandra, DynamoDB: No 2PC support
- Designed for availability and partition tolerance
- Trade consistency for scale

## Alternative Patterns

### 1. Three-Phase Commit (3PC)

Adds a "pre-commit" phase to reduce blocking:

```
Phase 1: CanCommit (like Prepare)
Phase 2: PreCommit (coordinator announces decision)
Phase 3: DoCommit (actual commit)

Advantage: Reduces blocking window
Disadvantage: More complex, still has issues with network partitions
```

### 2. Saga Pattern

Break transaction into sequence of local transactions with compensating actions:

```
Transfer $100 from A to B:

Forward flow:
  1. Debit $100 from A (local transaction)
  2. Credit $100 to B (local transaction)

Compensating flow (if #2 fails):
  1. Credit $100 back to A (compensate)

Advantage: No distributed locks, better availability
Disadvantage: Not truly ACID, eventual consistency
```

### 3. Event Sourcing + CQRS

Store events instead of state, rebuild state from events:

```
Events:
  - MoneyDebited(accountA, $100, txId)
  - MoneyCredited(accountB, $100, txId)

Advantage: Audit trail, flexible consistency models
Disadvantage: Complexity, eventual consistency
```

### 4. Single Database Approach

Avoid distribution altogether:

```
Instead of:
  - DB1: UserService
  - DB2: OrderService
  - 2PC needed for user orders

Use:
  - Single DB with multiple schemas
  - Local ACID transactions

Advantage: Simple, fast, consistent
Disadvantage: Scalability limits, single point of failure
```

## Design Considerations

### When to Use 2PC

**✓ Use when:**
- Strong consistency is absolutely required (financial transactions)
- Number of participants is small (2-5 nodes)
- Latency requirements are relaxed (hundreds of milliseconds acceptable)
- System is not highly distributed
- Network is reliable and low-latency
- Availability can be sacrificed for consistency

**✗ Avoid when:**
- Building web-scale systems (thousands of nodes)
- High throughput required (thousands of TPS)
- Low latency critical (sub-100ms)
- Network partitions are common
- Availability is prioritized over consistency
- Participants are geographically distributed

### Decision Matrix

| Requirement | 2PC Suitable? | Alternative |
|------------|---------------|-------------|
| Financial accuracy | ✓ YES | None - use 2PC |
| Social media likes | ✗ NO | Eventual consistency |
| Bank transfers | ✓ YES | 2PC or Saga with validation |
| Shopping cart | ✗ NO | Eventual consistency |
| Inventory reservation | ~ MAYBE | Saga pattern preferred |
| Multi-region replication | ✗ NO | Async replication |
| Microservices (2-3) | ~ MAYBE | Consider Saga |
| Microservices (10+) | ✗ NO | Saga or Event Sourcing |

## Modern System Design Implications

### Traditional Approach (2PC)

```
Order Service                 Inventory Service
     |                               |
     |--- PREPARE: Reserve SKU123 -->|
     |<-- YES -----------------------|
     |                               |
     |--- COMMIT: Confirm order ----->|
     |<-- ACK -----------------------|

Time: ~200-500ms
Locks held: ~200-500ms
Availability: Reduced during coordinator failure
```

### Modern Approach (Saga/Event-Driven)

```
Order Service                 Event Bus                 Inventory Service
     |                            |                            |
     |-- OrderCreated event ----->|                            |
     |                            |--- OrderCreated --------->|
     |                            |                            |
     |                            |<-- InventoryReserved ------|
     |<-- InventoryReserved ------|                            |

Time: ~50-100ms (async)
Locks held: Minimal (local only)
Availability: High (eventual consistency)
```

### Key Tradeoffs Summary

| Aspect | 2PC | Saga/Event-Driven |
|--------|-----|-------------------|
| Consistency | Strong (immediate) | Eventual |
| Latency | Higher (blocking) | Lower (async) |
| Availability | Lower | Higher |
| Complexity | Lower (protocol) | Higher (compensation) |
| Scalability | Limited | High |
| Debugging | Easier | Harder |

## Key Takeaways

1. **2PC guarantees atomicity** across distributed systems but at the cost of availability and performance

2. **Blocking nature** is the biggest limitation - failed coordinators leave participants locked indefinitely

3. **Write-Ahead Logs** are essential for durability and recovery after failures

4. **Not web-scale** - modern distributed systems prefer eventual consistency patterns (Saga, Event Sourcing)

5. **Still valuable** for small-scale distributed transactions where strong consistency is non-negotiable (financial systems, critical business transactions)

6. **[CAP theorem](../fundamentals/cap-theorem.md) implications** - 2PC chooses CP (Consistency + Partition tolerance) at the expense of Availability

7. **Use judiciously** - consider if you really need distributed transactions or if your problem can be solved with:
   - Single database with proper schema design
   - Eventual consistency with compensating actions
   - Idempotent operations with retry logic

8. **Modern alternatives** (Saga, Event Sourcing, CQRS) trade strong consistency for better availability and scalability - often a worthwhile trade in distributed systems

---

**Related Topics to Explore:**
- Paxos and Raft (Consensus algorithms)
- [Saga Pattern](saga.md) (Distributed transaction alternative)
- Event Sourcing and CQRS
- [CAP Theorem](../fundamentals/cap-theorem.md)
- Distributed Locks (Redlock, Zookeeper)
- XA Protocol (2PC implementation standard)
