# Microservices Architecture

## 1. Introduction

Microservices architecture is an architectural style that structures an application as a collection of loosely coupled, independently deployable services. Each service is self-contained, implements a specific business capability, and communicates with other services through well-defined APIs.

The term "microservices" was popularized by Martin Fowler and James Lewis in their seminal 2014 article, though the architectural patterns had been evolving at companies like Netflix and Amazon since the late 2000s.

**Core Definition:**
> A microservice is a small, autonomous service that works together with other services to deliver business value.

**Key Concept:**
Microservices represent a shift from building monolithic applications to composing applications from small, independent services that can be developed, deployed, and scaled independently.

## 2. Monolithic Architecture

Before diving into microservices, it's essential to understand the monolithic architecture they evolved from.

### 2.1 What is a Monolith?

A monolithic application is built as a single, indivisible unit. All components - UI, business logic, and data access layers - are tightly coupled and run as a single service.

```
┌─────────────────────────────────────────────────┐
│         MONOLITHIC APPLICATION                  │
│                                                 │
│  ┌───────────────────────────────────────────┐ │
│  │        User Interface Layer               │ │
│  └───────────────────────────────────────────┘ │
│                     ↓                           │
│  ┌───────────────────────────────────────────┐ │
│  │        Business Logic Layer               │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐  │ │
│  │  │ Module A │ │ Module B │ │ Module C │  │ │
│  │  └──────────┘ └──────────┘ └──────────┘  │ │
│  └───────────────────────────────────────────┘ │
│                     ↓                           │
│  ┌───────────────────────────────────────────┐ │
│  │        Data Access Layer                  │ │
│  └───────────────────────────────────────────┘ │
│                     ↓                           │
│  ┌───────────────────────────────────────────┐ │
│  │           Single Database                 │ │
│  └───────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
        Single Deployment Unit
```

### 2.2 Monolith Pros and Cons

| Aspect | Advantages | Disadvantages |
|--------|------------|---------------|
| **Development** | Simple to develop initially | Code becomes complex over time |
| **Testing** | Easy to test end-to-end | Long test execution times |
| **Deployment** | Single deployment process | All-or-nothing deployment |
| **Scaling** | Simple vertical scaling | Must scale entire application |
| **Technology** | Single tech stack | Difficult to adopt new technologies |
| **Team Structure** | Easy coordination initially | Teams step on each other's toes |
| **Performance** | No network latency between modules | Performance bottlenecks affect everything |
| **Reliability** | Simple failure model | Single point of failure |

### 2.3 When Monoliths Work Well

Monolithic architecture is appropriate for:

- **Early-stage startups** - Fast iteration, uncertain requirements
- **Small teams** (< 10 developers) - Low coordination overhead
- **Simple domains** - Limited business complexity
- **Prototypes and MVPs** - Quick time to market
- **Applications with tight coupling requirements** - Shared transactional boundaries

## 3. Microservices Characteristics

Martin Fowler and James Lewis identified nine key characteristics of microservices architecture:

### 3.1 Componentization via Services

Components are independently replaceable and upgradeable units. In microservices, componentization is achieved through services rather than libraries.

```
LIBRARY-BASED COMPONENTIZATION        SERVICE-BASED COMPONENTIZATION
┌──────────────────────┐             ┌─────────┐  ┌─────────┐
│   Single Process     │             │Service A│  │Service B│
│  ┌────┐  ┌────┐     │             │ ┌────┐  │  │ ┌────┐  │
│  │Lib │  │Lib │     │             │ │Lib │  │  │ │Lib │  │
│  │ A  │  │ B  │     │             │ │ A  │  │  │ │ B  │  │
│  └────┘  └────┘     │             │ └────┘  │  │ └────┘  │
└──────────────────────┘             └────┬────┘  └────┬────┘
   In-memory calls                        │            │
                                          └─────┬──────┘
                                            Network calls
```

**Benefits:**
- Independent deployment
- Technology diversity
- Clear component boundaries
- Fault isolation

### 3.2 Organized Around Business Capabilities

Services are organized around business capabilities, not technical layers (Conway's Law in action).

```
TRADITIONAL LAYERED TEAMS           MICROSERVICES BUSINESS TEAMS

┌────────────────────┐              ┌──────────────────────┐
│   UI Team          │              │  Order Service Team  │
├────────────────────┤              │  ┌────────────────┐  │
│   Business Team    │              │  │ UI             │  │
├────────────────────┤              │  ├────────────────┤  │
│   Database Team    │              │  │ Business Logic │  │
└────────────────────┘              │  ├────────────────┤  │
        ↓                           │  │ Database       │  │
  Coordination                      │  └────────────────┘  │
   Overhead                         └──────────────────────┘
                                    ┌──────────────────────┐
                                    │ Payment Service Team │
                                    │  ┌────────────────┐  │
                                    │  │ UI             │  │
                                    │  ├────────────────┤  │
                                    │  │ Business Logic │  │
                                    │  ├────────────────┤  │
                                    │  │ Database       │  │
                                    │  └────────────────┘  │
                                    └──────────────────────┘
```

### 3.3 Products Not Projects

Teams own the service for its entire lifetime - "You build it, you run it" (Amazon CTO Werner Vogels).

**Project Mentality:**
- Build and hand off to operations
- Team disbands after delivery
- Limited long-term responsibility

**Product Mentality:**
- Ongoing ownership and evolution
- Team stays with the service
- Full DevOps responsibility
- Business outcome focused

### 3.4 Smart Endpoints and Dumb Pipes

Use simple, lightweight communication mechanisms. Services receive requests, process them, and send responses - the complexity is in the endpoints, not the infrastructure.

```
ESB (Enterprise Service Bus) - AVOID
┌─────────┐           ┌─────────────────┐           ┌─────────┐
│Service A│──────────▶│  Smart ESB      │──────────▶│Service B│
└─────────┘           │ • Routing       │           └─────────┘
                      │ • Transformation│
                      │ • Orchestration │
                      │ • Business Logic│
                      └─────────────────┘

MICROSERVICES - PREFERRED
┌─────────┐           ┌─────────┐           ┌─────────┐
│Service A│──────────▶│Dumb Pipe│──────────▶│Service B│
│ (Smart) │  HTTP/    │ (HTTP,  │  Request  │ (Smart) │
│         │  Message  │  Msg    │           │         │
└─────────┘           │ Queue)  │           └─────────┘
                      └─────────┘
```

Common "dumb pipes":
- HTTP/REST
- gRPC
- Message brokers ([RabbitMQ, Kafka](../messaging/rabbitmq-vs-kafka.md))
- Simple message queues

### 3.5 Decentralized Governance

No single technology stack for all services. Teams choose the best tool for their specific problem.

**Traditional Governance:**
- Standardized on single platform (Java EE, .NET)
- Central architecture board approves all decisions
- One-size-fits-all solutions

**Decentralized Governance:**
- Polyglot programming (Java, Go, Python, Node.js)
- Teams choose their tools
- Shared practices, not mandated standards
- Internal open-source model

### 3.6 Decentralized Data Management

Each service manages its own database. No shared database across services.

```
SHARED DATABASE (Monolith)
┌─────────┐  ┌─────────┐  ┌─────────┐
│Service A│  │Service B│  │Service C│
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┴────────────┘
                  ↓
        ┌──────────────────┐
        │ Shared Database  │
        └──────────────────┘

DATABASE PER SERVICE (Microservices)
┌─────────┐       ┌─────────┐       ┌─────────┐
│Service A│       │Service B│       │Service C│
└────┬────┘       └────┬────┘       └────┬────┘
     ↓                 ↓                 ↓
┌─────────┐       ┌─────────┐       ┌─────────┐
│  DB A   │       │  DB B   │       │  DB C   │
└─────────┘       └─────────┘       └─────────┘
```

**Benefits:**
- Right database for the job (SQL, NoSQL, Graph)
- Independent schema evolution
- Service isolation

**Challenges:**
- Distributed transactions (see [Two-Phase Commit](../distributed-transactions/two-phase-commit.md) and [Saga Pattern](../distributed-transactions/saga.md))
- Data consistency (see [CAP Theorem](../fundamentals/cap-theorem.md))
- Cross-service queries

### 3.7 Infrastructure Automation

Heavy reliance on automated testing, deployment, and infrastructure provisioning.

**Essential Automation:**
- Continuous Integration/Continuous Deployment (CI/CD)
- Automated testing (unit, integration, contract, end-to-end)
- Infrastructure as Code (Terraform, CloudFormation)
- Container orchestration (Kubernetes, Docker Swarm)
- Automated monitoring and alerting
- Service discovery and load balancing

### 3.8 Design for Failure

Services must be designed to tolerate failures of dependent services.

**Resilience Patterns:**
- [Circuit Breaker](../resilience/circuit-breaker.md) - Prevent cascading failures
- Timeouts - Fail fast
- Retries with exponential backoff
- Bulkheads - Isolate resources
- Fallbacks - Graceful degradation
- Health checks and monitoring

```
SERVICE FAILURE HANDLING

Client                Service A               Service B
  │                      │                      │
  │──────request────────▶│                      │
  │                      │─────request─────────▶│
  │                      │                      │ (Service B fails)
  │                      │                      ✗
  │                      │
  │                      │◀──Circuit Breaker────┘
  │                      │   (Opens after N failures)
  │                      │
  │                      │──Fallback Response──▶
  │◀──fallback result────│
  │                      │
```

### 3.9 Evolutionary Design

Design for replaceability. Services should be small enough to be rewritten if needed.

**Principles:**
- Services are disposable - can be replaced
- Continuous refactoring
- Incremental migration from monolith
- Tolerate multiple versions simultaneously
- Progressive delivery (canary, blue-green deployments)

## 4. Monolith vs Microservices Comparison

| Dimension | Monolithic Architecture | Microservices Architecture |
|-----------|------------------------|----------------------------|
| **Deployment** | Single deployment unit | Multiple independent deployments |
| **Scaling** | Scale entire application | Scale individual services independently |
| **Technology Stack** | Single technology stack | Polyglot - different stacks per service |
| **Development** | Simple initially, complex over time | Complex initially, manageable over time |
| **Team Size** | Works well with small teams | Better for larger organizations |
| **Database** | Single shared database | Database per service |
| **Transactions** | ACID transactions easy | Distributed transactions complex |
| **Data Consistency** | Strong consistency | Eventual consistency often required |
| **Testing** | Easier end-to-end testing | Complex integration testing |
| **Performance** | Fast in-memory calls | Network latency between services |
| **Debugging** | Easier to trace execution | Distributed tracing required |
| **Deployment Risk** | High - all or nothing | Low - isolated deployments |
| **Failure Impact** | Total failure | Partial failure (if designed well) |
| **Time to Market** | Faster initially | Slower setup, faster iteration later |
| **Operational Complexity** | Low | High - requires DevOps maturity |
| **Cost** | Lower infrastructure initially | Higher infrastructure and tooling costs |
| **Team Autonomy** | Low - shared codebase | High - independent teams |
| **Onboarding** | Easier for new developers | Steeper learning curve |
| **Code Reuse** | Easy via shared libraries | Requires service calls or duplication |

## 5. Communication Patterns

Microservices communicate through well-defined interfaces. The choice between synchronous and asynchronous communication significantly impacts system behavior.

### 5.1 Synchronous Communication

Services wait for responses before proceeding. The caller blocks until the call completes.

#### REST (HTTP/JSON)

```
Client              API Gateway           Service A           Service B
  │                     │                     │                   │
  │──GET /orders/123───▶│                     │                   │
  │                     │───GET /order/123───▶│                   │
  │                     │                     │                   │
  │                     │                     │──GET /user/456───▶│
  │                     │                     │                   │
  │                     │                     │◀──user data───────│
  │                     │                     │                   │
  │                     │◀──order data────────│                   │
  │                     │                     │                   │
  │◀──combined data─────│                     │                   │
  │                     │                     │                   │
```

**Characteristics:**
- Request-response pattern
- Simple to understand and debug
- Tight temporal coupling
- HTTP status codes for errors
- RESTful conventions

**Best for:**
- User-facing operations requiring immediate response
- Simple queries
- Operations requiring confirmation

#### gRPC

High-performance RPC framework using Protocol Buffers.

**Advantages over REST:**
- Strongly typed contracts
- Efficient binary serialization
- HTTP/2 multiplexing
- Streaming support
- Code generation for clients

**Example:**
```protobuf
service OrderService {
  rpc GetOrder (OrderRequest) returns (OrderResponse);
  rpc StreamOrders (Empty) returns (stream Order);
}
```

### 5.2 Asynchronous Communication

Services don't wait for responses. Communication via messages and events.

```
ASYNCHRONOUS EVENT-DRIVEN COMMUNICATION

Publisher              Message Broker            Subscriber
  │                         │                        │
  │──publish(OrderCreated)─▶│                        │
  │                         │                        │
  │  (returns immediately)  │                        │
  │                         │                        │
  │                         │──OrderCreated event───▶│
  │                         │                        │
  │                         │                        │ (processes async)
  │                         │                        │
```

**Message Brokers:**
For detailed comparison, see [RabbitMQ vs Kafka](../messaging/rabbitmq-vs-kafka.md).

- **RabbitMQ** - Traditional message queue, AMQP protocol
- **Apache Kafka** - Distributed event streaming platform
- **Amazon SQS** - Managed queue service
- **Google Pub/Sub** - Managed messaging service

**Benefits:**
- Loose temporal coupling
- Better fault tolerance
- Easier to add new consumers
- Natural load leveling
- Supports [Event-Driven Architecture](../event-driven/event-driven-architecture.md)

**Challenges:**
- Eventually consistent
- Harder to debug
- Message ordering complexity
- Duplicate message handling (idempotency required)

### 5.3 Choosing Communication Style

| Factor | Synchronous (REST/gRPC) | Asynchronous (Messages/Events) |
|--------|------------------------|--------------------------------|
| **Response Time** | Immediate response needed | Can process later |
| **Consistency** | Strong consistency required | Eventual consistency acceptable |
| **Coupling** | Tighter coupling | Looser coupling |
| **Complexity** | Simpler to reason about | More complex |
| **Failure Handling** | Immediate failure feedback | Retry mechanisms needed |
| **Scalability** | Limited by slowest service | Better load handling |
| **Use Cases** | User queries, critical operations | Background tasks, notifications, analytics |

## 6. Data Management in Microservices

One of the most challenging aspects of microservices is managing data across service boundaries.

### 6.1 Database per Service Pattern

**Core Principle:** Each microservice has its own database schema. Services cannot access other services' databases directly.

```
┌─────────────────────┐         ┌─────────────────────┐
│  Order Service      │         │  Inventory Service  │
│  ┌──────────────┐   │         │  ┌──────────────┐   │
│  │ Order Logic  │   │         │  │ Inventory    │   │
│  └──────┬───────┘   │         │  │ Logic        │   │
│         │           │         │  └──────┬───────┘   │
│  ┌──────▼───────┐   │         │  ┌──────▼───────┐   │
│  │ Order DB     │   │         │  │ Inventory DB │   │
│  │ - orders     │   │         │  │ - stock      │   │
│  │ - order_items│   │         │  │ - warehouses │   │
│  └──────────────┘   │         │  └──────────────┘   │
└─────────────────────┘         └─────────────────────┘
         │                               │
         └──────────API calls────────────┘
         (NOT direct database access)
```

**Benefits:**
- Loose coupling - services evolve independently
- Right database for the job (polyglot persistence)
- Service isolation - database issues don't propagate
- Clear ownership - team owns data and schema

**Database Technology Choices:**

| Service Type | Database Type | Example |
|--------------|---------------|---------|
| User profiles | Document DB | MongoDB, DynamoDB |
| Product catalog | Document/Search | Elasticsearch, MongoDB |
| Orders | Relational | PostgreSQL, MySQL |
| Session data | Key-Value | Redis, Memcached |
| Social graph | Graph DB | Neo4j, Amazon Neptune |
| Time-series metrics | Time-series DB | InfluxDB, TimescaleDB |
| Event log | Append-only log | Kafka, EventStore |

### 6.2 Data Consistency Challenges

The database-per-service pattern introduces consistency challenges due to the [CAP Theorem](../fundamentals/cap-theorem.md).

**Problem:** How do you maintain consistency across services without distributed transactions?

**Solutions:**

#### Option 1: Two-Phase Commit (2PC)

Traditional distributed transaction protocol. See [Two-Phase Commit](../distributed-transactions/two-phase-commit.md) for details.

```
Coordinator         Service A         Service B
    │                   │                 │
    │───Prepare─────────▶│                 │
    │                   │                 │
    │                   │──Lock resources─┘
    │                   │                 │
    │───Prepare─────────┼────────────────▶│
    │                   │                 │
    │                   │                 │──Lock resources
    │◀──Vote Commit─────│                 │
    │◀──Vote Commit─────┼─────────────────│
    │                   │                 │
    │───Commit──────────▶│                 │
    │───Commit──────────┼────────────────▶│
    │                   │                 │
```

**Issues with 2PC:**
- Blocking protocol - poor performance
- Single point of failure (coordinator)
- Not suitable for microservices
- Violates service autonomy

#### Option 2: Saga Pattern (Recommended)

Sequence of local transactions with compensating transactions. See [Saga Pattern](../distributed-transactions/saga.md) for comprehensive details.

```
SAGA - Choreography Based

Order Service    Payment Service    Inventory Service
     │                  │                   │
     │──OrderCreated───▶│                   │
     │                  │                   │
     │                  │──PaymentProcessed─▶│
     │                  │                   │
     │                  │                   │──InventoryReserved
     │                  │                   │
     │◀─────────────────┴───────────────────┘
     │         OrderCompleted
     │
```

**Saga Benefits:**
- No distributed locks
- Better performance
- Each service remains autonomous
- Natural fit for event-driven systems

**Saga Challenges:**
- Eventual consistency
- Compensating logic complexity
- Lack of isolation (dirty reads possible)

### 6.3 Querying Across Services

**Problem:** How do you query data that spans multiple services?

**Solutions:**

#### API Composition

```
┌─────────────────┐
│  API Gateway    │
│  /order/123     │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌─────┐   ┌──────┐   ┌───────────┐
│Order│   │User  │   │ Inventory │
│Svc  │   │Svc   │   │ Svc       │
└─────┘   └──────┘   └───────────┘
```

1. Gateway queries Order Service
2. Gateway queries User Service with user ID
3. Gateway queries Inventory Service with product IDs
4. Gateway combines results

**Pros:** Simple, works for small-scale queries
**Cons:** Multiple network calls, potential performance issues

#### CQRS (Command Query Responsibility Segregation)

Separate read and write models. See [CQRS](../data-patterns/cqrs.md) for detailed patterns.

```
WRITE SIDE                          READ SIDE
┌─────────────┐                    ┌─────────────┐
│Order Service│                    │Query Service│
│  Write DB   │                    │  Read DB    │
└──────┬──────┘                    └──────▲──────┘
       │                                  │
       │──Events──▶│ Event Stream │──────┘
       │           └──────────────┘
       │                    │
       │                    │ (Projection)
       │                    ▼
       │           ┌──────────────────┐
       │           │ Denormalized     │
       │           │ Read-Optimized   │
       │           │ Views            │
       │           └──────────────────┘
```

**Benefits:**
- Optimized read models
- Scalable queries
- No impact on write performance

**Challenges:**
- Eventual consistency
- Duplication of data
- Complexity

## 7. Key Microservices Patterns

### 7.1 API Gateway Pattern

Single entry point for all clients. See [API Gateway](api-gateway.md) for comprehensive coverage.

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│Mobile   │  │Web App  │  │3rd Party│
│Client   │  │Client   │  │Client   │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┴────────────┘
                  │
          ┌───────▼────────┐
          │  API Gateway   │
          │  • Routing     │
          │  • Auth        │
          │  • Rate Limit  │
          │  • Aggregation │
          └───────┬────────┘
                  │
     ┌────────────┼────────────┐
     │            │            │
┌────▼───┐   ┌───▼────┐   ┌───▼────┐
│Service │   │Service │   │Service │
│   A    │   │   B    │   │   C    │
└────────┘   └────────┘   └────────┘
```

**Responsibilities:**
- Request routing
- Authentication and authorization
- Rate limiting and throttling
- Request/response transformation
- Protocol translation (REST to gRPC)
- Caching
- Monitoring and logging

**Popular Solutions:**
- Netflix Zuul
- Kong
- Amazon API Gateway
- Nginx
- Traefik

### 7.2 Circuit Breaker Pattern

Prevent cascading failures. See [Circuit Breaker](../resilience/circuit-breaker.md) for detailed implementation.

```
CIRCUIT BREAKER STATES

CLOSED (Normal Operation)          OPEN (Failing)
┌────────────────┐                ┌────────────────┐
│ Request        │                │ Request        │
│    ↓           │   Threshold    │    ↓           │
│ [Circuit]      │   Exceeded     │ [Circuit]      │
│    ↓           │   ────────▶    │    ↓           │
│ Service Call   │                │ Fail Fast      │
│    ↓           │                │ (No call)      │
│ Response       │                │    ↓           │
└────────────────┘                │ Fallback       │
                                  └────────────────┘
                                         │
                                    Timeout
                                         │
                                         ▼
                                  ┌────────────────┐
                                  │ HALF-OPEN      │
                                  │ (Testing)      │
                                  │ Limited calls  │
                                  │ to test health │
                                  └────────────────┘
```

**Configuration Parameters:**
- Failure threshold (e.g., 50% failures in 10 requests)
- Timeout period (e.g., 30 seconds)
- Success threshold for closing (e.g., 5 consecutive successes)

### 7.3 Service Discovery Pattern

Services dynamically discover network locations of other services.

```
SERVICE REGISTRY PATTERN

┌─────────────────────────────────────────┐
│         Service Registry                │
│  (Consul, Eureka, Zookeeper, etcd)     │
│                                         │
│  service-a: [10.0.1.5:8080,            │
│              10.0.1.6:8080]            │
│  service-b: [10.0.2.3:8081]            │
└────▲──────────────────────┬─────────────┘
     │                      │
     │ Register             │ Discover
     │ Heartbeat            │ Query
     │                      │
┌────┴──────┐          ┌────▼──────┐
│Service A  │          │Service B  │
│Instance 1 │          │           │
└───────────┘          └───────────┘
┌───────────┐
│Service A  │
│Instance 2 │
└───────────┘
```

**Client-Side Discovery:**
- Client queries registry
- Client performs load balancing
- Examples: Netflix Ribbon, Eureka

**Server-Side Discovery:**
- Load balancer queries registry
- Client calls load balancer
- Examples: AWS ELB, Kubernetes Services

### 7.4 Saga Pattern

Manage distributed transactions. See [Saga Pattern](../distributed-transactions/saga.md).

**Choreography-Based Saga:**
```
Service A         Service B         Service C
    │                 │                 │
    │──Event1────────▶│                 │
    │                 │                 │
    │                 │──Event2────────▶│
    │                 │                 │
    │                 │                 │──Event3
    │◀────────────────┴─────────────────┘
```

**Orchestration-Based Saga:**
```
                Saga Orchestrator
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
    Service A     Service B     Service C
        │             │             │
        │             │             │
        └─────────────┴─────────────┘
              (reports back)
```

### 7.5 CQRS Pattern

Separate models for reading and writing. See [CQRS](../data-patterns/cqrs.md).

```
Commands                                Queries
(Writes)                               (Reads)
    │                                      │
    ▼                                      │
┌─────────┐         ┌──────────┐          │
│Command  │─Events─▶│ Event    │──────────┤
│Service  │         │ Store    │          │
└─────────┘         └──────────┘          │
    │                    │                 │
    ▼                    │ Projection      ▼
┌─────────┐              ▼           ┌─────────┐
│Write DB │                          │Read DB  │
└─────────┘                          │(Denorm) │
                                     └─────────┘
```

**Benefits:**
- Optimized read and write models
- Independent scaling
- Simplified queries
- Event sourcing compatibility

### 7.6 Event Sourcing

Store state as sequence of events rather than current state.

```
TRADITIONAL STATE STORAGE    EVENT SOURCING

Current State:               Event Log:
┌──────────────┐            1. AccountCreated(id:123, balance:0)
│ Account      │            2. MoneyDeposited(id:123, amount:100)
│ id: 123      │            3. MoneyWithdrawn(id:123, amount:30)
│ balance: 70  │            4. MoneyDeposited(id:123, amount:50)
└──────────────┘
                            Current State = Replay all events
                            Balance: 0 + 100 - 30 + 50 = 120
```

**Benefits:**
- Complete audit trail
- Temporal queries (state at any point in time)
- Event-driven architecture natural fit
- Debugging and testing easier

**Challenges:**
- Event schema evolution
- Snapshot mechanism needed for performance
- Increased storage
- Query complexity

## 8. Trade-offs and Challenges

### 8.1 Benefits of Microservices

| Benefit | Description | Impact |
|---------|-------------|--------|
| **Independent Deployment** | Deploy services separately without affecting others | Faster releases, reduced deployment risk |
| **Technology Diversity** | Choose best tool for each job | Innovation, attract diverse talent |
| **Scalability** | Scale services independently based on demand | Cost efficiency, better performance |
| **Fault Isolation** | Failure in one service doesn't crash entire system | Higher availability |
| **Team Autonomy** | Teams own services end-to-end | Faster development, clear ownership |
| **Easier Maintenance** | Smaller codebases easier to understand | Lower cognitive load, faster onboarding |
| **Replaceability** | Services can be rewritten if needed | Reduced technical debt |
| **Parallel Development** | Teams work independently | Faster time to market |

### 8.2 Drawbacks and Challenges

| Challenge | Description | Mitigation |
|-----------|-------------|------------|
| **Distributed System Complexity** | Network calls, latency, partial failures | Circuit breakers, timeouts, retries |
| **Data Consistency** | No ACID transactions across services | Saga pattern, eventual consistency |
| **Operational Overhead** | More services to deploy, monitor, maintain | Automation, Kubernetes, monitoring tools |
| **Testing Complexity** | Integration testing harder | Contract testing, test environments |
| **Distributed Tracing** | Hard to trace requests across services | Zipkin, Jaeger, OpenTelemetry |
| **Network Latency** | Inter-service calls slower than in-process | Caching, async communication, batching |
| **Service Versioning** | Multiple versions running simultaneously | API versioning strategy, backward compatibility |
| **Security** | More attack surface, inter-service auth | Service mesh, mTLS, API gateway |
| **Data Duplication** | Data replicated across services | Accept as trade-off, keep denormalized data in sync |
| **Team Coordination** | Multiple teams need to coordinate | Clear contracts, documentation, communication |
| **Debugging** | Harder to debug distributed systems | Centralized logging, distributed tracing |
| **Initial Cost** | Infrastructure and tooling expensive | Start with modular monolith, migrate gradually |

### 8.3 The Microservices Tax

Martin Fowler calls the overhead of microservices the "microservices tax." You pay this tax in:

1. **Infrastructure Cost** - More servers, containers, orchestration
2. **Operational Complexity** - Monitoring, logging, tracing tools
3. **Development Overhead** - Service templates, shared libraries
4. **Testing Complexity** - Contract tests, integration tests
5. **Team Coordination** - API documentation, service contracts
6. **Learning Curve** - Distributed systems knowledge required

**Critical Question:** Are the benefits worth the tax for your organization?

## 9. When to Use Microservices

### 9.1 Good Candidates for Microservices

Microservices are appropriate when you have:

**Organizational Factors:**
- Large engineering organization (50+ developers)
- Multiple autonomous teams
- DevOps culture and automation maturity
- Long-term product vision

**Technical Factors:**
- Complex domain with distinct business capabilities
- Different scaling requirements for different features
- Need for technology diversity
- Frequent deployments required

**Business Factors:**
- High availability requirements
- Fast time to market for features
- Different parts of system have different SLAs
- Need to experiment and fail fast

**Real-World Indicators:**
- Monolith deployment takes hours
- Teams stepping on each other's toes
- Scaling entire app when only one feature needs it
- Technology stack holds back innovation

### 9.2 When to Avoid Microservices

Microservices may not be appropriate when:

**Organizational Red Flags:**
- Small team (< 10 developers)
- Limited DevOps maturity
- No automation culture
- First product or startup MVP

**Technical Red Flags:**
- Simple domain
- Tight data coupling requirements
- Real-time performance critical
- Limited distributed systems expertise

**Business Red Flags:**
- Uncertain requirements
- Need to validate business model first
- Limited budget
- Short-term project

### 9.3 The Modular Monolith Alternative

Before jumping to microservices, consider a **modular monolith**:

```
MODULAR MONOLITH
┌─────────────────────────────────────────┐
│          Single Deployment Unit         │
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐│
│  │ Order   │  │Payment  │  │Inventory││
│  │ Module  │  │Module   │  │Module   ││
│  │         │  │         │  │         ││
│  └────┬────┘  └────┬────┘  └────┬────┘│
│       │            │            │     │
│       └────────────┴────────────┘     │
│                    │                  │
│            ┌───────▼────────┐         │
│            │ Shared Database│         │
│            └────────────────┘         │
└─────────────────────────────────────────┘

Well-defined module boundaries
Clear interfaces between modules
Easier to extract to microservices later
```

**Benefits:**
- Lower operational complexity
- ACID transactions
- Easier testing and debugging
- Can evolve to microservices later

**Migration Path:**
1. Start with modular monolith
2. Validate business model
3. Identify boundaries
4. Extract one service at a time
5. Build automation and monitoring
6. Continue extracting as needed

## 10. Real-World Examples

### 10.1 Netflix

**Scale:** 200+ million subscribers, 15,000+ microservices

**Architecture Evolution:**
```
2008: Monolithic DVD rental system
2009: Migration to AWS begins
2012: Fully on AWS, microservices architecture
2020: 700+ microservices
2023: Continued evolution with 15,000+ microservices
```

**Key Services:**
- **Zuul** - API Gateway
- **Eureka** - Service discovery
- **Ribbon** - Client-side load balancing
- **Hystrix** - Circuit breaker (now deprecated, use Resilience4j)
- **Chaos Monkey** - Resilience testing

**Lessons Learned:**
- Design for failure from day one
- Automate everything
- Invest heavily in tooling and observability
- Chaos engineering essential
- Culture change as important as technology

### 10.2 Amazon

**Quote from Werner Vogels (CTO):** "You build it, you run it."

**Architecture:**
- Started microservices transformation in 2001
- Two-pizza teams (small, autonomous teams)
- Service-oriented architecture
- Each team owns services end-to-end

**Key Innovations:**
- AWS grew out of internal service platform
- API-first culture
- Distributed system patterns at scale

**Services Example:**
- Product catalog service
- Pricing service
- Inventory service
- Recommendation service
- Cart service
- Checkout service
- Payment service

### 10.3 Uber

**Scale:** 500+ million riders, 2,200+ microservices

**Evolution:**
```
2012: Monolithic Rails application
2014: Started microservices migration
2016: Hundreds of microservices
2020: 2,200+ microservices
```

**Architecture Highlights:**
- Real-time data processing
- Geo-distributed services
- Event-driven architecture
- Domain-oriented microservices

**Domain Services:**
- Trip management
- Dispatch and matching
- Pricing and surge
- Driver management
- Payment processing
- Maps and routing

**Challenges Faced:**
- Cascading failures in early days
- Debugging distributed systems
- Data consistency across services
- Service discovery at scale

**Solutions:**
- Built TChannel RPC framework (later adopted gRPC)
- Developed Jaeger for distributed tracing
- Created domain-oriented architecture
- Invested in observability platforms

### 10.4 Spotify

**Scale:** 500+ million users, hundreds of microservices

**Team Structure:**
- **Squads** - Small cross-functional teams (< 8 people)
- **Tribes** - Collection of squads in related areas
- **Chapters** - People with similar skills across squads
- **Guilds** - Community of interest across organization

**Architecture:**
- Backend services in Java, Python, Go
- Event-driven communication
- Google Cloud Platform and AWS
- Kubernetes for orchestration

**Key Learnings:**
- Conway's Law is real - team structure matters
- Autonomy requires clear boundaries
- Internal platforms reduce duplication
- Culture and architecture co-evolve

## 11. Migration Strategy

### 11.1 Strangler Fig Pattern

Incrementally replace monolith by putting new services in front.

```
PHASE 1: Monolith                PHASE 2: Partial Migration
┌────────────┐                   ┌────────────┐
│            │                   │  Proxy     │
│  Monolith  │                   └──────┬─────┘
│            │                          │
└────────────┘                    ┌─────┴─────┐
                                  │           │
                                  ▼           ▼
                             ┌──────┐   ┌──────────┐
                             │New   │   │ Monolith │
                             │Svc A │   │(reduced) │
                             └──────┘   └──────────┘

PHASE 3: Full Migration
┌────────────┐
│  Proxy     │
└──────┬─────┘
       │
   ┌───┴───┬────┬────┐
   ▼       ▼    ▼    ▼
┌────┐  ┌────┐┌────┐┌────┐
│SvcA│  │SvcB││SvcC││SvcD│
└────┘  └────┘└────┘└────┘
```

**Steps:**
1. Identify a bounded context to extract
2. Build new microservice
3. Route traffic through proxy/gateway
4. Gradually redirect traffic to new service
5. Remove functionality from monolith
6. Repeat for next service

### 11.2 Anti-Corruption Layer

Translate between monolith and microservices models.

```
┌──────────────┐         ┌────────────────┐         ┌──────────────┐
│  Monolith    │◀───────▶│ Anti-Corruption│◀───────▶│Microservice  │
│              │         │     Layer      │         │              │
│ Old model    │         │  Translation   │         │ New model    │
└──────────────┘         └────────────────┘         └──────────────┘
```

### 11.3 Migration Best Practices

1. **Start with clear boundaries** - Domain-driven design
2. **Extract one service at a time** - Incremental migration
3. **Build automation first** - CI/CD, monitoring
4. **Keep monolith healthy** - Don't let it rot during migration
5. **Measure everything** - Metrics before and after
6. **Plan for rollback** - Feature flags, canary deployments
7. **Team training** - Distributed systems knowledge
8. **Don't rush** - Migration can take years

## 12. Best Practices and Principles

### 12.1 Design Principles

1. **Single Responsibility** - One service, one job
2. **High Cohesion** - Related functionality together
3. **Loose Coupling** - Minimize dependencies
4. **API First** - Design contract before implementation
5. **Idempotency** - Same request, same result (critical for retries)
6. **Stateless Services** - Store state externally
7. **Fail Fast** - Don't wait indefinitely
8. **Design for Failure** - Assume things will break
9. **Backward Compatibility** - Don't break existing clients
10. **Observability** - Built-in logging, metrics, tracing

### 12.2 Operational Best Practices

**Monitoring:**
- Centralized logging (ELK, Splunk, CloudWatch)
- Distributed tracing (Jaeger, Zipkin, AWS X-Ray)
- Metrics and dashboards (Prometheus, Grafana, DataDog)
- Health checks and readiness probes
- Service Level Objectives (SLOs) and Indicators (SLIs)

**Deployment:**
- Blue-green deployments
- Canary releases
- Feature flags
- Rolling updates
- Automated rollback

**Security:**
- mTLS between services
- API gateway for authentication
- Service mesh for zero-trust networking
- Secrets management (HashiCorp Vault, AWS Secrets Manager)
- Regular security audits

**Testing:**
- Unit tests for business logic
- Integration tests for service interactions
- Contract tests (Pact, Spring Cloud Contract)
- End-to-end tests (limited, focused on critical paths)
- Chaos engineering (Chaos Monkey, Gremlin)

### 12.3 Service Sizing Guidelines

**How Small Should a Microservice Be?**

**Not too small:**
- More than a single function
- Has clear business capability
- Can be developed by a team
- Justifies operational overhead

**Not too large:**
- Can be rewritten in 2-4 weeks
- < 1000-2000 lines of code (rule of thumb)
- Single database
- Clear bounded context

**Sam Newman's guidance:** "Microservices should be as small as possible, but no smaller."

## 13. Microservices vs Serverless

For serverless architectures, see [Serverless](serverless.md).

**Quick Comparison:**

| Aspect | Microservices | Serverless (FaaS) |
|--------|--------------|-------------------|
| **Deployment Unit** | Service (container) | Function |
| **State** | Can be stateful | Stateless |
| **Scaling** | Manual/auto-scaling | Auto-scales to zero |
| **Cost Model** | Pay for running instances | Pay per invocation |
| **Startup Time** | Always running | Cold start latency |
| **Control** | More control | Less control |
| **Vendor Lock-in** | Less (containers portable) | More (platform-specific) |

**When to combine:**
- Event processing: Serverless functions
- Core business logic: Microservices
- Scheduled jobs: Serverless
- Long-running processes: Microservices

## 14. Interview Framework

### 14.1 System Design Interview Approach

When designing a microservices system in an interview:

**Step 1: Clarify Requirements**
- What are we building?
- Expected scale (users, requests, data)?
- Key features?
- Performance requirements?
- Availability requirements?

**Step 2: Identify Services**
- What are the core business capabilities?
- What are the bounded contexts?
- Draw service boundaries
- Start with 5-10 services, not 50

**Step 3: Define APIs**
- What operations does each service expose?
- RESTful? gRPC? Events?
- Request/response formats

**Step 4: Data Strategy**
- What data does each service own?
- Which services need to share data?
- How to handle consistency?
- Which database types?

**Step 5: Communication Patterns**
- Synchronous vs asynchronous?
- What events are published?
- What message broker?

**Step 6: Cross-Cutting Concerns**
- How do clients discover services?
- Authentication and authorization?
- How to prevent cascading failures?
- Monitoring and observability?

**Step 7: Discuss Trade-offs**
- Why microservices over monolith?
- Consistency vs availability choices
- Deployment complexity
- Operational overhead

### 14.2 Common Interview Questions

**Q: When would you NOT use microservices?**
- Small team
- Simple domain
- MVP/prototype
- Limited DevOps maturity
- Tight coupling requirements

**Q: How do you handle distributed transactions?**
- Avoid if possible
- Saga pattern (choreography or orchestration)
- Eventual consistency
- Compensating transactions

**Q: How do you prevent cascading failures?**
- Circuit breakers
- Timeouts
- Bulkheads
- Rate limiting
- Graceful degradation

**Q: How many services should a system have?**
- No magic number
- Based on business capabilities
- Team size (2-pizza teams)
- Start small, split as needed
- Typically 5-20 for most systems, not 100+

**Q: How do you version APIs?**
- URL versioning (/v1/resource)
- Header versioning (Accept: application/vnd.api+json; version=1)
- Backward compatibility preferred
- Deprecation strategy

**Q: How do you test microservices?**
- Unit tests per service
- Contract tests (service boundaries)
- Integration tests (limited)
- End-to-end tests (critical paths only)
- Chaos engineering

**Q: Database per service or shared database?**
- Database per service (preferred)
- Enables service autonomy
- Right database for the job
- Challenges with consistency and queries
- Shared database only for legacy migration phase

## 15. Key Takeaways

### Core Concepts

1. **Microservices are independently deployable services** organized around business capabilities
2. **Not a silver bullet** - high operational complexity, distributed system challenges
3. **Database per service** - data autonomy with consistency challenges
4. **Communication patterns matter** - synchronous vs asynchronous, choose wisely
5. **Design for failure** - circuit breakers, timeouts, retries essential
6. **Organizational impact** - Conway's Law, team structure matters
7. **Automation required** - CI/CD, infrastructure as code, monitoring
8. **Eventual consistency** - Saga pattern over distributed transactions
9. **Start with modular monolith** - evolve to microservices when justified
10. **Migration is gradual** - strangler fig pattern, anti-corruption layer

### Critical Patterns

- **API Gateway** - Single entry point for clients
- **Service Discovery** - Dynamic service location
- **Circuit Breaker** - Prevent cascading failures
- **Saga Pattern** - Distributed transaction management
- **CQRS** - Separate read/write models
- **Event Sourcing** - Audit trail and temporal queries
- **Strangler Fig** - Incremental migration strategy

### Success Factors

**Technical:**
- Strong DevOps culture
- Automation everywhere
- Observability from day one
- Clear service boundaries

**Organizational:**
- Autonomous teams
- Cross-functional capabilities
- Clear ownership
- Communication protocols

**Business:**
- Long-term commitment
- Investment in tooling
- Accept higher initial cost
- Focus on specific benefits

### Red Flags to Avoid

- Microservices for the sake of microservices
- Too many services too early
- Shared database across services
- Distributed monolith (tight coupling)
- Lack of automation
- Insufficient monitoring
- Synchronous chains of services
- No clear ownership

### Further Learning

**Essential Reading:**
- "Building Microservices" by Sam Newman
- "Designing Data-Intensive Applications" by Martin Kleppmann
- Martin Fowler's microservices resource guide
- Netflix Tech Blog
- AWS Architecture Blog

**Related Concepts:**
- Domain-Driven Design (DDD)
- [CAP Theorem](../fundamentals/cap-theorem.md)
- [Event-Driven Architecture](../event-driven/event-driven-architecture.md)
- Service Mesh (Istio, Linkerd)
- Container Orchestration (Kubernetes)
- Observability (Logging, Metrics, Tracing)

---

Remember: Microservices solve organizational and scalability problems, but introduce operational complexity. Choose them when the benefits outweigh the costs, not because they're trendy.
