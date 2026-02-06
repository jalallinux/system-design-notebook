# Remote Procedure Invocation (RPI)

## 1. Introduction

Remote Procedure Invocation (RPI) is a synchronous inter-service communication pattern in which a client sends a request to a remote service and **blocks until a response is received** (request/reply). It is one of the oldest and most widely used communication mechanisms in distributed systems, dating back to the original Remote Procedure Call (RPC) concept proposed by Bruce Jay Nelson in 1981.

**Why It Matters:**

In a [microservices architecture](../architecture/microservices.md), services must communicate with each other to fulfill business operations. RPI provides the simplest mental model for that communication: a service calls another service just like calling a local function, except the call travels over the network. This familiarity makes RPI the default choice for many teams building distributed systems.

**Core Principle:**

The caller invokes a procedure on a remote service, serializes the request, sends it over the network, and waits for the remote service to deserialize, process, and return a serialized response. From the caller's perspective the interaction looks like a regular function call.

```
┌──────────────┐                          ┌──────────────┐
│              │   1. Request (blocked)    │              │
│    Client    │─────────────────────────▶│   Service    │
│   Service    │                          │   (Remote)   │
│              │◀─────────────────────────│              │
│              │   2. Response (unblock)   │              │
└──────────────┘                          └──────────────┘
```

**RPI vs Messaging:**

While RPI is synchronous and request/reply-based, [Messaging](../messaging/rabbitmq-vs-kafka.md) is an asynchronous alternative where the caller sends a message and does not wait for an immediate response. Both patterns have their place; RPI shines when you need an immediate answer, while messaging excels at decoupling and resilience.

---

## 2. Context and Problem

### The Microservices Communication Challenge

When you decompose a monolith into [microservices](../architecture/microservices.md), what used to be in-process method calls become inter-service network calls. Each service owns its own data and logic, so services must collaborate over the network to compose a full business operation.

```
Monolith (single process)          Microservices (distributed)
┌─────────────────────────┐        ┌──────────┐   ┌──────────┐
│  Order  │  Payment      │        │  Order   │──▶│ Payment  │
│  Module │  Module       │   ──▶  │  Service │   │ Service  │
│─────────│───────────────│        └──────────┘   └──────────┘
│  Inventory │ Shipping   │        ┌──────────┐   ┌──────────┐
│  Module    │ Module     │        │Inventory │   │ Shipping │
└─────────────────────────┘        │ Service  │   │ Service  │
                                   └──────────┘   └──────────┘
  In-process calls (fast)          Network calls (latency, failures)
```

### The Problem

**How should services communicate in a microservices architecture when a response is needed immediately?**

Consider an e-commerce checkout flow:

1. The **Order Service** needs to verify that items are in stock by calling the **Inventory Service**.
2. It then needs to charge the customer by calling the **Payment Service**.
3. It needs the result of payment before confirming the order.

Each of these steps requires an immediate answer. The Order Service cannot proceed to step 2 without the result of step 1, and it cannot confirm the order without the result of step 2. This is a natural fit for synchronous, request/reply communication.

---

## 3. Forces

When deciding whether to use RPI, the following forces (trade-offs) come into play:

| Force | In Favor of RPI | Against RPI |
|-------|-----------------|-------------|
| **Simplicity** | Familiar request/reply model; easy to reason about | Hides the complexity of network calls behind a simple interface |
| **Immediacy** | Caller gets a response right away | Caller is blocked while waiting |
| **Developer experience** | Feels like calling a local function; rich tooling | Can mislead developers into ignoring latency and failure modes |
| **Temporal coupling** | N/A | Both services must be running at the same time |
| **Availability** | N/A | Overall availability is the product of individual service availabilities |
| **Cascading failures** | N/A | A slow or failing downstream service can cascade upward |
| **Scalability** | Simple load balancing across instances | Synchronous calls consume threads/connections while waiting |
| **Debugging** | Request/response traces are easy to follow | Long chains of synchronous calls are hard to reason about at scale |

**Availability Impact:**

If Service A calls Service B via RPI and each has 99.5% availability independently, the combined availability for that operation drops to approximately 99.5% x 99.5% = 99.0%. With each additional synchronous hop, availability decreases further. This is one of the strongest arguments for considering asynchronous [messaging](../messaging/rabbitmq-vs-kafka.md) where possible.

---

## 4. Solution

### Use a Request/Reply Protocol

The solution is to adopt a **request/reply protocol** where the client (calling service) invokes a remote procedure on the server (target service) and blocks until the server returns a response.

```
┌───────────┐                                      ┌───────────┐
│  Client   │                                      │  Server   │
│  Service  │                                      │  Service  │
└─────┬─────┘                                      └─────┬─────┘
      │                                                  │
      │  1. Serialize request                            │
      │  2. Send over network ──────────────────────────▶│
      │                                                  │  3. Deserialize
      │                                                  │  4. Process
      │                                                  │  5. Serialize response
      │◀────────────────────────── 6. Send response ─────│
      │  7. Deserialize response                         │
      │  8. Continue processing                          │
      │                                                  │
```

### RPI Technologies

There are several popular technologies for implementing RPI. The most common ones are:

#### 4.1 REST over HTTP

REST (Representational State Transfer) is a resource-oriented architectural style that uses standard HTTP verbs (GET, POST, PUT, DELETE) and typically JSON for serialization.

```
Client                                            Server
  │                                                  │
  │  GET /api/products/42  HTTP/1.1                  │
  │  Accept: application/json                        │
  │─────────────────────────────────────────────────▶│
  │                                                  │
  │  HTTP/1.1 200 OK                                 │
  │  Content-Type: application/json                  │
  │  {"id": 42, "name": "Widget", "price": 9.99}    │
  │◀─────────────────────────────────────────────────│
  │                                                  │
```

**Characteristics:**
- Resource-oriented (nouns, not verbs)
- Human-readable JSON payloads
- Stateless; each request carries all necessary context
- Leverages HTTP caching (ETags, Cache-Control)
- Wide browser and tooling support (curl, Postman, Swagger/OpenAPI)
- Mature ecosystem with extensive middleware

#### 4.2 gRPC (Protocol Buffers)

gRPC is a high-performance, open-source RPC framework originally developed by Google. It uses Protocol Buffers (protobuf) for serialization and HTTP/2 as the transport layer.

```
Client                                            Server
  │                                                  │
  │  HTTP/2 POST /payment.PaymentService/Charge      │
  │  Content-Type: application/grpc+proto             │
  │  [binary protobuf payload]                        │
  │─────────────────────────────────────────────────▶│
  │                                                  │
  │  HTTP/2 200 OK                                   │
  │  Content-Type: application/grpc+proto             │
  │  [binary protobuf response]                       │
  │◀─────────────────────────────────────────────────│
  │                                                  │
```

**Characteristics:**
- Binary Protocol Buffers serialization (smaller, faster)
- HTTP/2 transport (multiplexing, header compression, streaming)
- Strongly typed contracts defined in `.proto` files
- Automatic code generation for 10+ languages
- Supports four communication patterns: unary, server streaming, client streaming, bidirectional streaming
- Built-in deadlines, cancellation, and load balancing

#### 4.3 Other RPI Technologies

- **Apache Thrift:** Facebook-developed RPC framework with its own IDL and binary protocol. Similar to gRPC but predates it.
- **GraphQL:** A query language for APIs where the client specifies exactly what data it needs. Reduces over-fetching and under-fetching but adds complexity on the server side.

### 4.4 REST vs gRPC Comparison

| Aspect | REST (HTTP/JSON) | gRPC (HTTP/2 + Protobuf) |
|--------|-------------------|--------------------------|
| **Serialization** | JSON (text, human-readable) | Protocol Buffers (binary, compact) |
| **Performance** | Slower (text parsing, larger payloads) | Faster (binary, smaller payloads) |
| **Contract** | OpenAPI/Swagger (optional) | `.proto` files (mandatory, strict) |
| **Code generation** | Optional (various tools) | Built-in (protoc compiler) |
| **Browser support** | Native (fetch, XMLHttpRequest) | Requires grpc-web proxy |
| **Streaming** | Limited (SSE, WebSocket separate) | Native (4 streaming modes) |
| **Caching** | HTTP caching built-in | No native HTTP caching |
| **Tooling** | curl, Postman, browser DevTools | grpcurl, BloomRPC, Evans |
| **Learning curve** | Low | Moderate |
| **Best for** | Public APIs, web frontends | Internal service-to-service calls |
| **Debugging** | Easy (JSON is readable) | Harder (binary payloads) |
| **Backward compat.** | Manual versioning (/v1, /v2) | Built into protobuf (field numbers) |

**When to Choose REST:**
- Public-facing APIs consumed by third parties
- Web browser clients
- Teams new to microservices
- When human-readability of payloads is important
- When HTTP caching is beneficial

**When to Choose gRPC:**
- High-performance internal service-to-service communication
- Polyglot environments needing code generation
- Streaming data requirements (real-time updates)
- Strict contract enforcement is desired
- Bandwidth-constrained environments (mobile, IoT)

### 4.5 Service Discovery

Before a client can invoke a remote procedure, it must know **where** the target service is running. In a dynamic microservices environment, service instances come and go. Service discovery solves this.

```
              Client-Side Discovery                    Server-Side Discovery

┌────────┐  1.Query  ┌──────────┐              ┌────────┐        ┌────────────┐
│ Client │─────────▶│ Service  │              │ Client │───────▶│   Load     │
│Service │           │ Registry │              │Service │        │  Balancer  │
└───┬────┘          └──────────┘              └────────┘        └──────┬─────┘
    │  2.Get instances                                                │
    │  [svc-b:8080,                                          ┌───────┼───────┐
    │   svc-b:8081]                                          ▼       ▼       ▼
    │                                                    ┌──────┐┌──────┐┌──────┐
    │  3.Call directly                                   │Svc-B ││Svc-B ││Svc-B │
    ├──────────────▶ svc-b:8080                          │:8080 ││:8081 ││:8082 │
    │                                                    └──────┘└──────┘└──────┘
```

**Client-Side Discovery:**
- Client queries a service registry (e.g., Consul, Eureka, etcd)
- Client selects an instance (round-robin, random, least connections)
- Client calls the instance directly
- Pros: no additional network hop; client can apply smart routing
- Cons: client needs discovery logic; language-specific libraries

**Server-Side Discovery:**
- Client sends request to a load balancer or [API Gateway](../architecture/api-gateway.md)
- The load balancer queries the registry and forwards to an instance
- Pros: simpler client; discovery logic centralized
- Cons: additional network hop; load balancer is a potential bottleneck

### 4.6 Handling Failures

Because RPI happens over the network, failures are inevitable. A robust RPI implementation must handle:

**Timeouts:**
- Set a maximum time to wait for a response
- Prevents threads from being blocked indefinitely
- Choose timeout values based on SLA of the target service

**Retries with Exponential Backoff:**
- Retry transient failures (network blips, temporary overload)
- Use exponential backoff with jitter to avoid thundering herd
- Set a maximum number of retries

**[Circuit Breaker](../resilience/circuit-breaker.md):**
- Monitor failure rates for each downstream service
- Open the circuit when failures exceed a threshold
- Fail fast instead of wasting resources on doomed calls
- Periodically test recovery (half-open state)

```
RPI Failure Handling Pipeline:

Request
  │
  ▼
┌──────────────────┐
│  Circuit Breaker │──── Open? ───▶ Fail Fast / Fallback
│  Check           │
└────────┬─────────┘
         │ Closed
         ▼
┌──────────────────┐
│  Send Request    │
│  (with timeout)  │
└────────┬─────────┘
         │
    ┌────┴────┐
    │         │
  Success   Failure
    │         │
    ▼         ▼
  Return   ┌──────────────────┐
  Result   │  Retry?          │
           │  (max 3 attempts │
           │   with backoff)  │
           └────────┬─────────┘
                    │
               ┌────┴────┐
               │         │
           Retry OK   Max retries
               │      exceeded
               ▼         │
            Return       ▼
            Result    Report failure
                      to Circuit Breaker
```

---

## 5. Example

### Order Service Calls Payment Service via gRPC

Consider a microservices e-commerce system where the **Order Service** needs to charge a customer during checkout. It uses gRPC for synchronous payment processing.

**Step 1: Define the Contract (.proto file)**

```protobuf
syntax = "proto3";

package payment;

service PaymentService {
  rpc Charge (ChargeRequest) returns (ChargeResponse);
}

message ChargeRequest {
  string order_id = 1;
  string customer_id = 2;
  double amount = 3;
  string currency = 4;
  string payment_method_token = 5;
}

message ChargeResponse {
  string transaction_id = 1;
  PaymentStatus status = 2;
  string message = 3;
}

enum PaymentStatus {
  PAYMENT_SUCCESS = 0;
  PAYMENT_DECLINED = 1;
  PAYMENT_FAILED = 2;
}
```

**Step 2: Server Implementation (Payment Service)**

```python
class PaymentServicer(payment_pb2_grpc.PaymentServiceServicer):

    def Charge(self, request, context):
        # Validate request
        if request.amount <= 0:
            context.set_code(grpc.StatusCode.INVALID_ARGUMENT)
            context.set_details("Amount must be positive")
            return payment_pb2.ChargeResponse()

        # Process payment with external gateway
        try:
            result = payment_gateway.charge(
                customer_id=request.customer_id,
                amount=request.amount,
                currency=request.currency,
                token=request.payment_method_token
            )
            return payment_pb2.ChargeResponse(
                transaction_id=result.txn_id,
                status=payment_pb2.PAYMENT_SUCCESS,
                message="Payment processed successfully"
            )
        except PaymentDeclinedException as e:
            return payment_pb2.ChargeResponse(
                status=payment_pb2.PAYMENT_DECLINED,
                message=str(e)
            )
```

**Step 3: Client Implementation (Order Service)**

```python
class OrderService:

    def __init__(self):
        channel = grpc.insecure_channel('payment-service:50051')
        self.payment_stub = payment_pb2_grpc.PaymentServiceStub(channel)

    def checkout(self, order):
        # Build the charge request
        charge_request = payment_pb2.ChargeRequest(
            order_id=order.id,
            customer_id=order.customer_id,
            amount=order.total,
            currency="USD",
            payment_method_token=order.payment_token
        )

        try:
            # Synchronous gRPC call - blocks until response
            response = self.payment_stub.Charge(
                charge_request,
                timeout=5.0  # 5-second deadline
            )

            if response.status == payment_pb2.PAYMENT_SUCCESS:
                order.mark_paid(response.transaction_id)
                return OrderResult.success(order)
            else:
                return OrderResult.payment_failed(response.message)

        except grpc.RpcError as e:
            if e.code() == grpc.StatusCode.DEADLINE_EXCEEDED:
                # Handle timeout - maybe queue for retry
                return OrderResult.timeout()
            raise
```

**Sequence Diagram:**

```
┌────────┐         ┌──────────┐         ┌──────────┐         ┌──────────┐
│  User  │         │  Order   │         │ Payment  │         │ External │
│Browser │         │ Service  │         │ Service  │         │ Gateway  │
└───┬────┘         └────┬─────┘         └────┬─────┘         └────┬─────┘
    │                    │                    │                    │
    │ POST /checkout     │                    │                    │
    ├───────────────────▶│                    │                    │
    │                    │                    │                    │
    │                    │  gRPC Charge()     │                    │
    │                    ├───────────────────▶│                    │
    │                    │  (blocked)         │                    │
    │                    │                    │  HTTP POST /charge │
    │                    │                    ├───────────────────▶│
    │                    │                    │                    │
    │                    │                    │  200 OK            │
    │                    │                    │◀───────────────────┤
    │                    │                    │                    │
    │                    │  ChargeResponse    │                    │
    │                    │◀───────────────────┤                    │
    │                    │  (unblocked)       │                    │
    │                    │                    │                    │
    │  201 Order Created │                    │                    │
    │◀───────────────────┤                    │                    │
    │                    │                    │                    │
```

---

## 6. Benefits and Drawbacks

### Benefits

**Simple Mental Model:**
RPI mirrors the familiar function-call paradigm. Developers understand request/response intuitively, making onboarding faster and code easier to read.

**Easy Debugging and Tracing:**
Each call has a clear request and response. Distributed tracing tools (Jaeger, Zipkin) can follow the request chain from start to finish. HTTP-based RPI can be inspected with standard tools like curl and browser DevTools.

**Immediate Response:**
The caller knows the result right away. This is essential for operations where the user is waiting (checkout, login, search).

**Rich Ecosystem:**
REST has decades of tooling (OpenAPI, Postman, API gateways). gRPC has strong code generation and type safety. Both have mature libraries in every major language.

**Familiar Error Handling:**
HTTP status codes (REST) and gRPC status codes provide a standard vocabulary for success, client errors, server errors, timeouts, and more.

### Drawbacks

**Temporal Coupling:**
Both the client and server must be running simultaneously. If the server is down, the client's operation fails. This is in contrast to messaging, where the message broker decouples sender and receiver in time.

**Cascading Failures:**
A slow or failing downstream service can cause the caller to block, exhausting its threads and connections. Without [circuit breakers](../resilience/circuit-breaker.md), this can cascade through the entire system.

**Reduced Availability:**
As noted in Section 3, the combined availability of a synchronous call chain is the product of each service's availability. A chain of five services each at 99.5% yields only ~97.5% availability.

```
Availability of synchronous chain:

Service A ──▶ Service B ──▶ Service C ──▶ Service D

Combined = 99.5% x 99.5% x 99.5% x 99.5% = ~98.0%

vs. Asynchronous (messaging):

Service A ──▶ Message Broker ──▶ Service B (processes later)

Service A availability is independent of Service B
```

**Thread/Connection Consumption:**
Each in-flight RPI call consumes a thread or connection in the caller. Under high load, this limits concurrency. Reactive or async frameworks (e.g., Spring WebFlux, asyncio) can mitigate this but add complexity.

**Tight Interface Coupling:**
Both client and server must agree on the interface (API contract). Changes to the API require coordination. gRPC's protobuf field numbering helps with backward compatibility, but breaking changes still require deployment coordination.

### When to Use RPI vs Messaging

| Scenario | Recommended | Reason |
|----------|-------------|--------|
| User waiting for result (checkout, search) | **RPI** | Immediate response needed |
| Fire-and-forget operations (send email, log event) | **Messaging** | No response needed; decouple |
| Long-running workflows (order fulfillment) | **Messaging** | Steps can happen asynchronously |
| Real-time queries (check stock, get price) | **RPI** | Immediate, up-to-date answer needed |
| Event notification (order placed, user signed up) | **Messaging** | Multiple consumers, loose coupling |
| High-availability requirement | **Messaging** | Avoids temporal coupling |
| Simple CRUD operations | **RPI** | Straightforward, well-understood |
| Cross-team/cross-org integration | **RPI (REST)** | Universal, self-documenting |

---

## 7. Related Patterns

- **[Messaging / Async Communication](../messaging/rabbitmq-vs-kafka.md):** The asynchronous alternative to RPI. Instead of blocking for a response, the caller sends a message to a broker (RabbitMQ, Kafka) and continues. The receiver processes the message independently. Use messaging when you do not need an immediate response or when you want to decouple services temporally.

- **[Circuit Breaker](../resilience/circuit-breaker.md):** A resilience pattern that protects RPI callers from cascading failures. When a downstream service fails repeatedly, the circuit breaker opens and fails requests fast instead of letting them block and time out. Essential companion to any RPI implementation.

- **[API Gateway](../architecture/api-gateway.md):** An edge service that acts as a single entry point for external clients. The API Gateway receives RPI calls from clients, routes them to the correct microservice, and can aggregate responses from multiple services. It also handles cross-cutting concerns like authentication, rate limiting, and SSL termination.

- **[Microservices Architecture](../architecture/microservices.md):** The broader architectural style within which RPI operates. Understanding microservices decomposition, data ownership, and service boundaries is essential context for deciding where and how to use RPI.

- **[Event-Driven Architecture](../event-driven/event-driven-architecture.md):** A complementary architectural style. Many systems use RPI for synchronous queries and commands while using events for asynchronous notifications and data propagation. Understanding both patterns helps you choose the right tool for each interaction.

- **[Saga Pattern](../distributed-transactions/saga.md):** When a business transaction spans multiple services, the Saga pattern coordinates the steps. Each step may use RPI for the individual service call, but the overall saga is managed asynchronously with compensating transactions for rollback.

---

## 8. Real-World Usage

### Google (gRPC Origin)

Google developed gRPC based on their internal RPC framework called **Stubby**, which has been used for over 15 years to connect their massive fleet of microservices. Virtually all inter-service communication at Google uses a form of RPI via Stubby/gRPC:

- **Scale:** Billions of RPC calls per second across Google's infrastructure
- **Key design:** Strict protobuf contracts, deadlines propagated through the call chain, automatic load balancing
- **Open-sourced** as gRPC in 2015, it is now used by Netflix, Square, Lyft, Docker, Cisco, and many others

### Netflix

Netflix uses a hybrid approach, combining RPI and messaging:

- **RPI (REST/gRPC):** Used for real-time user-facing operations such as fetching the content catalog, resolving playback URLs, and authenticating users. These operations require immediate responses.
- **Messaging:** Used for background processing, data pipelines, and analytics. Netflix built its own messaging infrastructure on top of Apache Kafka.
- **Resilience:** Netflix pioneered the [Circuit Breaker](../resilience/circuit-breaker.md) pattern (Hystrix) specifically to handle RPI failures at scale.

### Uber

Uber's ride-matching platform relies on RPI for latency-sensitive operations:

- **gRPC** for internal service-to-service communication (matching riders to drivers must happen in real-time)
- **REST** for public-facing APIs (driver and rider mobile apps)
- **Hybrid:** RPI for synchronous commands (request ride, calculate fare), messaging for async updates (trip events, notifications)

### Most Microservice Systems

In practice, the majority of microservice systems use RPI as their primary communication mechanism for synchronous operations:

- REST is the most common choice for public APIs and teams starting with microservices
- gRPC is increasingly adopted for internal service-to-service communication where performance matters
- Messaging complements RPI for asynchronous operations, event propagation, and decoupling

---

## 9. Summary

Remote Procedure Invocation (RPI) is the foundational synchronous communication pattern for microservices. It provides a simple, familiar, request/reply model where the client sends a request and blocks until it receives a response.

**Key Takeaways:**

- **RPI is synchronous:** Client blocks until server responds (request/reply)
- **Two dominant technologies:** REST (HTTP/JSON) for simplicity and universality; gRPC (HTTP/2 + Protobuf) for performance and type safety
- **Service discovery** is required to locate target instances (client-side vs server-side)
- **Failure handling is critical:** Timeouts, retries with backoff, and [circuit breakers](../resilience/circuit-breaker.md) are essential companions
- **Temporal coupling** is the main drawback: both services must be available simultaneously
- **Use RPI when** an immediate response is needed; use [messaging](../messaging/rabbitmq-vs-kafka.md) when decoupling and resilience matter more than immediacy
- **Most real-world systems** use a hybrid: RPI for synchronous queries and commands, messaging for asynchronous events and workflows

```
RPI Decision Summary:

Need immediate response?
│
├── Yes ──▶ Use RPI
│           ├── Public/external API? ──▶ REST
│           ├── Internal, high-perf?  ──▶ gRPC
│           └── Flexible queries?     ──▶ GraphQL
│
└── No  ──▶ Consider Messaging
            ├── Fire-and-forget?      ──▶ Async message
            ├── Event notification?   ──▶ Event streaming
            └── Long-running workflow?──▶ Saga + messaging
```

---

**References:**
- Chris Richardson, "Microservices Patterns" (2018) - Chapter 3: Interprocess Communication
- gRPC Documentation (grpc.io)
- Martin Fowler, "Microservices" article series
- Sam Newman, "Building Microservices" (2021) - Chapter 4: Microservice Communication Styles
- Google Cloud Architecture: Microservices Communication
