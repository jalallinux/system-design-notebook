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
| 4 | Monolithic Architecture | Architecture | [📄 English](docs/architecture/monolithic.md) | [📄 فارسی](docs/architecture/monolithic.fa.md) |
| 5 | Microservices Architecture | Architecture | [📄 English](docs/architecture/microservices.md) | [📄 فارسی](docs/architecture/microservices.fa.md) |
| 6 | API Gateway Pattern | Architecture | [📄 English](docs/architecture/api-gateway.md) | [📄 فارسی](docs/architecture/api-gateway.fa.md) |
| 7 | Backend for Frontend (BFF) Pattern | Architecture | [📄 English](docs/architecture/backend-for-frontend.md) | [📄 فارسی](docs/architecture/backend-for-frontend.fa.md) |
| 8 | Circuit Breaker Pattern | Resilience | [📄 English](docs/resilience/circuit-breaker.md) | [📄 فارسی](docs/resilience/circuit-breaker.fa.md) |
| 9 | Messaging | Communication | [📄 English](docs/communication/messaging.md) | [📄 فارسی](docs/communication/messaging.fa.md) |
| 10 | Event-Driven Architecture | Event-Driven | [📄 English](docs/event-driven/event-driven-architecture.md) | [📄 فارسی](docs/event-driven/event-driven-architecture.fa.md) |
| 11 | Domain Event | Event-Driven | [📄 English](docs/event-driven/domain-event.md) | [📄 فارسی](docs/event-driven/domain-event.fa.md) |
| 12 | Event Sourcing | Event-Driven | [📄 English](docs/event-driven/event-sourcing.md) | [📄 فارسی](docs/event-driven/event-sourcing.fa.md) |
| 13 | CQRS | Data Patterns | [📄 English](docs/data-patterns/cqrs.md) | [📄 فارسی](docs/data-patterns/cqrs.fa.md) |
| 14 | Database per Service | Data Patterns | [📄 English](docs/data-patterns/database-per-service.md) | [📄 فارسی](docs/data-patterns/database-per-service.fa.md) |
| 15 | Shared Database | Data Patterns | [📄 English](docs/data-patterns/shared-database.md) | [📄 فارسی](docs/data-patterns/shared-database.fa.md) |
| 16 | API Composition | Data Patterns | [📄 English](docs/data-patterns/api-composition.md) | [📄 فارسی](docs/data-patterns/api-composition.fa.md) |
| 17 | Command-side Replica | Data Patterns | [📄 English](docs/data-patterns/command-side-replica.md) | [📄 فارسی](docs/data-patterns/command-side-replica.fa.md) |
| 18 | RabbitMQ vs Kafka | Messaging | [📄 English](docs/messaging/rabbitmq-vs-kafka.md) | [📄 فارسی](docs/messaging/rabbitmq-vs-kafka.fa.md) |
| 19 | Transactional Outbox | Messaging | [📄 English](docs/messaging/transactional-outbox.md) | [📄 فارسی](docs/messaging/transactional-outbox.fa.md) |
| 20 | Polling Publisher | Messaging | [📄 English](docs/messaging/polling-publisher.md) | [📄 فارسی](docs/messaging/polling-publisher.fa.md) |
| 21 | Transaction Log Tailing (CDC) | Messaging | [📄 English](docs/messaging/transaction-log-tailing.md) | [📄 فارسی](docs/messaging/transaction-log-tailing.fa.md) |
| 22 | Remote Procedure Invocation (RPI) | Communication | [📄 English](docs/communication/rpi.md) | [📄 فارسی](docs/communication/rpi.fa.md) |
| 23 | Idempotent Consumer | Communication | [📄 English](docs/communication/idempotent-consumer.md) | [📄 فارسی](docs/communication/idempotent-consumer.fa.md) |
| 24 | Domain-Specific Protocol | Communication | [📄 English](docs/communication/domain-specific-protocol.md) | [📄 فارسی](docs/communication/domain-specific-protocol.fa.md) |
| 25 | Serverless Architecture | Architecture | [📄 English](docs/architecture/serverless.md) | [📄 فارسی](docs/architecture/serverless.fa.md) |

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

#### 4. Monolithic Architecture
The traditional architectural approach where an application is built as a single, unified unit. Learn about:
- What defines a monolithic architecture and its internal structure
- Benefits: simple development, testing, deployment, and debugging
- Challenges at scale: tight coupling, deployment bottleneck, technology lock-in
- When monolithic architecture is the right choice (startups, small teams, simple domains)
- Modular monolith as a middle ground between monolith and microservices
- Migration strategies from monolith to microservices (Strangler Fig pattern)
- Real-world examples and when companies outgrow monoliths

**Best for:** Understanding the baseline architecture before microservices, knowing when a monolith is the right choice, and planning migration strategies.

#### 5. Microservices Architecture
A comprehensive guide to microservices architectural style for building distributed systems. Learn about:
- Monolithic vs Microservices comparison with trade-offs
- The 9 key characteristics from Martin Fowler (componentization via services, organized around business capabilities, smart endpoints/dumb pipes, decentralized governance, etc.)
- Communication patterns: synchronous (REST, gRPC) vs asynchronous (messaging, events)
- Data management: database per service pattern, consistency challenges
- Key supporting patterns: API Gateway, Circuit Breaker, Service Discovery, Saga, CQRS
- Real-world examples: Netflix, Amazon, Uber architecture evolution

**Best for:** Understanding when to use microservices vs monoliths, service decomposition strategies, and the ecosystem of patterns needed for successful microservices adoption.

#### 6. API Gateway Pattern
The API Gateway pattern for managing client-to-microservice communication. Learn about:
- Request routing, composition, and protocol translation
- Core features: authentication, rate limiting, load balancing, caching, SSL termination
- API Gateway patterns: routing, aggregation, offloading
- Backend for Frontend (BFF) pattern for different client types
- API Gateway vs Load Balancer vs Reverse Proxy comparison
- Implementation approaches: Kong, NGINX, AWS API Gateway, Azure API Management

**Best for:** Understanding API management in microservices, request aggregation, cross-cutting concerns, and the BFF pattern.

#### 7. Backend for Frontend (BFF) Pattern
A dedicated backend service per frontend type, evolved from the API Gateway pattern. Learn about:
- One BFF per frontend type (web, mobile, TV, partner API) with tailored data shapes
- Origin: Sam Newman coined the term, evolved from API Gateway pattern
- How BFF solves the "fat gateway" problem of one-size-fits-all APIs
- BFF vs general API Gateway: key differences and when to combine them
- Implementation approaches: REST BFF, GraphQL BFF, gRPC BFF
- Data aggregation, transformation, and authentication at the BFF layer
- BFF ownership model: frontend team owns their BFF
- Real-world examples: Netflix, SoundCloud, Spotify

**Best for:** Understanding how to optimize APIs for different client types, frontend team autonomy, and reducing over-fetching in microservices architectures.

### Resilience

#### 8. Circuit Breaker Pattern
A design pattern that prevents cascading failures in distributed systems by monitoring for failures and temporarily blocking requests to failing services. Learn about:
- The three states: CLOSED, OPEN, and HALF-OPEN with state machine diagram
- Fallback strategies (cached responses, default values, alternative services, graceful degradation)
- Circuit Breaker vs Retry pattern comparison
- Implementation with popular libraries (Resilience4j, Polly, Hystrix, Istio)
- Monitoring, alerting, and Prometheus metrics
- Real-world examples from Netflix, Amazon, Uber, and Twitter

**Best for:** Understanding resilience patterns in microservices, cascading failure prevention, graceful degradation, and building fault-tolerant distributed systems.

### Communication

#### 9. Messaging
An asynchronous inter-service communication pattern where services exchange messages through a message broker without direct coupling. Learn about:
- Core messaging concepts: producers, consumers, channels, brokers
- Message types: Command, Event, and Document messages
- Channel types: Point-to-Point vs Publish-Subscribe
- Delivery guarantees: at-most-once, at-least-once, exactly-once
- Message ordering and its trade-offs
- Popular broker comparison: RabbitMQ, Kafka, SQS/SNS, Azure Service Bus, Google Pub/Sub
- Messaging vs synchronous RPI comparison and decision framework
- Real-world adoption: Netflix, Uber, LinkedIn, Shopify, Airbnb

**Best for:** Understanding asynchronous inter-service communication, message broker selection, and designing loosely coupled event-driven systems.

### Event-Driven

#### 10. Event-Driven Architecture
A comprehensive guide to event-driven architecture patterns and their applications. Learn about:
- Four patterns from Martin Fowler: Event Notification, Event-Carried State Transfer, Event Sourcing, CQRS
- Event delivery guarantees: at-most-once, at-least-once, exactly-once
- Message brokers vs event streams comparison
- Choreography vs orchestration for event-driven workflows
- Event Sourcing deep dive: event store, snapshots, temporal queries
- Trade-offs: loose coupling and scalability vs complexity and debugging difficulty

**Best for:** Understanding event-driven patterns, Event Sourcing, and building loosely coupled, scalable systems.

#### 11. Domain Event
A pattern from Domain-Driven Design (DDD) for capturing significant state changes in the domain model. Learn about:
- Domain Events as immutable facts about what happened (OrderPlaced, PaymentReceived)
- Event structure: eventId, eventType, timestamp, aggregateId, payload, metadata
- Naming conventions (past tense) and event granularity (coarse vs fine-grained)
- Publishing from aggregates vs application services
- Guaranteed publishing with the Transactional Outbox pattern
- Domain Events vs Integration Events distinction
- E-commerce example: OrderPlaced triggering inventory, payment, and notification
- Real-world usage at Amazon, Netflix, Uber, Shopify, and Stripe

**Best for:** Understanding how microservices communicate state changes, DDD event modeling, reliable event publishing, and building extensible event-driven systems.

#### 12. Event Sourcing
A pattern that stores the state of an entity as a sequence of immutable events rather than the current state. Learn about:
- Traditional state storage vs event sourcing approach
- Event store architecture and schema design
- State reconstruction by replaying events
- Snapshot optimization for performance
- Natural pairing with CQRS for separate read/write models
- Event schema evolution strategies (upcasting, versioning, weak schema)
- Banking and shopping cart examples with temporal queries
- Real-world implementations: LMAX Exchange, banking, healthcare, betting platforms

**Best for:** Understanding how to build audit-complete systems, implement temporal queries, and combine Event Sourcing with CQRS for complex domain models.

### Data Patterns

#### 13. CQRS (Command Query Responsibility Segregation)
An architectural pattern that separates read operations from write operations using different models. Learn about:
- CQS principle vs CQRS pattern
- Traditional CRUD vs CQRS architecture comparison
- Command model (write side) and Query model (read side)
- Natural pairing with Event Sourcing
- Synchronization strategies and eventual consistency implications
- Read model optimization (denormalized views, materialized views, hybrid databases)
- Real-world examples (e-commerce product catalog, banking ledger system)

**Best for:** Understanding read/write separation patterns, eventual consistency handling, and optimizing systems with very different read and write requirements.

#### 14. Database per Service
A foundational data management pattern in microservices architecture where each service owns a private database inaccessible by other services. Learn about:
- Private database per service with varying levels of isolation (private tables, private schema, private DB server, polyglot persistence)
- How services access each other's data exclusively through APIs
- Data consistency challenges and solutions (Saga pattern, eventual consistency)
- Cross-service querying strategies (API Composition, CQRS)
- Polyglot persistence: choosing the best database technology per service
- Real-world examples: e-commerce with PostgreSQL, MongoDB, and Elasticsearch
- Trade-offs between data isolation and operational complexity

**Best for:** Understanding data architecture in microservices, service decoupling at the data layer, polyglot persistence strategies, and managing distributed data consistency.

#### 15. Shared Database
The traditional approach to data management where multiple services share a single centralized database. Learn about:
- Why shared database is the natural starting point for monolith-to-microservices migrations
- Two variants: shared tables vs shared server with separate schemas
- ACID transactions across service boundaries without distributed transaction complexity
- The coupling problem: schema changes breaking multiple services simultaneously
- Comparison with Database per Service pattern
- When shared database is acceptable and when to migrate away
- Real-world usage in enterprises, financial institutions, and legacy systems
- Step-by-step migration example from monolith to microservices

**Best for:** Understanding data management trade-offs during microservices migration, when shared databases are a pragmatic choice, and planning the transition to Database per Service.

#### 16. API Composition
A pattern for querying data spread across multiple microservices by invoking their APIs and combining the results in memory. Learn about:
- How API Composition replaces SQL JOINs when migrating to microservices
- Two approaches: API Gateway as composer vs dedicated composer service
- Failure handling strategies: partial results, fallbacks, default values
- Performance optimization: sequential vs parallel vs hybrid execution
- Availability impact: combined availability decreases with each service dependency
- When to outgrow API Composition and move to CQRS
- Real-world usage: Netflix API Gateway, GraphQL as composition layer, Amazon product pages

**Best for:** Understanding cross-service data querying in microservices, choosing between API Composition and CQRS, and managing the availability trade-offs.

#### 17. Command-side Replica
A pattern where a service maintains a local read-only replica of another service's data to support command-side validation without cross-service calls. Learn about:
- How command-side replicas differ from CQRS read models
- Event-driven synchronization from the source service
- Using replicas for real-time command validation without network calls
- Eventual consistency considerations and conflict resolution
- When to use vs alternatives (API calls, CQRS, shared database)
- Real-world usage at Amazon, Uber, Netflix, and banking systems

**Best for:** Understanding how services can validate commands locally using replicated data, reducing inter-service coupling and latency in command processing.

### Messaging

#### 18. RabbitMQ vs Kafka
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

#### 19. Transactional Outbox
A pattern for reliably publishing events/messages as part of a database transaction in microservices. Learn about:
- The dual write problem: atomically updating a database and publishing to a message broker
- Writing events to an outbox table within the same ACID transaction as business data
- Two delivery mechanisms: Polling Publisher and Transaction Log Tailing (CDC)
- Outbox table schema design and indexing strategies
- Exactly-once delivery considerations and idempotent consumers
- Real-world implementations with Debezium, MassTransit, Axon Framework
- Cleanup strategies for outbox table management

**Best for:** Understanding reliable event publishing in microservices, solving the dual write problem, and enabling consistent event-driven communication without distributed transactions.

#### 20. Polling Publisher
A delivery mechanism for the Transactional Outbox pattern that periodically queries the outbox table for unpublished events and forwards them to the message broker. Learn about:
- How Polling Publisher works as one of two delivery mechanisms for Transactional Outbox
- Polling cycle: SELECT unpublished events, publish to broker, UPDATE as published
- Event state machine: PENDING → PUBLISHING → PUBLISHED / FAILED / DEAD LETTER
- Poll interval tuning: trade-offs between latency and database load
- Failure handling and retry strategies
- Comparison with Transaction Log Tailing (CDC) approach
- Scaling strategies: partitioned polling, leader election, row locking

**Best for:** Understanding the simpler, database-agnostic approach to outbox event delivery, and knowing when Polling Publisher is preferable to CDC-based alternatives.

#### 21. Transaction Log Tailing (CDC)
A pattern that reads the database's transaction log to detect and publish changes as events, commonly used as the delivery mechanism for the Transactional Outbox pattern. Learn about:
- How database transaction logs work (WAL in PostgreSQL, binlog in MySQL)
- Change Data Capture (CDC) as a non-invasive, low-latency event publishing mechanism
- Debezium architecture: DB connector, Kafka Connect, and Outbox Event Router
- Comparison with Polling Publisher approach (latency, reliability, operational complexity)
- CDC tools ecosystem: Debezium, Maxwell's Daemon, AWS DMS, LinkedIn Databus
- Configuration and operational considerations (log retention, schema evolution, monitoring)
- Real-world usage at LinkedIn, Uber, Airbnb, Shopify, and Zalando

**Best for:** Understanding how to reliably publish database changes as events, CDC tool selection, and building production-grade event pipelines for microservices.

### Communication (continued)

#### 22. Remote Procedure Invocation (RPI)
A synchronous inter-service communication pattern where a client sends a request to a remote service and blocks until a response is received (request/reply). Learn about:
- RPI as the synchronous counterpart to asynchronous messaging
- REST over HTTP: resource-oriented, JSON, HTTP caching, wide browser support
- gRPC: binary Protocol Buffers, HTTP/2, streaming, code generation for 10+ languages
- REST vs gRPC comparison (performance, serialization, tooling, browser support, streaming)
- Service discovery: client-side vs server-side discovery for locating service instances
- Failure handling: timeouts, retries with exponential backoff, Circuit Breaker integration
- When to use RPI vs Messaging decision framework
- Real-world usage: Google (gRPC origin from Stubby), Netflix, Uber hybrid approaches

**Best for:** Understanding synchronous inter-service communication in microservices, choosing between REST and gRPC, service discovery patterns, and building resilient RPI with proper failure handling.

#### 23. Idempotent Consumer
A messaging pattern that ensures consumers can safely handle duplicate message deliveries, producing the same result regardless of how many times a message is processed. Learn about:
- Why at-least-once delivery in message brokers leads to duplicate messages
- Five implementation strategies: message deduplication, natural idempotency, idempotency keys, optimistic locking, and database constraints
- Message ID tracking tables with TTL-based cleanup
- Idempotency key pattern used by Stripe and PayPal
- Payment processing and order creation examples with duplicate protection
- Real-world implementations: Stripe, AWS SQS FIFO, Kafka consumer groups

**Best for:** Understanding how to build reliable message consumers in microservices, preventing duplicate processing, and implementing idempotency in event-driven architectures.

#### 24. Domain-Specific Protocol
A communication pattern where services use specialized, domain-optimized protocols instead of generic ones like REST or gRPC. Learn about:
- When standard protocols (REST, gRPC) are insufficient for domain-specific needs
- Examples: SMTP for email, AMQP for messaging, FIX for financial trading, HL7/FHIR for healthcare
- Protocol selection based on domain requirements
- Integration challenges with domain-specific protocols in microservices
- Bridging domain protocols with standard APIs using adapter/translator patterns

**Best for:** Understanding when to use specialized protocols, how to integrate domain-specific communication in microservices, and the trade-offs between standard and specialized protocols.

### Architecture (continued)

#### 25. Serverless Architecture
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
- ~~Event Sourcing~~ ✅
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
│   │   ├── monolithic.md                        # Monolithic Architecture (English)
│   │   ├── monolithic.fa.md                     # Monolithic Architecture (Persian)
│   │   ├── microservices.md                     # Microservices Architecture (English)
│   │   ├── microservices.fa.md                  # Microservices Architecture (Persian)
│   │   ├── api-gateway.md                       # API Gateway Pattern (English)
│   │   ├── api-gateway.fa.md                    # API Gateway Pattern (Persian)
│   │   ├── backend-for-frontend.md              # Backend for Frontend (English)
│   │   ├── backend-for-frontend.fa.md           # Backend for Frontend (Persian)
│   │   ├── serverless.md                        # Serverless Architecture (English)
│   │   └── serverless.fa.md                     # Serverless Architecture (Persian)
│   ├── resilience/                              # Resilience & fault tolerance patterns
│   │   ├── README.md                            # Folder table of contents
│   │   ├── circuit-breaker.md                   # Circuit Breaker (English)
│   │   └── circuit-breaker.fa.md                # Circuit Breaker (Persian)
│   ├── event-driven/                            # Event-driven architecture patterns
│   │   ├── README.md                            # Folder table of contents
│   │   ├── event-driven-architecture.md         # Event-Driven Architecture (English)
│   │   ├── event-driven-architecture.fa.md      # Event-Driven Architecture (Persian)
│   │   ├── domain-event.md                      # Domain Event (English)
│   │   ├── domain-event.fa.md                   # Domain Event (Persian)
│   │   ├── event-sourcing.md                    # Event Sourcing (English)
│   │   └── event-sourcing.fa.md                 # Event Sourcing (Persian)
│   ├── data-patterns/                           # Data management patterns
│   │   ├── README.md                            # Folder table of contents
│   │   ├── cqrs.md                              # CQRS (English)
│   │   ├── cqrs.fa.md                           # CQRS (Persian)
│   │   ├── database-per-service.md              # Database per Service (English)
│   │   ├── database-per-service.fa.md           # Database per Service (Persian)
│   │   ├── shared-database.md                   # Shared Database (English)
│   │   ├── shared-database.fa.md                # Shared Database (Persian)
│   │   ├── api-composition.md                   # API Composition (English)
│   │   ├── api-composition.fa.md                # API Composition (Persian)
│   │   ├── command-side-replica.md              # Command-side Replica (English)
│   │   └── command-side-replica.fa.md           # Command-side Replica (Persian)
│   ├── communication/                           # Inter-service communication patterns
│   │   ├── README.md                            # Folder table of contents
│   │   ├── messaging.md                         # Messaging (English)
│   │   ├── messaging.fa.md                      # Messaging (Persian)
│   │   ├── rpi.md                               # Remote Procedure Invocation (English)
│   │   ├── rpi.fa.md                            # Remote Procedure Invocation (Persian)
│   │   ├── idempotent-consumer.md               # Idempotent Consumer (English)
│   │   ├── idempotent-consumer.fa.md            # Idempotent Consumer (Persian)
│   │   ├── domain-specific-protocol.md          # Domain-Specific Protocol (English)
│   │   └── domain-specific-protocol.fa.md       # Domain-Specific Protocol (Persian)
│   └── messaging/                               # Messaging & streaming systems
│       ├── README.md                            # Folder table of contents
│       ├── rabbitmq-vs-kafka.md                 # RabbitMQ vs Kafka (English)
│       ├── rabbitmq-vs-kafka.fa.md              # RabbitMQ vs Kafka (Persian)
│       ├── transactional-outbox.md              # Transactional Outbox (English)
│       ├── transactional-outbox.fa.md           # Transactional Outbox (Persian)
│       ├── polling-publisher.md                 # Polling Publisher (English)
│       ├── polling-publisher.fa.md              # Polling Publisher (Persian)
│       ├── transaction-log-tailing.md           # Transaction Log Tailing / CDC (English)
│       └── transaction-log-tailing.fa.md        # Transaction Log Tailing / CDC (Persian)
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
