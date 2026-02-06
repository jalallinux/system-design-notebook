# CAP Theorem

## 1. Introduction

The **CAP theorem** (also known as Brewer's theorem) is one of the most fundamental theorems in distributed systems. It states that a distributed data store cannot simultaneously provide more than two out of three guarantees: **Consistency**, **Availability**, and **Partition Tolerance**. Originally conjectured by Eric Brewer in 2000 and formally proven by Seth Gilbert and Nancy Lynch in 2002, the CAP theorem has become a cornerstone for understanding the trade-offs inherent in distributed system design.

Understanding the CAP theorem is essential for anyone designing or evaluating distributed databases, microservices architectures, or any system that operates across multiple networked nodes. It provides a framework for reasoning about what guarantees a system can and cannot make, especially during failure scenarios.

## 2. The Three Properties

### Consistency (C)

**Definition:** A read sees all previously completed writes. Every read request receives the most recent write or an error.

In a consistent system, all nodes see the same data at the same time. When a write is acknowledged, any subsequent read from any node in the system will return that updated value. This is sometimes called **linearizability** or **strong consistency**.

```
Client writes X=5 to Node A
        |
        v
    [Node A: X=5] ---replication---> [Node B: X=5] ---replication---> [Node C: X=5]
        |
Client reads X from Node B => returns 5 ✓
Client reads X from Node C => returns 5 ✓
```

**Key Points:**
- All nodes agree on the current state of data
- No stale reads are possible
- Writes may need to be coordinated across nodes before being acknowledged
- Stronger than eventual consistency but may introduce latency

### Availability (A)

**Definition:** Every request (read or write) receives a non-error response, without the guarantee that it contains the most recent write.

An available system guarantees that every request made to a non-failing node will return a response. The system remains operational and responsive even under adverse conditions. In the strict CAP sense, availability means **all** nodes can serve reads and writes at all times.

```
Client ---request---> [Node A] ---> Response ✓  (always responds)
Client ---request---> [Node B] ---> Response ✓  (always responds)
Client ---request---> [Node C] ---> Response ✓  (always responds)
```

**Key Points:**
- Every non-crashed node must return a response for every request
- No timeout or error responses are allowed
- The response doesn't need to be the most recent data
- The system never refuses to answer a query

### Partition Tolerance (P)

**Definition:** The system continues to operate despite arbitrary message loss or failure of part of the network.

A network partition occurs when communication between some nodes is disrupted. Partition tolerance means the system continues to function even when the network is unreliable - messages between nodes can be dropped, delayed, or reordered. In real-world distributed systems, network partitions are inevitable, making partition tolerance a practical necessity rather than an optional feature.

```
Before Partition:
[Node A] <---> [Node B] <---> [Node C]

During Partition:
[Node A] <--X---> [Node B] <---> [Node C]
    |                    |
 Isolated         Can still communicate
```

**Key Points:**
- Network partitions are a fact of life in distributed systems
- Messages between nodes can be arbitrarily delayed or lost
- The system must handle split-brain scenarios
- Partition tolerance is not optional in any real distributed system

## 3. The Theorem Explained

### Original Formulation: "Pick Two"

Brewer originally framed the CAP theorem as forcing a choice of "two out of three" properties, suggesting three viable design options:

| Combination | Description |
|-------------|-------------|
| **CP** (Consistency + Partition Tolerance) | Consistent and partition-tolerant, but may sacrifice availability during partitions |
| **AP** (Availability + Partition Tolerance) | Available and partition-tolerant, but may return stale data |
| **CA** (Consistency + Availability) | Consistent and available, but cannot tolerate partitions |

### Modern Interpretation

The "pick two" framing, while useful for intuition, is somewhat misleading. A more accurate modern interpretation recognizes that:

1. **Partition tolerance is not optional.** In any real distributed system, network partitions can and will occur. A system that is not partition-tolerant will, by definition, be forced to give up either consistency or availability when a partition happens.

2. **CA is not a coherent option** for distributed systems. A system that cannot handle network partitions is essentially a single-node system or a system that assumes a perfectly reliable network - an unrealistic assumption.

3. **The real choice is between C and A during a partition.** When the network is functioning normally, a system can provide both consistency and availability. The CAP theorem only forces a choice when a partition actually occurs.

> *"During a network partition, a distributed system must choose either Consistency or Availability."*

This modern framing is far more practical and has been endorsed by Brewer himself in his 2012 retrospective article.

### Visual Representation

```
                    Consistency (C)
                         /\
                        /  \
                       /    \
                      / CP   \
                     /  systems\
                    /    (e.g., \
                   / HBase,     \
                  / MongoDB,     \
                 / FoundationDB)  \
                /------------------\
               / CA        AP       \
              / (Single   systems    \
             /  node or  (e.g.,       \
            /  no real   Cassandra,    \
           /  partition  DynamoDB,      \
          /   tolerance) CouchDB)       \
         /______________________________ \
   Availability (A) ---------- Partition Tolerance (P)
```

## 4. CP Systems: Choosing Consistency

A **CP system** guarantees consistency and partition tolerance but may sacrifice availability during network partitions. When a partition occurs, the system chooses to return an error or timeout rather than return potentially inconsistent data.

### How CP Systems Work

When a network partition occurs in a CP system:

1. The system detects the partition
2. Nodes that cannot confirm they have the latest data stop accepting writes (and possibly reads)
3. Only the partition that can maintain consistency continues to operate
4. When the partition heals, the isolated nodes resynchronize

```
Normal Operation:
[Node A] <--sync--> [Node B] <--sync--> [Node C]
 Write ✓             Write ✓             Write ✓

During Partition:
[Node A]   X    [Node B] <--sync--> [Node C]
 Write ✗          Write ✓             Write ✓
 (unavailable)    (consistent)        (consistent)
```

### Characteristics

- **Strong consistency guarantees**: All reads reflect the most recent write
- **Reduced availability during partitions**: Some nodes may reject requests
- **Majority-based consensus**: Often uses quorum or leader-based protocols
- **Higher latency**: Writes may require coordination across nodes

### Real-World Examples

| System | Consistency Mechanism |
|--------|----------------------|
| **FoundationDB** | Uses Paxos-based coordination servers with majority rule |
| **HBase** | ZooKeeper-based coordination with strong consistency |
| **MongoDB** (default) | Single-primary replication with majority write concern |
| **etcd** | Raft consensus protocol |
| **ZooKeeper** | ZAB (ZooKeeper Atomic Broadcast) protocol |
| **CockroachDB** | Raft consensus with serializable isolation |

### When to Choose CP

- Financial systems where data accuracy is critical (banking, payments)
- Distributed locking and coordination services
- Configuration management systems
- Systems where stale data could cause cascading failures
- Leader election and service discovery

## 5. AP Systems: Choosing Availability

An **AP system** guarantees availability and partition tolerance but may sacrifice consistency during network partitions. When a partition occurs, all nodes continue to accept reads and writes, even though they may return stale or conflicting data.

### How AP Systems Work

When a network partition occurs in an AP system:

1. All nodes continue to accept reads and writes
2. Nodes on different sides of the partition may diverge
3. When the partition heals, the system must resolve conflicts
4. Conflict resolution strategies (last-write-wins, vector clocks, CRDTs) are used

```
Normal Operation:
[Node A] <--sync--> [Node B] <--sync--> [Node C]
  X=5                 X=5                 X=5

During Partition:
[Node A]     X     [Node B] <--sync--> [Node C]
  X=7                X=9                 X=9
(accepts writes)   (accepts writes)    (accepts writes)

After Partition Heals:
[Node A] <--resolve--> [Node B] <--sync--> [Node C]
  X=?                    X=?                 X=?
         (conflict resolution needed)
```

### Data Divergence Problem

As illustrated in the FoundationDB documentation, the consequence of choosing availability is stark:

> *Imagine a simple distributed database consisting of two nodes and a network partition making them unable to communicate. To be Available, each of the two nodes must continue to accept writes from clients. Because the partition makes communication impossible, a write on one node cannot be seen by the other. Such a system is now "a database" in name only.*

During a partition, an AP system is effectively equivalent to multiple independent databases whose contents need not even be related, much less consistent.

### Characteristics

- **Always responds**: No request ever gets an error due to partition
- **Eventual consistency**: Data converges after partition heals
- **Conflict resolution needed**: Must handle divergent writes
- **Lower latency**: No cross-node coordination needed for writes
- **Higher availability**: System never rejects a valid request

### Real-World Examples

| System | Conflict Resolution Strategy |
|--------|------------------------------|
| **Cassandra** | Last-write-wins with tunable consistency |
| **DynamoDB** | Vector clocks and application-level resolution |
| **CouchDB** | Multi-version concurrency with conflict detection |
| **Riak** | Vector clocks, CRDTs, and siblings |
| **Voldemort** | Vector clocks with read-repair |

### When to Choose AP

- Social media feeds where slight staleness is acceptable
- Shopping carts and wishlists
- DNS systems
- Content delivery and caching layers
- IoT sensor data collection
- Systems where high write throughput is more important than immediate consistency

## 6. The Confusion: CAP Availability vs. High Availability

One of the most common sources of confusion around the CAP theorem involves the interpretation of "Availability." There is a critical distinction between **CAP availability** and **operational high availability**.

### CAP Availability

CAP availability is an absolute property: **all** nodes must remain able to read and write even when partitioned. This is a theoretical guarantee about every single node in the system.

### Operational High Availability

Operational high availability means the system remains accessible to clients and satisfies its SLAs (e.g., 99.99% uptime). A system can be highly available in practice even if some nodes are temporarily unable to serve requests during a partition.

### The Key Distinction

A system that keeps **some, but not all**, of its nodes able to read and write during a partition is:
- **NOT** available in the CAP sense
- **But IS** highly available in the practical, operational sense

```
During Partition:
[Node A - isolated]  [Node B] <---> [Node C]
  Cannot write ✗       Writes ✓       Writes ✓

CAP Availability:  NO  (Node A is unavailable)
High Availability: YES (Clients can still reach B and C)
```

This distinction is crucial. Many CP systems (like FoundationDB) are marketed as and practically behave as highly available systems, even though they sacrifice CAP availability. They achieve this by ensuring that the majority of nodes remain operational during partitions, making the system available to most clients even if not to all nodes.

## 7. Practical Example: FoundationDB's Approach

FoundationDB provides an excellent real-world example of how a CP system achieves practical high availability while maintaining strict consistency. This example is drawn from the FoundationDB documentation.

### Architecture

FoundationDB is configured with a set of **coordination servers** that use the **Paxos algorithm** to maintain a small amount of shared state. This state is itself Consistent and Partition-tolerant. The coordination servers determine which partition of the cluster should continue accepting transactions.

**Majority Rule:** FoundationDB selects the partition in which a majority of coordination servers are available as the one that will remain responsive. Only if failures are so pervasive that there is no such partition will the database become truly unavailable.

### Example: Minimal Three-Node Cluster

Consider a small startup running FoundationDB on three machines (A, B, C), each running a database server and a coordination server, configured in `double` redundancy mode (two copies of each piece of data).

**Normal Operation:**
```
[Machine A]  <---->  [Machine B]  <---->  [Machine C]
 DB Server            DB Server            DB Server
 Coord Server         Coord Server         Coord Server
 Data Copy 1          Data Copy 2          Data Copy 1
                      Data Copy 1          Data Copy 2
```

**During Partition (Machine A isolated):**
```
[Machine A]    X    [Machine B]  <---->  [Machine C]
 DB Server          DB Server            DB Server
 Coord Server       Coord Server         Coord Server

 Cannot commit       Full functionality    Full functionality
 (no majority)       (majority: B+C)      (majority: B+C)
```

**What happens:**
1. Machine A cannot commit new transactions because it cannot reach a majority of coordination servers (needs 2 out of 3)
2. Machines B and C can reach each other, forming a majority
3. The replication ensures a full copy of data is available on B and C
4. Clients connected to B or C continue with full read/write access
5. Clients connected only to A experience the database as unavailable

**After Partition Heals:**
1. Machine A can again communicate with the majority of coordination servers
2. A rejoins by receiving missed transactions or, in the worst case, transferring the full database contents
3. All machines resume full fault-tolerant operation

### Production Configuration

In production, FoundationDB is typically configured in `triple` redundancy mode on five or more machines, providing correspondingly higher availability and fault tolerance.

## 8. Beyond CAP: PACELC Theorem

The CAP theorem only describes system behavior during partitions. The **PACELC theorem** (proposed by Daniel Abadi in 2012) extends CAP by also considering the trade-off during normal operation (no partition):

> **If** there is a **P**artition, choose between **A**vailability and **C**onsistency;
> **E**lse (normal operation), choose between **L**atency and **C**onsistency.

```
                    Is there a partition?
                         /        \
                       Yes         No
                      /              \
               Choose:              Choose:
            A or C?               L or C?
              /  \                  /  \
            PA    PC              EL    EC
```

### PACELC Classifications

| System | Partition (P) | Else (E) | Classification |
|--------|--------------|----------|----------------|
| **Cassandra** | PA | EL | PA/EL - Favors availability and low latency |
| **DynamoDB** | PA | EL | PA/EL - Favors availability and low latency |
| **MongoDB** | PC | EC | PC/EC - Favors consistency always |
| **FoundationDB** | PC | EC | PC/EC - Favors consistency always |
| **CockroachDB** | PC | EC | PC/EC - Favors consistency always |
| **Cosmos DB** | PA/PC | EL/EC | Tunable - configurable per request |
| **YugabyteDB** | PC | EC | PC/EC - Favors consistency with low latency |

PACELC is more nuanced than CAP because it acknowledges that even when there are no partitions, there is still a trade-off between latency and consistency due to the need to synchronize replicas.

## 9. Tunable Consistency

Many modern distributed databases don't strictly fall into one CAP category. Instead, they offer **tunable consistency** that allows operators to make trade-off decisions on a per-query or per-table basis.

### How Tunable Consistency Works

The most common approach uses **quorum-based reads and writes**:

- **N**: Total number of replicas
- **W**: Number of replicas that must acknowledge a write
- **R**: Number of replicas that must respond to a read

**Strong consistency** is achieved when: `W + R > N`

```
Example with N=3:

Strong Consistency (W=2, R=2):
  Write to 2/3 nodes, Read from 2/3 nodes
  Guaranteed overlap => always see latest write

Eventual Consistency (W=1, R=1):
  Write to 1/3 nodes, Read from 1/3 nodes
  No guaranteed overlap => may see stale data
```

### Examples of Tunable Systems

**Cassandra** offers multiple consistency levels:
- `ONE`: Write/read to one replica (fast, eventually consistent)
- `QUORUM`: Write/read to majority of replicas (balanced)
- `ALL`: Write/read to all replicas (slow, strongly consistent)
- `LOCAL_QUORUM`: Quorum within the local data center

**DynamoDB** offers:
- Eventually consistent reads (default, higher throughput)
- Strongly consistent reads (always returns latest data, higher latency)

## 10. Common Misconceptions

### Misconception 1: "You must always choose two of three"
**Reality:** The choice is only relevant during partitions. When the network is healthy, you can have both consistency and availability. The CAP theorem forces a trade-off only during partition events.

### Misconception 2: "CP means the system is always unavailable"
**Reality:** CP systems are highly available in practice. They only sacrifice availability on the specific nodes affected by a partition, not the entire system. Most clients continue to be served normally.

### Misconception 3: "AP means data is always inconsistent"
**Reality:** AP systems are eventually consistent. During normal operation (no partitions), data is typically consistent across nodes within milliseconds. Inconsistency only arises during actual partition events.

### Misconception 4: "CAP means you can ignore one property entirely"
**Reality:** All three properties exist on a spectrum. Systems make nuanced trade-offs rather than completely abandoning one property. Many modern databases offer tunable consistency levels.

### Misconception 5: "Network partitions are rare, so CAP doesn't matter"
**Reality:** Network partitions happen more often than you might think, especially in cloud environments, across data centers, and at scale. Any system that runs on more than one machine must plan for partitions.

## 11. CAP Theorem in System Design Interviews

The CAP theorem is a frequent topic in system design interviews. Here's how to effectively discuss it:

### Framework for Analysis

1. **Identify the data type**: What kind of data does the system manage?
2. **Determine consistency requirements**: Can the system tolerate stale reads?
3. **Assess availability requirements**: Can the system tolerate downtime or errors?
4. **Consider the partition scenario**: What happens when nodes can't communicate?
5. **Choose and justify**: Pick CP or AP based on business requirements

### Common Interview Scenarios

| System to Design | Recommended | Reasoning |
|-----------------|-------------|-----------|
| Banking/Payments | CP | Cannot show wrong balance; financial accuracy is critical |
| Social Media Feed | AP | Slight delay in showing new posts is acceptable |
| Inventory/Stock | CP | Overselling due to stale reads is unacceptable |
| DNS | AP | Stale DNS records are better than no resolution |
| Chat Messaging | AP (with ordering) | Message delivery is more important than strict ordering |
| Distributed Lock | CP | Locks must be consistent; split-brain would cause data corruption |
| Shopping Cart | AP | Cart can be merged later; losing items is worse than duplicates |
| Configuration Store | CP | All nodes must see the same configuration |

## 12. Trade-offs and Design Considerations

### Choosing Between CP and AP

| Factor | CP System | AP System |
|--------|-----------|-----------|
| **During Partition** | Some nodes reject requests | All nodes accept requests |
| **Data Guarantee** | Always consistent | Eventually consistent |
| **Write Latency** | Higher (coordination needed) | Lower (local writes) |
| **Read Latency** | Potentially higher | Lower |
| **Conflict Resolution** | Not needed (prevented) | Required (after partition) |
| **Complexity** | Consensus protocols | Conflict resolution logic |
| **Best For** | Financial, coordination, config | Social, caching, IoT, CDN |

### Strategies to Minimize Trade-offs

1. **Separate concerns**: Use CP for critical data (account balances) and AP for less critical data (user preferences) within the same system
2. **Use tunable consistency**: Adjust consistency level per query based on requirements
3. **Design for partition recovery**: Have clear strategies for conflict resolution when partitions heal
4. **Multi-region deployment**: Use consensus within a region (low latency) and eventual consistency across regions
5. **Read replicas**: Serve reads from replicas (potentially stale) while maintaining strong consistency for writes

## 13. Related Concepts

- **Paxos Algorithm**: Consensus protocol used by many CP systems (e.g., FoundationDB) to agree on values across distributed nodes
- **Raft Consensus**: A more understandable consensus algorithm used by etcd, CockroachDB, and others
- **Eventual Consistency**: The consistency model of AP systems where all replicas converge given enough time
- **Strong Consistency (Linearizability)**: The consistency model of CP systems where reads always reflect the latest write
- **CRDTs (Conflict-free Replicated Data Types)**: Data structures that can be replicated across nodes and merged without conflicts
- **Vector Clocks**: Mechanism for tracking causality in distributed systems to detect and resolve conflicts
- **[Two-Phase Commit (2PC)](../distributed-transactions/two-phase-commit.md)**: Distributed transaction protocol that provides consistency but sacrifices availability
- **[Saga Pattern](../distributed-transactions/saga.md)**: Pattern for managing distributed transactions with eventual consistency
- **Quorum**: The minimum number of nodes that must agree for an operation to succeed

## 14. Summary

The CAP theorem establishes a fundamental constraint on distributed systems: during a network partition, you must choose between consistency and availability. Key takeaways:

1. **Partition tolerance is not optional** in any real distributed system - the true choice is between C and A during partitions
2. **CP systems** (FoundationDB, HBase, etcd) sacrifice availability on affected nodes during partitions to maintain data consistency
3. **AP systems** (Cassandra, DynamoDB, CouchDB) sacrifice consistency to remain fully available, requiring conflict resolution
4. **CAP availability ≠ high availability** - a system can be highly available in practice while not meeting CAP's strict definition
5. **The PACELC theorem** extends CAP by considering the latency vs. consistency trade-off during normal operation
6. **Modern databases** often offer tunable consistency, allowing per-query trade-off decisions
7. **System design** requires analyzing business requirements to determine the appropriate consistency model for each component

Understanding the CAP theorem helps architects make informed decisions about database selection, replication strategies, and system behavior during failure scenarios. Rather than viewing it as a limitation, treat it as a guide for understanding what guarantees your system can realistically provide.

---

**References:**
- Brewer, E. (2000). *Towards Robust Distributed Systems* (CAP Conjecture keynote)
- Gilbert, S. & Lynch, N. (2002). *Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services*
- Brewer, E. (2012). *CAP Twelve Years Later: How the "Rules" Have Changed*
- Abadi, D. (2012). *Consistency Tradeoffs in Modern Distributed Database System Design*
- Apple/FoundationDB Documentation - *CAP Theorem*
