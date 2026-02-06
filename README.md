# System Design Notebook

A comprehensive learning and reference resource for system design concepts, patterns, and best practices.

## About This Repository

This repository contains detailed documentation on various system design topics, explained from both theoretical and practical perspectives. Each topic includes real-world examples, trade-offs, failure scenarios, and design considerations.

## Documentation

All documentation is available in both **English** and **Persian (فارسی)** for accessibility.

### Table of Contents

| # | Topic | English | Persian (فارسی) |
|---|-------|---------|-----------------|
| 1 | Two-Phase Commit (2PC) | [📄 English](docs/two-phase-commit.md) | [📄 فارسی](docs/two-phase-commit.fa.md) |
| 2 | Saga Pattern | [📄 English](docs/saga.md) | [📄 فارسی](docs/saga.fa.md) |
| 3 | CAP Theorem | [📄 English](docs/cap-theorem.md) | [📄 فارسی](docs/cap-theorem.fa.md) |

---

## Topics Covered

### 1. Two-Phase Commit (2PC)
A distributed algorithm for ensuring atomicity in distributed transactions. Learn about:
- Problem statement and solution overview
- Detailed phase breakdown (Prepare & Commit)
- Practical examples (bank transfers)
- Failure scenarios and recovery mechanisms
- Trade-offs and real-world applications
- Alternative patterns (Saga, Event Sourcing, 3PC)
- When to use and when to avoid 2PC

**Best for:** Understanding distributed transaction coordination, ACID guarantees across systems, and CAP theorem implications.

### 2. Saga Pattern
A design pattern for managing distributed transactions across multiple microservices. Learn about:
- Choreography vs Orchestration coordination approaches
- Compensating transactions and failure handling
- Practical e-commerce order processing example
- Isolation challenges and countermeasures
- Saga vs Two-Phase Commit comparison
- Real-world applications and industry adoption
- Modern system design implications (cloud-native, Kubernetes, serverless)

**Best for:** Understanding distributed transaction management in microservices, eventual consistency patterns, and compensating transaction design.

### 3. CAP Theorem
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

---

## How to Use This Repository

1. **Browse by Topic**: Use the table of contents above to find topics of interest
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
├── README.md                    # This file
├── README.fa.md                 # This file (Persian)
├── CLAUDE.md                    # Project instructions for Claude Code
├── .claude/                     # Claude Code configuration
│   └── rules/
│       └── communication.md     # Communication and translation rules
├── docs/                        # Documentation files
│   ├── two-phase-commit.md      # Two-Phase Commit (English)
│   ├── two-phase-commit.fa.md   # Two-Phase Commit (Persian)
│   ├── saga.md                  # Saga Pattern (English)
│   ├── saga.fa.md               # Saga Pattern (Persian)
│   ├── cap-theorem.md           # CAP Theorem (English)
│   └── cap-theorem.fa.md        # CAP Theorem (Persian)
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
