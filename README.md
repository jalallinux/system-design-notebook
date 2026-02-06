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
| 4 | Microservices Architecture | Architecture | [📄 English](docs/architecture/microservices.md) | [📄 فارسی](docs/architecture/microservices.fa.md) |
| 5 | API Gateway Pattern | Architecture | [📄 English](docs/architecture/api-gateway.md) | [📄 فارسی](docs/architecture/api-gateway.fa.md) |
| 6 | Circuit Breaker Pattern | Resilience | [📄 English](docs/resilience/circuit-breaker.md) | [📄 فارسی](docs/resilience/circuit-breaker.fa.md) |
| 7 | Event-Driven Architecture | Event-Driven | [📄 English](docs/event-driven/event-driven-architecture.md) | [📄 فارسی](docs/event-driven/event-driven-architecture.fa.md) |
| 8 | CQRS | Data Patterns | [📄 English](docs/data-patterns/cqrs.md) | [📄 فارسی](docs/data-patterns/cqrs.fa.md) |
| 9 | RabbitMQ vs Kafka | Messaging | [📄 English](docs/messaging/rabbitmq-vs-kafka.md) | [📄 فارسی](docs/messaging/rabbitmq-vs-kafka.fa.md) |
| 10 | Serverless Architecture | Architecture | [📄 English](docs/architecture/serverless.md) | [📄 فارسی](docs/architecture/serverless.fa.md) |

### Examples

| Topic | Stack | English | Persian (فارسی) |
|-------|-------|---------|-----------------|
| Circuit Breaker | PHP / Laravel | [📄 English](examples/circuit-breaker.md) | [📄 فارسی](examples/circuit-breaker.fa.md) |

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

### Architecture

#### 4. Microservices Architecture
A comprehensive guide to microservices architectural style for building distributed systems. Learn about:
- Monolithic vs Microservices comparison with trade-offs
- The 9 key characteristics from Martin Fowler (componentization via services, organized around business capabilities, smart endpoints/dumb pipes, decentralized governance, etc.)
- Communication patterns: synchronous (REST, gRPC) vs asynchronous (messaging, events)
- Data management: database per service pattern, consistency challenges
- Key supporting patterns: API Gateway, Circuit Breaker, Service Discovery, Saga, CQRS
- Real-world examples: Netflix, Amazon, Uber architecture evolution

**Best for:** Understanding when to use microservices vs monoliths, service decomposition strategies, and the ecosystem of patterns needed for successful microservices adoption.

#### 5. API Gateway Pattern
The API Gateway pattern for managing client-to-microservice communication. Learn about:
- Request routing, composition, and protocol translation
- Core features: authentication, rate limiting, load balancing, caching, SSL termination
- API Gateway patterns: routing, aggregation, offloading
- Backend for Frontend (BFF) pattern for different client types
- API Gateway vs Load Balancer vs Reverse Proxy comparison
- Implementation approaches: Kong, NGINX, AWS API Gateway, Azure API Management

**Best for:** Understanding API management in microservices, request aggregation, cross-cutting concerns, and the BFF pattern.

#### 6. Circuit Breaker Pattern
A design pattern that prevents cascading failures in distributed systems by monitoring for failures and temporarily blocking requests to failing services. Learn about:
- The three states: CLOSED, OPEN, and HALF-OPEN with state machine diagram
- Fallback strategies (cached responses, default values, alternative services, graceful degradation)
- Circuit Breaker vs Retry pattern comparison
- Implementation with popular libraries (Resilience4j, Polly, Hystrix, Istio)
- Monitoring, alerting, and Prometheus metrics
- Real-world examples from Netflix, Amazon, Uber, and Twitter

**Best for:** Understanding resilience patterns in microservices, cascading failure prevention, graceful degradation, and building fault-tolerant distributed systems.

### Event-Driven

#### 7. Event-Driven Architecture
A comprehensive guide to event-driven architecture patterns and their applications. Learn about:
- Four patterns from Martin Fowler: Event Notification, Event-Carried State Transfer, Event Sourcing, CQRS
- Event delivery guarantees: at-most-once, at-least-once, exactly-once
- Message brokers vs event streams comparison
- Choreography vs orchestration for event-driven workflows
- Event Sourcing deep dive: event store, snapshots, temporal queries
- Trade-offs: loose coupling and scalability vs complexity and debugging difficulty

**Best for:** Understanding event-driven patterns, Event Sourcing, and building loosely coupled, scalable systems.

### Data Patterns

#### 8. CQRS (Command Query Responsibility Segregation)
An architectural pattern that separates read operations from write operations using different models. Learn about:
- CQS principle vs CQRS pattern
- Traditional CRUD vs CQRS architecture comparison
- Command model (write side) and Query model (read side)
- Natural pairing with Event Sourcing
- Synchronization strategies and eventual consistency implications
- Read model optimization (denormalized views, materialized views, hybrid databases)
- Real-world examples (e-commerce product catalog, banking ledger system)

**Best for:** Understanding read/write separation patterns, eventual consistency handling, and optimizing systems with very different read and write requirements.

### Messaging

#### 9. RabbitMQ vs Kafka
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

#### 10. Serverless Architecture
A comprehensive guide to serverless computing, covering both Backend as a Service (BaaS) and Function as a Service (FaaS). Learn about:
- BaaS vs FaaS: two facets of serverless
- How FaaS works: event triggers, cold starts, scale to zero
- Serverless vs Traditional Server vs Containers comparison
- Architecture patterns: API backend, event processing pipeline, scheduled tasks, stream processing
- Cold start problem and mitigation strategies
- State management challenges and solutions
- Vendor lock-in concerns and multi-cloud strategies
- Real-world examples: Coca-Cola, iRobot, Thomson Reuters

**Best for:** Understanding serverless computing trade-offs, FaaS patterns, and when serverless is the right architectural choice.

### Examples

#### Circuit Breaker - Laravel Example
A practical PHP/Laravel implementation of the Circuit Breaker pattern, building a gold price fetching service with:
- `CircuitBreaker` service class using Laravel Cache as state store
- `GoldPriceService` wrapping an external API with circuit breaker protection
- State transition table and request flow diagrams
- Cache key management for multi-process environments

**Best for:** Hands-on understanding of how to implement a circuit breaker in a real Laravel application.

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
- ~~CQRS~~ ✅
- Event Sourcing
- ~~CAP Theorem~~ ✅
- ~~Circuit Breaker~~ ✅
- ~~Microservices Architecture~~ ✅
- ~~API Gateway~~ ✅
- ~~Event-Driven Architecture~~ ✅
- ~~Serverless Architecture~~ ✅
- Distributed caching strategies
- Load balancing patterns
- Database replication strategies
- Service mesh patterns
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
│   │   ├── README.md                            # Folder table of contents
│   │   ├── cap-theorem.md                       # CAP Theorem (English)
│   │   └── cap-theorem.fa.md                    # CAP Theorem (Persian)
│   ├── distributed-transactions/                # Transaction coordination patterns
│   │   ├── README.md                            # Folder table of contents
│   │   ├── two-phase-commit.md                  # Two-Phase Commit (English)
│   │   ├── two-phase-commit.fa.md               # Two-Phase Commit (Persian)
│   │   ├── saga.md                              # Saga Pattern (English)
│   │   └── saga.fa.md                           # Saga Pattern (Persian)
│   ├── architecture/                            # Architectural styles and patterns
│   │   ├── README.md                            # Folder table of contents
│   │   ├── microservices.md                     # Microservices Architecture (English)
│   │   ├── microservices.fa.md                  # Microservices Architecture (Persian)
│   │   ├── api-gateway.md                       # API Gateway Pattern (English)
│   │   ├── api-gateway.fa.md                    # API Gateway Pattern (Persian)
│   │   ├── serverless.md                        # Serverless Architecture (English)
│   │   └── serverless.fa.md                     # Serverless Architecture (Persian)
│   ├── resilience/                              # Resilience & fault tolerance patterns
│   │   ├── README.md                            # Folder table of contents
│   │   ├── circuit-breaker.md                   # Circuit Breaker (English)
│   │   └── circuit-breaker.fa.md                # Circuit Breaker (Persian)
│   ├── event-driven/                            # Event-driven architecture patterns
│   │   ├── README.md                            # Folder table of contents
│   │   ├── event-driven-architecture.md         # Event-Driven Architecture (English)
│   │   └── event-driven-architecture.fa.md      # Event-Driven Architecture (Persian)
│   ├── data-patterns/                           # Data management patterns
│   │   ├── README.md                            # Folder table of contents
│   │   ├── cqrs.md                              # CQRS (English)
│   │   └── cqrs.fa.md                           # CQRS (Persian)
│   └── messaging/                               # Messaging & streaming systems
│       ├── README.md                            # Folder table of contents
│       ├── rabbitmq-vs-kafka.md                 # RabbitMQ vs Kafka (English)
│       └── rabbitmq-vs-kafka.fa.md              # RabbitMQ vs Kafka (Persian)
├── examples/                                    # Practical implementation examples
│   ├── circuit-breaker.md                       # Circuit Breaker Laravel Example (English)
│   └── circuit-breaker.fa.md                    # Circuit Breaker Laravel Example (Persian)
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
