# System Design Notebook

A comprehensive learning and reference resource for system design concepts, patterns, and best practices.

## About This Repository

This repository contains detailed documentation on various system design topics, explained from both theoretical and practical perspectives. Each topic includes real-world examples, trade-offs, failure scenarios, and design considerations.

## Documentation

All documentation is available in both **English** and **Persian (فارسی)** for accessibility. Topics are ordered by **learning priority** — start from the top and work your way down to build a solid mental model.

### Table of Contents

| Priority | Topic | Category | English | Persian (فارسی) |
|----------|-------|----------|---------|-----------------|
| 1 | CAP Theorem | Fundamentals | [📄 English](docs/fundamentals/cap-theorem.md) | [📄 فارسی](docs/fundamentals/cap-theorem.fa.md) |
| 2 | Two-Phase Commit (2PC) | Distributed Transactions | [📄 English](docs/distributed-transactions/two-phase-commit.md) | [📄 فارسی](docs/distributed-transactions/two-phase-commit.fa.md) |
| 3 | Saga Pattern | Distributed Transactions | [📄 English](docs/distributed-transactions/saga.md) | [📄 فارسی](docs/distributed-transactions/saga.fa.md) |
| 4 | RabbitMQ vs Kafka | Messaging | [📄 English](docs/messaging/rabbitmq-vs-kafka.md) | [📄 فارسی](docs/messaging/rabbitmq-vs-kafka.fa.md) |

---

## Topics Covered

### Fundamentals

#### 1. CAP Theorem
A fundamental theorem in distributed systems that defines the trade-offs between consistency, availability, and partition tolerance. Learn about:
- The three CAP properties explained in depth
- Modern interpretation vs. original "pick two" formulation
- CP systems (FoundationDB, HBase, etcd) and AP systems (Cassandra, DynamoDB)
- CAP availability vs. operational high availability distinction
- Practical example: FoundationDB's fault tolerance with Paxos
- PACELC theorem extending CAP for normal operation
- Tunable consistency with quorum-based reads/writes
- Common misconceptions and interview frameworks

**Best for:** Understanding distributed system trade-offs, database selection criteria, replication strategies, and system behavior during network partitions.

### Distributed Transactions

#### 2. Two-Phase Commit (2PC)
A distributed algorithm for ensuring atomicity in distributed transactions. Learn about:
- Problem statement and solution overview
- Detailed phase breakdown (Prepare & Commit)
- Practical examples (bank transfers)
- Failure scenarios and recovery mechanisms
- Trade-offs and real-world applications
- Alternative patterns (Saga, Event Sourcing, 3PC)
- When to use and when to avoid 2PC

**Best for:** Understanding distributed transaction coordination, ACID guarantees across systems, and CAP theorem implications.

#### 3. Saga Pattern
A design pattern for managing distributed transactions across multiple microservices. Learn about:
- Choreography vs Orchestration coordination approaches
- Compensating transactions and failure handling
- Practical e-commerce order processing example
- Isolation challenges and countermeasures
- Saga vs Two-Phase Commit comparison
- Real-world applications and industry adoption
- Modern system design implications (cloud-native, Kubernetes, serverless)

**Best for:** Understanding distributed transaction management in microservices, eventual consistency patterns, and compensating transaction design.

### Messaging

#### 4. RabbitMQ vs Kafka
A comprehensive comparison of two of the most popular messaging technologies in distributed systems. Learn about:
- Core concepts: message broker vs event streaming platform
- Background workers vs event-driven processing patterns
- Message retention, replay capability, and consumer models
- Push-based (RabbitMQ) vs pull-based (Kafka) delivery
- Routing flexibility, scaling approaches, and state management
- Decision framework for choosing the right technology
- Common mistakes and real-world architecture examples
- When to use each, and when to use both together

**Best for:** Understanding messaging system selection criteria, asynchronous communication patterns, and designing event-driven or task-based architectures.

---

## How to Use This Repository

1. **Follow the Learning Path**: Topics are ordered by priority — start with fundamentals before moving to patterns and tool comparisons
2. **Language Selection**: Choose your preferred language (English or Persian)
3. **Deep Learning**: Each document provides comprehensive coverage with:
   - Clear explanations with diagrams
   - Step-by-step practical examples
   - Real-world use cases
   - Design trade-offs and considerations
   - Related topics for further exploration

## Contributing

This repository is continuously updated with new system design topics. Stay tuned for more content on:
- Distributed consensus algorithms (Paxos, Raft)
- ~~Saga Pattern~~ ✅
- Event Sourcing and CQRS
- ~~CAP Theorem~~ ✅
- Distributed caching strategies
- Load balancing patterns
- Database replication strategies
- And much more...

## Repository Structure

```
system-design-notebook/
├── README.md                                    # This file
├── README.fa.md                                 # This file (Persian)
├── CLAUDE.md                                    # Project instructions for Claude Code
├── .claude/                                     # Claude Code configuration
│   └── rules/
│       └── communication.md                     # Communication and translation rules
├── docs/                                        # Documentation files
│   ├── fundamentals/                            # Core distributed systems theory
│   │   ├── cap-theorem.md                       # CAP Theorem (English)
│   │   └── cap-theorem.fa.md                    # CAP Theorem (Persian)
│   ├── distributed-transactions/                # Transaction coordination patterns
│   │   ├── two-phase-commit.md                  # Two-Phase Commit (English)
│   │   ├── two-phase-commit.fa.md               # Two-Phase Commit (Persian)
│   │   ├── saga.md                              # Saga Pattern (English)
│   │   └── saga.fa.md                           # Saga Pattern (Persian)
│   └── messaging/                               # Messaging & streaming systems
│       ├── rabbitmq-vs-kafka.md                 # RabbitMQ vs Kafka (English)
│       └── rabbitmq-vs-kafka.fa.md              # RabbitMQ vs Kafka (Persian)
└── ... (more topics coming soon)
```

## Languages

- **English**: Primary documentation language
- **Persian (فارسی)**: Complete translations for Persian-speaking developers

---

## License

This repository is intended for educational purposes.

## Feedback

If you find any issues or have suggestions for new topics, please feel free to contribute or provide feedback.

---

**Last Updated:** February 6, 2026
