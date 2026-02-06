# API Gateway Pattern

## 1. Introduction

An **API Gateway** is a server that acts as an intermediary between clients and backend services in a microservices architecture. It serves as the single entry point for all client requests, providing a unified interface to multiple backend services while handling cross-cutting concerns such as authentication, rate limiting, load balancing, and protocol translation.

The API Gateway pattern is fundamental to modern distributed systems, particularly in [microservices architectures](microservices.md). It decouples clients from the internal service topology, allowing backend services to evolve independently without impacting client applications.

**Key Characteristics:**

- **Single Entry Point**: All client requests pass through the gateway
- **Request Routing**: Intelligently routes requests to appropriate backend services
- **Protocol Translation**: Converts between client and service protocols (HTTP, gRPC, WebSocket)
- **Cross-cutting Concerns**: Centralizes authentication, monitoring, rate limiting, and caching
- **Service Composition**: Aggregates responses from multiple services into a single response

## 2. Problem Statement

In a microservices architecture without an API Gateway, clients face several challenges:

### 2.1 Multiple Round Trips

```
┌─────────────┐
│   Client    │
│  (Mobile)   │
└──────┬──────┘
       │
       │ GET /user/123
       ├──────────────────────────────────────┐
       │                                      │
       │ GET /orders/user/123                 │
       ├────────────────────────┐             │
       │                        │             │
       │ GET /recommendations   │             │
       ├──────────────┐         │             │
       │              │         │             │
       ▼              ▼         ▼             ▼
┌────────────┐  ┌──────────┐ ┌──────────┐ ┌──────────┐
│   User     │  │  Order   │ │Recommend │ │ Inventory│
│  Service   │  │ Service  │ │ Service  │ │ Service  │
└────────────┘  └──────────┘ └──────────┘ └──────────┘
```

**Problems:**

- **High Latency**: Multiple sequential calls increase total response time
- **Network Overhead**: Each call incurs network latency and bandwidth costs
- **Mobile Battery Drain**: Excessive network requests drain mobile device batteries
- **Complex Client Logic**: Clients must orchestrate multiple service calls

### 2.2 Protocol Mismatch

- **Backend Diversity**: Services may use different protocols (REST, gRPC, SOAP, WebSocket)
- **Client Limitations**: Mobile clients may not support all protocols efficiently
- **Protocol Overhead**: gRPC is efficient but not browser-friendly without transcoding

### 2.3 Tight Coupling

- **Service Discovery**: Clients must know the location of every service
- **Service Evolution**: Backend changes require client updates
- **Version Management**: Multiple API versions must be managed across all clients
- **Security Exposure**: Internal service URLs and topology exposed to clients

### 2.4 Duplicated Cross-cutting Concerns

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Service A │     │   Service B │     │   Service C │
│             │     │             │     │             │
│ • Auth      │     │ • Auth      │     │ • Auth      │
│ • Logging   │     │ • Logging   │     │ • Logging   │
│ • Rate Limit│     │ • Rate Limit│     │ • Rate Limit│
│ • SSL       │     │ • SSL       │     │ • SSL       │
│ • Caching   │     │ • Caching   │     │ • Caching   │
└─────────────┘     └─────────────┘     └─────────────┘
```

**Problems:**

- **Code Duplication**: Same logic implemented in every service
- **Inconsistency**: Different implementations lead to different behaviors
- **Maintenance Burden**: Updates must be applied to all services
- **Performance Impact**: Each service performs redundant operations

## 3. How API Gateway Works

### 3.1 Basic Architecture

```
                          ┌─────────────────────────────────────────┐
                          │          API Gateway                    │
                          │                                         │
┌──────────┐              │  ┌────────────────────────────────┐   │
│          │              │  │    Request Processing Layer    │   │
│  Web     │──────┐       │  │  • Authentication              │   │
│  Client  │      │       │  │  • Authorization               │   │
└──────────┘      │       │  │  • Rate Limiting               │   │
                  │       │  │  • Request Validation          │   │
┌──────────┐      ├──────▶│  │  • Protocol Translation        │   │
│          │      │       │  │  • SSL Termination             │   │
│  Mobile  │──────┤       │  └────────────────────────────────┘   │
│  Client  │      │       │                                         │
└──────────┘      │       │  ┌────────────────────────────────┐   │
                  │       │  │    Routing & Composition       │   │
┌──────────┐      │       │  │  • Service Discovery           │   │
│          │      │       │  │  • Load Balancing              │   │
│ 3rd Party│──────┘       │  │  • Request Routing             │   │
│   API    │              │  │  • Response Aggregation        │   │
└──────────┘              │  │  • Circuit Breaking            │   │
                          │  └────────────────────────────────┘   │
                          │                                         │
                          │  ┌────────────────────────────────┐   │
                          │  │    Monitoring & Analytics      │   │
                          │  │  • Logging                     │   │
                          │  │  • Metrics Collection          │   │
                          │  │  • Request Tracing             │   │
                          │  └────────────────────────────────┘   │
                          └───────────────┬─────────────────────────┘
                                          │
                          ┌───────────────┼───────────────┐
                          │               │               │
                          ▼               ▼               ▼
                    ┌──────────┐    ┌──────────┐    ┌──────────┐
                    │  User    │    │  Order   │    │ Payment  │
                    │ Service  │    │ Service  │    │ Service  │
                    └──────────┘    └──────────┘    └──────────┘
```

### 3.2 Request Flow

```
Step 1: Client Request
Client ────[HTTPS Request]────▶ API Gateway
        (https://api.example.com/user/profile)

Step 2: Gateway Processing
API Gateway:
  ├─ Terminate SSL
  ├─ Validate JWT Token
  ├─ Check Rate Limit (100 req/min)
  ├─ Check Cache (Cache Miss)
  ├─ Route Decision: /user/* → User Service
  └─ Transform Request (Add Headers)

Step 3: Backend Communication
API Gateway ────[HTTP/gRPC]────▶ User Service
             (http://user-service:8080/profile)

Step 4: Response Processing
User Service ────[Response]────▶ API Gateway
API Gateway:
  ├─ Transform Response (Remove Internal Fields)
  ├─ Add CORS Headers
  ├─ Cache Response (TTL: 5min)
  ├─ Log Request (Latency: 45ms)
  └─ Compress Response (gzip)

Step 5: Client Response
API Gateway ────[HTTPS Response]────▶ Client
```

### 3.3 Request Aggregation

```
Client Request: GET /dashboard
       │
       ▼
┌──────────────────────────────────────┐
│         API Gateway                  │
│  Parallel Requests:                  │
│    ┌───────────────────────┐         │
│    │ GET /user/profile     │         │
│    │ GET /orders/recent    │         │
│    │ GET /notifications    │         │
│    └───────────────────────┘         │
└──────┬────────┬────────┬─────────────┘
       │        │        │
       ▼        ▼        ▼
   ┌────────┐ ┌────────┐ ┌────────────┐
   │  User  │ │ Order  │ │Notification│
   │Service │ │Service │ │  Service   │
   └───┬────┘ └───┬────┘ └─────┬──────┘
       │          │            │
       │ 50ms     │ 120ms      │ 80ms
       │          │            │
       └──────────┴────────────┘
                  │
                  ▼
       ┌──────────────────────┐
       │   API Gateway        │
       │   Aggregate:         │
       │   {                  │
       │     user: {...},     │
       │     orders: [...],   │
       │     notifications:[..]│
       │   }                  │
       │   Total: 120ms       │
       └──────────────────────┘
                  │
                  ▼
              Client
```

## 4. Core Features

### 4.1 Authentication & Authorization

```
┌─────────────┐
│   Client    │
│ (JWT Token) │
└──────┬──────┘
       │ Authorization: Bearer eyJhbGc...
       ▼
┌─────────────────────────────────────┐
│      API Gateway                    │
│                                     │
│  1. Extract JWT Token               │
│  2. Validate Signature              │
│  3. Check Expiration                │
│  4. Extract Claims (user_id, role)  │
│  5. Check Permissions               │
│     • Is user allowed?              │
│     • Does role have access?        │
│  6. Add User Context to Request     │
│     X-User-ID: 12345                │
│     X-User-Role: admin              │
└──────────┬──────────────────────────┘
           │ Authenticated Request
           ▼
    ┌──────────────┐
    │   Backend    │
    │   Service    │
    │ (Trust Gateway)│
    └──────────────┘
```

**Authentication Methods:**

- **JWT Tokens**: Stateless, self-contained tokens
- **OAuth 2.0**: Delegated authorization
- **API Keys**: Simple key-based authentication
- **mTLS**: Mutual TLS certificate-based authentication
- **Session Cookies**: Traditional session management

### 4.2 Rate Limiting

```
Algorithm: Token Bucket

┌─────────────────────────────────┐
│  Rate Limiter (per user)        │
│                                 │
│  Bucket Capacity: 100 tokens    │
│  Refill Rate: 10 tokens/second  │
│                                 │
│  Current State:                 │
│  ┌─────────────────────────┐   │
│  │ ▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░ │   │
│  │ 45 tokens available      │   │
│  └─────────────────────────┘   │
│                                 │
│  Request: Cost 1 token          │
│  Result: ✓ Allow (44 remaining) │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  If bucket empty:               │
│  ┌─────────────────────────┐   │
│  │ ░░░░░░░░░░░░░░░░░░░░░░░░ │   │
│  │ 0 tokens available       │   │
│  └─────────────────────────┘   │
│                                 │
│  Result: ✗ Reject               │
│  Response: 429 Too Many Requests│
│  Headers:                       │
│    X-RateLimit-Limit: 100       │
│    X-RateLimit-Remaining: 0     │
│    X-RateLimit-Reset: 1609459200│
└─────────────────────────────────┘
```

**Rate Limiting Strategies:**

| Strategy | Scope | Use Case |
|----------|-------|----------|
| Per User | Individual user ID | Fair usage per account |
| Per IP | Source IP address | DDoS protection |
| Per API Key | API key identifier | Third-party API limits |
| Per Endpoint | Specific route | Protect expensive operations |
| Global | Entire gateway | Infrastructure protection |

### 4.3 Load Balancing

```
┌─────────────────────────────────────┐
│         API Gateway                 │
│      Load Balancer Module           │
└──────────┬──────────────────────────┘
           │
    ┌──────┴───────┐
    │ Routing      │
    │ Algorithm:   │
    │ Round Robin  │
    └──────┬───────┘
           │
    ┌──────┴──────────────────────┐
    │                             │
    ▼                             ▼
┌─────────┐  Healthy          ┌─────────┐  Healthy
│Instance │  ✓ 200 OK         │Instance │  ✓ 200 OK
│   #1    │  Latency: 45ms    │   #2    │  Latency: 52ms
│ 75% CPU │  Connections: 150 │ 60% CPU │  Connections: 120
└─────────┘                   └─────────┘

    ▼                             ▼
┌─────────┐  Unhealthy        ┌─────────┐  Healthy
│Instance │  ✗ Timeout        │Instance │  ✓ 200 OK
│   #3    │  Circuit: OPEN    │   #4    │  Latency: 48ms
│    -    │  (Excluded)       │ 65% CPU │  Connections: 135
└─────────┘                   └─────────┘
```

**Load Balancing Algorithms:**

| Algorithm | Description | Best For |
|-----------|-------------|----------|
| Round Robin | Distribute requests sequentially | Uniform backend capacity |
| Weighted Round Robin | Assign weights based on capacity | Heterogeneous instances |
| Least Connections | Route to instance with fewest connections | Long-lived connections |
| Least Response Time | Route to fastest responding instance | Variable workload complexity |
| IP Hash | Consistent routing based on client IP | Session affinity |
| Random | Random selection | Simple, stateless services |

### 4.4 Request/Response Transformation

```
┌──────────────────────────────────────────────────────┐
│  Client Request                                       │
│  GET /api/v2/users/12345                             │
│  Accept: application/json                            │
│  Authorization: Bearer token123                       │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  API Gateway Transformations                         │
│  ┌────────────────────────────────────────────────┐ │
│  │ Request Transformation                         │ │
│  │ • Remove /api/v2 prefix                        │ │
│  │ • Map /users → /internal/user-service          │ │
│  │ • Add correlation ID                           │ │
│  │ • Add gateway metadata                         │ │
│  │ • Convert REST → gRPC (if needed)              │ │
│  └────────────────────────────────────────────────┘ │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  Backend Request                                      │
│  POST /internal/user-service/get-user                │
│  Content-Type: application/grpc                      │
│  X-User-ID: 12345                                    │
│  X-Correlation-ID: abc-123-def                       │
│  X-Gateway-Version: 2.1.0                            │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
    [Backend Service]
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  Backend Response                                     │
│  {                                                    │
│    "userId": 12345,                                  │
│    "internalFields": {...},      ← Remove            │
│    "debugInfo": {...},           ← Remove            │
│    "name": "John Doe",                               │
│    "email": "john@example.com"                       │
│  }                                                    │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  API Gateway Transformations                         │
│  ┌────────────────────────────────────────────────┐ │
│  │ Response Transformation                        │ │
│  │ • Remove internal fields                       │ │
│  │ • Add HATEOAS links                            │ │
│  │ • Format dates (ISO 8601)                      │ │
│  │ • Add cache headers                            │ │
│  │ • Add CORS headers                             │ │
│  └────────────────────────────────────────────────┘ │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  Client Response                                      │
│  200 OK                                              │
│  Cache-Control: max-age=300                          │
│  Access-Control-Allow-Origin: *                      │
│  {                                                    │
│    "id": 12345,                                      │
│    "name": "John Doe",                               │
│    "email": "john@example.com",                      │
│    "_links": {                                       │
│      "self": "/api/v2/users/12345",                 │
│      "orders": "/api/v2/users/12345/orders"         │
│    }                                                 │
│  }                                                    │
└──────────────────────────────────────────────────────┘
```

### 4.5 Caching

```
┌─────────────────────────────────────────────────────┐
│              API Gateway Cache                      │
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │         Cache Layer (Redis/Memcached)       │  │
│  │                                             │  │
│  │  Key: GET:/users/123                        │  │
│  │  Value: {"id":123,"name":"John"}            │  │
│  │  TTL: 300 seconds                           │  │
│  │  Hits: 1,542 | Misses: 23                   │  │
│  └─────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘

Request Flow with Cache:

1. Cache Hit:
   Client ──▶ Gateway ──▶ Cache ──▶ Return Cached
                          (5ms)

2. Cache Miss:
   Client ──▶ Gateway ──▶ Cache (Miss) ──▶ Backend (50ms)
                                            │
                                            ▼
              Gateway ◀──── Store in Cache ◀─┘
                │
                ▼
              Client
```

**Cache Strategies:**

| Strategy | Description | Invalidation | Use Case |
|----------|-------------|--------------|----------|
| Time-based (TTL) | Cache expires after duration | Automatic | Static/semi-static data |
| Event-based | Invalidate on specific events | Manual/Event | Data with known updates |
| Cache-Aside | Load on miss, cache explicitly | TTL + Manual | Read-heavy workloads |
| Write-Through | Write to cache and DB together | Synchronous | Write + Read operations |
| Write-Behind | Write to cache, async to DB | Asynchronous | Write-heavy workloads |

### 4.6 SSL/TLS Termination

```
┌──────────┐
│  Client  │
└─────┬────┘
      │ HTTPS (TLS 1.3)
      │ Encrypted Traffic
      ▼
┌──────────────────────────────────────┐
│      API Gateway                     │
│                                      │
│  ┌────────────────────────────────┐ │
│  │   SSL/TLS Termination          │ │
│  │   • Decrypt incoming traffic   │ │
│  │   • Validate certificates      │ │
│  │   • Handle TLS handshake       │ │
│  │   • Offload encryption CPU     │ │
│  └────────────────────────────────┘ │
└──────────┬───────────────────────────┘
           │ HTTP (Plain)
           │ Internal Network (Trusted)
           ▼
    ┌──────────────┐
    │   Backend    │
    │   Services   │
    │ (No SSL)     │
    └──────────────┘
```

**Benefits:**

- **Performance**: Backend services don't handle encryption overhead
- **Centralized Management**: Single point for certificate management
- **Simplified Backend**: Services focus on business logic
- **Protocol Flexibility**: Can use different protocols internally

### 4.7 Monitoring & Logging

```
┌─────────────────────────────────────────────────────┐
│         API Gateway Observability                   │
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │            Request Logging                  │  │
│  │  • Request ID: abc-123                      │  │
│  │  • Method: GET                              │  │
│  │  • Path: /users/123                         │  │
│  │  • Client IP: 203.0.113.42                  │  │
│  │  • User Agent: Mobile/iOS                   │  │
│  │  • Response Status: 200                     │  │
│  │  • Latency: 45ms                            │  │
│  │  • Upstream: user-service-2                 │  │
│  └─────────────────────────────────────────────┘  │
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │            Metrics Collection               │  │
│  │  • Request Rate: 10,000 req/sec             │  │
│  │  • Error Rate: 0.05%                        │  │
│  │  • P50 Latency: 35ms                        │  │
│  │  • P95 Latency: 120ms                       │  │
│  │  • P99 Latency: 250ms                       │  │
│  │  • Cache Hit Rate: 75%                      │  │
│  └─────────────────────────────────────────────┘  │
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │         Distributed Tracing                 │  │
│  │  Trace ID: xyz-789                          │  │
│  │  ├─ API Gateway: 5ms                        │  │
│  │  ├─ User Service: 25ms                      │  │
│  │  │  ├─ Cache Check: 2ms                     │  │
│  │  │  └─ DB Query: 20ms                       │  │
│  │  └─ Order Service: 15ms                     │  │
│  │     └─ DB Query: 12ms                       │  │
│  └─────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

## 5. API Gateway Patterns

### 5.1 Gateway Routing Pattern

Simple request routing based on URL path or headers.

```
┌─────────────────────────────────────┐
│         API Gateway                 │
│                                     │
│  Routing Rules:                     │
│  /users/*     → User Service        │
│  /orders/*    → Order Service       │
│  /products/*  → Product Service     │
│  /payments/*  → Payment Service     │
└──────────┬──────────────────────────┘
           │
    ┌──────┴──────┬──────────┬────────────┐
    │             │          │            │
    ▼             ▼          ▼            ▼
┌────────┐  ┌─────────┐ ┌────────┐  ┌─────────┐
│  User  │  │  Order  │ │Product │  │ Payment │
│Service │  │ Service │ │Service │  │ Service │
└────────┘  └─────────┘ └────────┘  └─────────┘
```

**Use Case**: Microservices with clear domain boundaries

### 5.2 Gateway Aggregation Pattern

Aggregate multiple backend calls into a single response.

```
Client: GET /user/123/dashboard
           │
           ▼
┌──────────────────────────────────────┐
│      API Gateway                     │
│   Aggregation Logic                  │
└───┬────────┬────────┬────────────────┘
    │        │        │
    │ (1)    │ (2)    │ (3)
    │        │        │
    ▼        ▼        ▼
┌────────┐ ┌─────┐ ┌──────────┐
│  User  │ │Order│ │Recommend │
│Service │ │Svc  │ │ Service  │
└───┬────┘ └──┬──┘ └────┬─────┘
    │         │         │
    │ {user}  │{orders} │{items}
    └─────────┴─────────┘
              │
              ▼
    ┌──────────────────┐
    │   Combined:      │
    │   {              │
    │     user: {...}, │
    │     orders:[...],│
    │     items: [...] │
    │   }              │
    └──────────────────┘
              │
              ▼
          Client
```

**Use Case**: Dashboard pages, mobile apps requiring multiple data sources

### 5.3 Gateway Offloading Pattern

Offload cross-cutting concerns from backend services to the gateway.

```
┌─────────────────────────────────────────────────┐
│            API Gateway                          │
│  ┌───────────────────────────────────────────┐ │
│  │     Offloaded Responsibilities            │ │
│  │  • Authentication & Authorization         │ │
│  │  • SSL/TLS Termination                    │ │
│  │  • Rate Limiting & Throttling             │ │
│  │  • Request/Response Caching               │ │
│  │  • Logging & Monitoring                   │ │
│  │  • Request/Response Transformation        │ │
│  │  • CORS Handling                          │ │
│  │  • IP Whitelisting/Blacklisting           │ │
│  └───────────────────────────────────────────┘ │
└──────────────┬──────────────────────────────────┘
               │ Simplified Request
               ▼
        ┌──────────────┐
        │   Backend    │
        │   Services   │
        │              │
        │ Focus on:    │
        │ • Business   │
        │   Logic      │
        │ • Data       │
        │   Processing │
        └──────────────┘
```

**Use Case**: Simplifying backend services, consistent policy enforcement

## 6. Backend for Frontend (BFF) Pattern

The BFF pattern involves creating separate API Gateways tailored for different client types. Each BFF is optimized for specific client needs.

### 6.1 Architecture

```
┌──────────────┐         ┌─────────────────────────────┐
│              │         │      Web BFF Gateway        │
│ Web Browser  │────────▶│  • Full HTML responses      │
│              │         │  • Large payloads OK        │
│              │         │  • Rich data structures     │
└──────────────┘         └───────────┬─────────────────┘
                                     │
┌──────────────┐         ┌───────────┼─────────────────┐
│              │         │    Mobile BFF Gateway       │
│ Mobile App   │────────▶│  • Minimal payloads         │
│ (iOS/Android)│         │  • Optimized for bandwidth  │
│              │         │  • Aggregated responses     │
└──────────────┘         └───────────┬─────────────────┘
                                     │
┌──────────────┐         ┌───────────┼─────────────────┐
│              │         │   Partner BFF Gateway       │
│ 3rd Party    │────────▶│  • API versioning           │
│ Integration  │         │  • Rate limiting per key    │
│              │         │  • Stricter validation      │
└──────────────┘         └───────────┬─────────────────┘
                                     │
                         ┌───────────┴─────────────────┐
                         │                             │
                         ▼                             ▼
                  ┌────────────┐              ┌────────────┐
                  │  Shared    │              │   Domain   │
                  │  Services  │              │  Services  │
                  │            │              │            │
                  │ • User     │              │ • Orders   │
                  │ • Auth     │              │ • Products │
                  │ • Catalog  │              │ • Payments │
                  └────────────┘              └────────────┘
```

### 6.2 BFF Comparison

| Aspect | Web BFF | Mobile BFF | Partner BFF |
|--------|---------|------------|-------------|
| **Payload Size** | Large (verbose) | Small (minimal) | Medium (documented) |
| **Response Format** | Nested objects | Flat structures | Versioned schemas |
| **Aggregation** | Moderate | Heavy | Minimal |
| **Caching** | Browser-friendly | Aggressive | Conservative |
| **Authentication** | Session cookies | JWT tokens | API keys + OAuth |
| **Rate Limiting** | Lenient | Moderate | Strict per partner |
| **Documentation** | Internal | Internal | OpenAPI/Swagger |
| **Versioning** | Flexible | Flexible | Strict SLA |

### 6.3 Example: Mobile vs Web Response

**Web BFF Response** (verbose, rich):

```json
{
  "user": {
    "id": 123,
    "firstName": "John",
    "lastName": "Doe",
    "email": "john@example.com",
    "address": {
      "street": "123 Main St",
      "city": "Springfield",
      "state": "IL",
      "zip": "62701",
      "country": "USA"
    },
    "preferences": {
      "theme": "dark",
      "language": "en",
      "notifications": true
    }
  },
  "orders": [
    {
      "id": 456,
      "date": "2024-01-15T10:30:00Z",
      "items": [...],
      "shipping": {...},
      "billing": {...}
    }
  ],
  "_links": {...}
}
```

**Mobile BFF Response** (minimal, optimized):

```json
{
  "userId": 123,
  "name": "John Doe",
  "orders": [
    {
      "id": 456,
      "date": "2024-01-15",
      "total": 99.99,
      "status": "shipped"
    }
  ]
}
```

### 6.4 BFF Benefits

- **Client Optimization**: Each BFF optimized for specific client constraints
- **Parallel Development**: Frontend and BFF teams can iterate independently
- **Reduced Coupling**: Changes to one client don't affect others
- **Easier Testing**: Each BFF can be tested in isolation
- **Performance**: Tailored responses reduce over-fetching and under-fetching

### 6.5 BFF Drawbacks

- **Code Duplication**: Similar logic across multiple BFFs
- **Operational Overhead**: More services to deploy and monitor
- **Consistency Challenges**: Ensuring consistent behavior across BFFs
- **Resource Cost**: Multiple gateway instances increase infrastructure cost

## 7. API Gateway vs Load Balancer vs Reverse Proxy

| Feature | API Gateway | Load Balancer | Reverse Proxy |
|---------|-------------|---------------|---------------|
| **Primary Purpose** | API management & routing | Traffic distribution | Request forwarding |
| **OSI Layer** | Layer 7 (Application) | Layer 4 (Transport) or Layer 7 | Layer 7 (Application) |
| **Routing Logic** | Complex (path, header, method) | Simple (IP, port) | Moderate (host, path) |
| **Protocol Translation** | ✓ (HTTP↔gRPC, REST↔SOAP) | ✗ | Limited |
| **Authentication** | ✓ (JWT, OAuth, API keys) | ✗ | Limited |
| **Rate Limiting** | ✓ (per user/key/IP) | Basic (per connection) | Basic |
| **Request Aggregation** | ✓ | ✗ | ✗ |
| **Response Transformation** | ✓ | ✗ | ✗ |
| **Caching** | ✓ (application-level) | ✗ | ✓ (HTTP caching) |
| **API Versioning** | ✓ | ✗ | Limited |
| **Analytics & Monitoring** | Advanced (API metrics) | Basic (health checks) | Moderate (access logs) |
| **Circuit Breaking** | ✓ | ✗ | Limited |
| **Service Discovery** | ✓ | Limited | Limited |
| **SSL Termination** | ✓ | ✓ | ✓ |
| **Latency** | Higher (more processing) | Lowest | Low |
| **Use Case** | Microservices, public APIs | High-traffic websites | Web servers, CDN origin |
| **Examples** | Kong, AWS API Gateway, Apigee | AWS ELB, NGINX, HAProxy | NGINX, Apache, Envoy |

### 7.1 When to Use What

```
┌─────────────────────────────────────────────────────────┐
│  Client Request                                         │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
          ┌─────────────────────┐
          │   Load Balancer     │  ← Distribute traffic
          │   (Layer 4)         │    across gateways
          └─────────┬───────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼                       ▼
┌────────────────┐      ┌────────────────┐
│  API Gateway   │      │  API Gateway   │  ← API management,
│   Instance 1   │      │   Instance 2   │    authentication,
└───────┬────────┘      └───────┬────────┘    routing
        │                       │
        └───────────┬───────────┘
                    │
            ┌───────┴────────┐
            │                │
            ▼                ▼
    ┌──────────────┐  ┌──────────────┐
    │Service Pool A│  │Service Pool B│
    └──────┬───────┘  └──────┬───────┘
           │                 │
    ┌──────┴──┐       ┌──────┴──┐
    ▼         ▼       ▼         ▼
  ┌───┐     ┌───┐   ┌───┐     ┌───┐
  │S1 │     │S2 │   │S3 │     │S4 │  ← Backend services
  └───┘     └───┘   └───┘     └───┘
```

**Decision Matrix:**

- **Use Load Balancer**: When you need to distribute traffic across multiple identical instances
- **Use Reverse Proxy**: When you need basic request forwarding and SSL termination
- **Use API Gateway**: When you need API management, authentication, rate limiting, and request composition
- **Use All Three**: Large-scale architectures often use all three in combination

## 8. Implementation Approaches

### 8.1 Self-Hosted Solutions

#### 8.1.1 Kong

```
┌─────────────────────────────────────────────────────┐
│                Kong API Gateway                     │
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │            Core Components                  │  │
│  │  • NGINX-based proxy                        │  │
│  │  • Lua plugin system                        │  │
│  │  • PostgreSQL/Cassandra storage             │  │
│  └─────────────────────────────────────────────┘  │
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │            Popular Plugins                  │  │
│  │  • Authentication (JWT, OAuth2, Basic)      │  │
│  │  • Rate Limiting (multiple strategies)      │  │
│  │  • Request/Response Transformation          │  │
│  │  • CORS, ACL, IP Restriction                │  │
│  │  • Logging (file, syslog, HTTP)             │  │
│  │  • Monitoring (Prometheus, Datadog)         │  │
│  └─────────────────────────────────────────────┘  │
│                                                     │
│  Admin API: localhost:8001                         │
│  Proxy: localhost:8000                             │
└─────────────────────────────────────────────────────┘
```

**Strengths:**

- Open-source with commercial support
- Rich plugin ecosystem (50+ official plugins)
- High performance (NGINX-based)
- Kubernetes-native (Kong Ingress Controller)
- Declarative configuration

**Weaknesses:**

- Database dependency adds complexity
- Lua learning curve for custom plugins
- Limited built-in analytics

#### 8.1.2 NGINX Plus

```
┌─────────────────────────────────────────────────────┐
│              NGINX Plus API Gateway                 │
│                                                     │
│  Core Features:                                     │
│  • High-performance reverse proxy                  │
│  • Advanced load balancing                         │
│  • SSL/TLS termination                             │
│  • HTTP/2, gRPC support                            │
│  • Dynamic reconfiguration                         │
│  • Active health checks                            │
│  • JWT authentication                              │
│  • Rate limiting                                    │
│  • Content caching                                  │
│                                                     │
│  Configuration: nginx.conf                         │
│  API: /api/                                        │
└─────────────────────────────────────────────────────┘
```

**Strengths:**

- Extremely high performance
- Battle-tested reliability
- Low resource footprint
- Flexible configuration
- Commercial support available

**Weaknesses:**

- Configuration complexity (not developer-friendly)
- Limited API management features out-of-box
- Requires NGINX Plus for advanced features

#### 8.1.3 Spring Cloud Gateway

```
┌─────────────────────────────────────────────────────┐
│         Spring Cloud Gateway (Java/Spring)          │
│                                                     │
│  Built on:                                          │
│  • Spring WebFlux (reactive)                       │
│  • Project Reactor                                  │
│  • Netty                                            │
│                                                     │
│  Features:                                          │
│  • Route predicates (path, method, header)         │
│  • Gateway filters (rate limit, retry, circuit)    │
│  • Integration with Spring ecosystem               │
│  • Eureka service discovery                        │
│  • Config Server integration                       │
│                                                     │
│  Configuration: YAML or Java DSL                   │
└─────────────────────────────────────────────────────┘
```

**Strengths:**

- Native Java/Spring integration
- Reactive, non-blocking architecture
- Easy for Java developers
- Strong Spring ecosystem integration
- Built-in [Circuit Breaker](../resilience/circuit-breaker.md) support

**Weaknesses:**

- JVM overhead
- Limited language support (Java-only)
- Less mature than Kong/NGINX

### 8.2 Cloud-Managed Solutions

#### 8.2.1 AWS API Gateway

```
┌─────────────────────────────────────────────────────┐
│           AWS API Gateway                           │
│                                                     │
│  Types:                                             │
│  ┌─────────────────────────────────────────────┐  │
│  │  REST API                                   │  │
│  │  • Full API management features             │  │
│  │  • Request/response transformation          │  │
│  │  • SDK generation                           │  │
│  └─────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────┐  │
│  │  HTTP API                                   │  │
│  │  • Lower cost (70% cheaper)                 │  │
│  │  • Lower latency                            │  │
│  │  • Simpler feature set                      │  │
│  └─────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────┐  │
│  │  WebSocket API                              │  │
│  │  • Real-time bidirectional communication    │  │
│  │  • Persistent connections                   │  │
│  └─────────────────────────────────────────────┘  │
│                                                     │
│  Integrations:                                      │
│  • Lambda functions                                │
│  • HTTP endpoints                                  │
│  • AWS services (DynamoDB, SQS, S3)                │
│  • VPC Link (private resources)                    │
│                                                     │
│  Features:                                          │
│  • Cognito authentication                          │
│  • WAF integration                                 │
│  • CloudWatch logging                              │
│  • X-Ray tracing                                   │
│  • Request throttling                              │
│  • API keys & usage plans                          │
└─────────────────────────────────────────────────────┘
```

**Strengths:**

- Fully managed (no infrastructure)
- Native AWS integration
- Auto-scaling
- Pay-per-use pricing
- Built-in monitoring

**Weaknesses:**

- AWS lock-in
- Cold start latency (with Lambda)
- Limited customization
- Cost can escalate at high volume

#### 8.2.2 Azure API Management

```
┌─────────────────────────────────────────────────────┐
│          Azure API Management                       │
│                                                     │
│  Components:                                        │
│  • API Gateway (request handling)                  │
│  • Management Plane (configuration)                │
│  • Developer Portal (documentation)                │
│  • Analytics (insights)                            │
│                                                     │
│  Features:                                          │
│  • Multi-protocol support (REST, SOAP, GraphQL)    │
│  • API versioning & revisions                      │
│  • Policy-based transformations                    │
│  • OAuth 2.0, OpenID Connect                       │
│  • Rate limiting & quotas                          │
│  • Response caching                                │
│  • Mock responses                                  │
│  • Self-hosted gateway option                      │
│                                                     │
│  Tiers: Developer, Basic, Standard, Premium        │
└─────────────────────────────────────────────────────┘
```

**Strengths:**

- Comprehensive API management platform
- Developer portal out-of-box
- Hybrid deployment (cloud + on-premises)
- Rich policy engine
- Strong enterprise features

**Weaknesses:**

- Azure lock-in
- Higher cost than AWS API Gateway
- Complex pricing model

#### 8.2.3 Google Cloud API Gateway

```
┌─────────────────────────────────────────────────────┐
│       Google Cloud API Gateway                      │
│                                                     │
│  • OpenAPI/Swagger specification-based             │
│  • Fully managed                                    │
│  • Cloud Endpoints backend                         │
│  • Cloud Run, Cloud Functions, App Engine support  │
│  • Authentication: API keys, Firebase, Auth0       │
│  • Cloud Armor integration (DDoS protection)       │
│  • Cloud Logging & Monitoring                      │
│                                                     │
│  Pricing: Per million API calls                    │
└─────────────────────────────────────────────────────┘
```

**Strengths:**

- Simple, developer-friendly
- OpenAPI-first approach
- Good GCP integration
- Competitive pricing

**Weaknesses:**

- GCP lock-in
- Limited features vs Azure/AWS
- Newer offering (less mature)

### 8.3 Service Mesh Integration

#### 8.3.1 Istio + Envoy

```
┌─────────────────────────────────────────────────────┐
│         Service Mesh (Istio) + API Gateway          │
│                                                     │
│  External Traffic:                                  │
│  Client → Istio Ingress Gateway (Envoy)            │
│                    │                                │
│                    ▼                                │
│         ┌──────────────────────┐                   │
│         │   Service Mesh       │                   │
│         │   (Internal Traffic) │                   │
│         │                      │                   │
│         │   Service A ←→ Envoy │                   │
│         │   Service B ←→ Envoy │                   │
│         │   Service C ←→ Envoy │                   │
│         └──────────────────────┘                   │
│                                                     │
│  Features:                                          │
│  • mTLS between services                           │
│  • Advanced routing (canary, A/B)                  │
│  • Automatic retries & circuit breaking            │
│  • Distributed tracing                             │
│  • Traffic mirroring                               │
└─────────────────────────────────────────────────────┘
```

**Use Case**: Kubernetes environments needing both API Gateway and service-to-service communication management

### 8.4 Comparison Matrix

| Solution | Deployment | Cost | Performance | Features | Learning Curve |
|----------|-----------|------|-------------|----------|----------------|
| **Kong** | Self-hosted | Low | Very High | Excellent | Medium |
| **NGINX Plus** | Self-hosted | Medium | Highest | Good | High |
| **Spring Cloud Gateway** | Self-hosted | Low | High | Good | Medium |
| **AWS API Gateway** | Managed | Pay-per-use | High | Excellent | Low |
| **Azure API Management** | Managed/Hybrid | High | High | Excellent | Medium |
| **GCP API Gateway** | Managed | Pay-per-use | High | Good | Low |
| **Istio/Envoy** | Self-hosted | Medium | Very High | Excellent | Very High |

## 9. Trade-offs & Considerations

### 9.1 Benefits

```
✓ Simplified Client Integration
  • Single endpoint for all services
  • Consistent API contract
  • Protocol abstraction

✓ Centralized Security
  • Authentication/authorization in one place
  • Consistent security policies
  • Easier compliance auditing

✓ Reduced Service Coupling
  • Clients don't depend on service topology
  • Services can be refactored independently
  • Easier service versioning

✓ Cross-cutting Concerns
  • No duplication of common logic
  • Consistent behavior across services
  • Easier to update policies

✓ Improved Observability
  • Centralized logging and monitoring
  • Request tracing across services
  • API analytics in one place

✓ Performance Optimization
  • Response caching
  • Request aggregation reduces round trips
  • Protocol optimization (HTTP/2, gRPC)
```

### 9.2 Drawbacks

```
✗ Single Point of Failure
  • Gateway downtime affects all services
  • Requires high availability setup
  • Mitigation: Multiple instances, load balancing

✗ Additional Latency
  • Extra network hop
  • Processing overhead (auth, transformation)
  • Typical: 5-20ms additional latency

✗ Potential Bottleneck
  • All traffic flows through gateway
  • Must be scaled appropriately
  • Can become performance constraint

✗ Increased Complexity
  • Another component to deploy and manage
  • Configuration management challenges
  • Requires operational expertise

✗ Development Bottleneck
  • Changes may require gateway updates
  • Potential single team ownership issue
  • Can slow down feature delivery

✗ Vendor Lock-in (Managed Solutions)
  • Cloud-specific features
  • Migration complexity
  • Proprietary configuration formats
```

### 9.3 Single Point of Failure Mitigation

```
┌─────────────────────────────────────────────────────┐
│              High Availability Setup                │
│                                                     │
│                  ┌──────────────┐                  │
│      ┌──────────▶│Load Balancer │◀──────────┐     │
│      │           │  (Layer 4)   │           │     │
│      │           └──────────────┘           │     │
│      │                                      │     │
│      ▼                                      ▼     │
│ ┌──────────┐                          ┌──────────┐│
│ │   API    │                          │   API    ││
│ │ Gateway  │                          │ Gateway  ││
│ │Instance 1│                          │Instance 2││
│ │(Active)  │                          │(Active)  ││
│ └────┬─────┘                          └─────┬────┘│
│      │                                      │     │
│      │      ┌──────────────────┐           │     │
│      └─────▶│  Shared Cache    │◀──────────┘     │
│              │  (Redis Cluster) │                 │
│              └──────────────────┘                 │
│                                                     │
│  • Health checks (every 5s)                        │
│  • Automatic failover                              │
│  • Graceful shutdown                               │
│  • Circuit breakers to backends                    │
│  • Rate limiting with distributed state            │
└─────────────────────────────────────────────────────┘
```

**Strategies:**

1. **Multiple Instances**: Deploy gateway across availability zones
2. **Health Checks**: Continuous monitoring with automatic failover
3. **Circuit Breakers**: Prevent cascading failures (see [Circuit Breaker pattern](../resilience/circuit-breaker.md))
4. **Graceful Degradation**: Serve stale cache when backends fail
5. **Chaos Engineering**: Regularly test failure scenarios

### 9.4 Performance Considerations

```
Latency Breakdown:

┌────────────────────────────────────────┐
│ Total Request Time: 100ms              │
│                                        │
│ ┌────────────────────────────────────┐ │
│ │ API Gateway Overhead: 15ms         │ │
│ │ ├─ SSL/TLS: 5ms                    │ │
│ │ ├─ Authentication: 3ms             │ │
│ │ ├─ Rate Limiting: 1ms              │ │
│ │ ├─ Routing Decision: 1ms           │ │
│ │ ├─ Request Transform: 2ms          │ │
│ │ └─ Logging: 3ms                    │ │
│ └────────────────────────────────────┘ │
│                                        │
│ ┌────────────────────────────────────┐ │
│ │ Network: 10ms                      │ │
│ │ Gateway → Backend                  │ │
│ └────────────────────────────────────┘ │
│                                        │
│ ┌────────────────────────────────────┐ │
│ │ Backend Processing: 75ms           │ │
│ │ (Business logic, database)         │ │
│ └────────────────────────────────────┘ │
└────────────────────────────────────────┘
```

**Optimization Strategies:**

| Strategy | Impact | Complexity |
|----------|--------|------------|
| Response Caching | High (avoid backend call) | Low |
| Connection Pooling | Medium (reduce connection overhead) | Low |
| Request Batching | High (reduce round trips) | Medium |
| Async Processing | Medium (non-blocking I/O) | Medium |
| Edge Deployment | High (reduce network latency) | High |
| Protocol Optimization (HTTP/2) | Medium (multiplexing) | Low |
| Compression | Medium (reduce bandwidth) | Low |

### 9.5 Scaling Strategies

```
Vertical Scaling:
┌──────────┐         ┌──────────┐
│ Gateway  │  ──▶    │ Gateway  │
│ 2 CPU    │         │ 8 CPU    │
│ 4GB RAM  │         │ 16GB RAM │
└──────────┘         └──────────┘
Pros: Simple, no architectural changes
Cons: Limited, expensive, single point of failure

Horizontal Scaling:
┌──────────┐         ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Gateway  │  ──▶    │ Gateway  │  │ Gateway  │  │ Gateway  │
│          │         │Instance 1│  │Instance 2│  │Instance 3│
└──────────┘         └──────────┘  └──────────┘  └──────────┘
Pros: Unlimited scale, redundancy, cost-effective
Cons: Requires load balancer, stateless design

Regional Distribution:
        ┌──────────────────────────────────┐
        │        Global Load Balancer      │
        │        (GeoDNS/Anycast)          │
        └──────┬───────────────────┬───────┘
               │                   │
        ┌──────▼──────┐     ┌──────▼──────┐
        │   US East   │     │  EU West    │
        │  Gateway    │     │  Gateway    │
        │  Cluster    │     │  Cluster    │
        └─────────────┘     └─────────────┘
Pros: Low latency worldwide, disaster recovery
Cons: Complex, consistency challenges, higher cost
```

## 10. Related Patterns

The API Gateway pattern works in conjunction with other architectural patterns:

### 10.1 Microservices Architecture

The API Gateway is a fundamental component of [microservices architectures](microservices.md), providing the entry point to a distributed system of services.

### 10.2 Circuit Breaker

API Gateways often implement the [Circuit Breaker pattern](../resilience/circuit-breaker.md) to prevent cascading failures when backend services are unhealthy.

```
┌─────────────────────────────────────┐
│      API Gateway                    │
│  ┌───────────────────────────────┐ │
│  │  Circuit Breaker               │ │
│  │  State: OPEN                   │ │
│  │  Failed Requests: 15/20        │ │
│  │  Last Failure: 2s ago          │ │
│  │                                │ │
│  │  Action: Return cached         │ │
│  │          response or error     │ │
│  └───────────────────────────────┘ │
└──────────────┬──────────────────────┘
               │
               ✗ (Don't call)
               │
        ┌──────▼────────┐
        │   Unhealthy   │
        │    Backend    │
        │   Service     │
        └───────────────┘
```

### 10.3 Event-Driven Architecture

API Gateways can integrate with [event-driven architectures](../event-driven/event-driven-architecture.md), publishing events when specific API calls are made.

### 10.4 Saga Pattern

For long-running transactions spanning multiple services, the API Gateway can orchestrate the [Saga pattern](../distributed-transactions/saga.md).

### 10.5 CQRS

API Gateways can route read queries to read-optimized services and write commands to write-optimized services in a CQRS architecture.

### 10.6 Strangler Fig

During migration from monolith to microservices, the API Gateway enables the Strangler Fig pattern by routing requests to either legacy or new services.

```
┌─────────────────────────────────────┐
│         API Gateway                 │
│  ┌───────────────────────────────┐ │
│  │  Routing Rules:               │ │
│  │  /new/feature → Microservice  │ │
│  │  /legacy/*    → Monolith      │ │
│  └───────────────────────────────┘ │
└──────────┬──────────────────────────┘
           │
    ┌──────┴───────┐
    │              │
    ▼              ▼
┌─────────┐  ┌──────────────┐
│  New    │  │   Legacy     │
│Micro    │  │   Monolith   │
│Services │  │ (Being phased│
│         │  │     out)     │
└─────────┘  └──────────────┘
```

### 10.7 Serverless Architecture

API Gateways are essential in [serverless architectures](serverless.md), routing requests to serverless functions (AWS Lambda, Azure Functions, etc.).

## 11. Real-World Examples

### 11.1 Netflix - Zuul

Netflix developed Zuul, their edge service that provides dynamic routing, monitoring, resiliency, and security.

```
┌─────────────────────────────────────────────────────┐
│             Netflix Zuul Architecture               │
│                                                     │
│  Client → [AWS ELB] → [Zuul] → Backend Services    │
│                                                     │
│  Zuul Features:                                     │
│  • Dynamic routing                                  │
│  • Traffic shaping                                  │
│  • Request authentication & security                │
│  • Insights & monitoring                            │
│  • Stress testing                                   │
│  • Canary testing                                   │
│  • Multiregion resiliency                           │
│                                                     │
│  Evolution:                                         │
│  Zuul 1 (Blocking I/O) → Zuul 2 (Async/Non-blocking)│
│                                                     │
│  Scale: Handles billions of requests per day       │
└─────────────────────────────────────────────────────┘
```

**Key Insights:**

- **Dynamic Routing**: Routes requests based on various criteria (user, device, region)
- **Chaos Engineering**: Built-in failure injection for testing
- **Multi-Region**: Handles failover between AWS regions
- **Monitoring**: Extensive metrics and distributed tracing

**Netflix Later Moved to Spring Cloud Gateway** for Reactive support and better performance.

### 11.2 Amazon - AWS API Gateway

Amazon uses API Gateway to expose AWS services and Lambda functions.

```
┌─────────────────────────────────────────────────────┐
│          AWS API Gateway Use Cases                  │
│                                                     │
│  1. Serverless Backends                            │
│     Client → API Gateway → Lambda → DynamoDB       │
│                                                     │
│  2. Legacy Modernization                           │
│     Mobile → API Gateway → { Lambda (new)          │
│                            { EC2 (legacy)          │
│                                                     │
│  3. Microservices Facade                           │
│     API Gateway → { User Service (ECS)             │
│                   { Order Service (EKS)            │
│                   { Payment Service (Lambda)       │
│                                                     │
│  4. Third-Party API Monetization                   │
│     Partner → API Gateway → Usage Plans → Billing  │
└─────────────────────────────────────────────────────┘
```

**Key Features Used:**

- **Lambda Integration**: Seamless serverless backend
- **API Keys & Usage Plans**: Monetization and rate limiting
- **Request Validation**: Schema validation before reaching backend
- **SDK Generation**: Auto-generated client SDKs

### 11.3 Uber - API Gateway for Microservices

Uber uses an API Gateway to manage thousands of microservices.

```
┌─────────────────────────────────────────────────────┐
│            Uber API Gateway                         │
│                                                     │
│  Challenges:                                        │
│  • 2000+ microservices                             │
│  • Multiple programming languages                  │
│  • High request volume                             │
│  • Low latency requirements                        │
│                                                     │
│  Solutions:                                         │
│  • Custom-built gateway (proprietary)              │
│  • Protocol translation (HTTP ↔ Thrift)            │
│  • Intelligent routing                             │
│  • Advanced load balancing                         │
│  • Real-time traffic shaping                       │
│  • Circuit breaking & retries                      │
│                                                     │
│  Results:                                           │
│  • Sub-millisecond gateway latency                 │
│  • Handles millions of requests per second         │
│  • Enables rapid service development               │
└─────────────────────────────────────────────────────┘
```

### 11.4 Spotify - API Gateway with GraphQL

Spotify uses API Gateway with GraphQL to provide a flexible API to clients.

```
┌─────────────────────────────────────────────────────┐
│         Spotify API Architecture                    │
│                                                     │
│  Client (Mobile/Web)                               │
│       │                                             │
│       ▼                                             │
│  ┌─────────────────┐                               │
│  │  API Gateway    │                               │
│  │  (GraphQL)      │                               │
│  └────────┬────────┘                               │
│           │                                         │
│    ┌──────┼──────┬──────┬──────┐                  │
│    ▼      ▼      ▼      ▼      ▼                  │
│  User  Playlist Track  Album  Artist              │
│  Svc    Svc     Svc    Svc    Svc                 │
│                                                     │
│  Benefits:                                          │
│  • Clients request exactly what they need          │
│  • Reduced over-fetching                           │
│  • Single round trip for complex queries           │
│  • Strongly typed schema                           │
└─────────────────────────────────────────────────────┘
```

### 11.5 Twitter - Edge API

Twitter uses an edge API layer to handle client requests.

```
┌─────────────────────────────────────────────────────┐
│           Twitter Edge API                          │
│                                                     │
│  Responsibilities:                                  │
│  • Authentication (OAuth)                          │
│  • Rate limiting (per user, per endpoint)          │
│  • Response caching                                │
│  • Request routing                                 │
│  • Protocol translation                            │
│  • A/B testing                                     │
│  • Feature flags                                   │
│                                                     │
│  Technologies:                                      │
│  • Custom Scala-based gateway                      │
│  • Finagle (RPC framework)                         │
│  • Distributed tracing with Zipkin                 │
│                                                     │
│  Scale:                                             │
│  • Handles hundreds of thousands of req/sec        │
│  • Sub-50ms latency at P99                         │
└─────────────────────────────────────────────────────┘
```

## 12. Interview Framework & Key Takeaways

### 12.1 When Discussing API Gateway in Interviews

**Start with the problem:**

- "In a microservices architecture with 50+ services, having clients call each service directly creates several issues: tight coupling, multiple round trips, duplicated security logic, and complex client code."

**Explain the solution:**

- "An API Gateway provides a single entry point that handles routing, authentication, rate limiting, and aggregation. It decouples clients from backend services."

**Discuss trade-offs:**

- "While it simplifies client integration and centralizes cross-cutting concerns, it introduces an additional network hop and can become a single point of failure. We mitigate this with multiple instances, health checks, and circuit breakers."

**Provide real examples:**

- "Netflix uses Zuul/Spring Cloud Gateway to route traffic to thousands of backend services. AWS API Gateway enables serverless architectures with Lambda."

### 12.2 Common Interview Questions

**Q: How does API Gateway differ from a Load Balancer?**

A: A Load Balancer operates at Layer 4 (TCP) or Layer 7 (HTTP) and distributes traffic across identical instances. An API Gateway operates at Layer 7 and provides API management features like authentication, rate limiting, request aggregation, and protocol translation. You often use both: a Load Balancer distributes traffic across multiple API Gateway instances.

**Q: How do you handle authentication in an API Gateway?**

A: The gateway validates tokens (JWT, OAuth) on every request, extracts user context, checks permissions, and forwards authenticated requests to backend services with user metadata in headers. Backend services trust the gateway and don't re-authenticate. This centralizes security logic and reduces duplication.

**Q: What happens if the API Gateway goes down?**

A: This is the single point of failure concern. Mitigation strategies include:

- Deploy multiple gateway instances behind a load balancer
- Use health checks with automatic failover
- Implement circuit breakers to prevent cascading failures
- Deploy across multiple availability zones
- Consider regional distribution for disaster recovery

**Q: How do you handle versioning with an API Gateway?**

A: Common approaches:

- URL-based: `/v1/users`, `/v2/users`
- Header-based: `Accept: application/vnd.api+json;version=2`
- Gateway routing: Route based on version to different backend services
- Gradual migration: Use gateway to route percentage of traffic to new version (canary deployment)

**Q: How does API Gateway impact latency?**

A: It adds 5-20ms overhead for authentication, routing, and transformation. This is typically acceptable because:

- Request aggregation reduces total round trips
- Caching can eliminate backend calls entirely
- The simplification of backend services often improves their performance
- Protocol optimization (HTTP/2, compression) can offset the overhead

**Q: When would you NOT use an API Gateway?**

A: Consider alternatives when:

- You have a simple monolithic application (over-engineering)
- Internal service-to-service communication (use service mesh instead)
- Ultra-low latency requirements where every millisecond matters
- Very small scale where operational complexity outweighs benefits

### 12.3 Key Takeaways

1. **API Gateway = Single Entry Point**: All external requests flow through the gateway, which routes to appropriate backend services.

2. **Cross-Cutting Concerns**: Centralizes authentication, rate limiting, logging, caching, and SSL termination, removing duplication from backend services.

3. **Request Aggregation**: Can combine multiple backend calls into a single client request, reducing round trips and improving mobile performance.

4. **Protocol Translation**: Bridges different protocols (REST ↔ gRPC, HTTP ↔ WebSocket), allowing backends to use optimal protocols.

5. **BFF Pattern**: Consider separate gateways for different client types (web, mobile, partners) to optimize for specific needs.

6. **Not Just a Reverse Proxy**: While it includes reverse proxy functionality, it provides much richer API management capabilities.

7. **Trade-off: Simplicity vs Complexity**: Simplifies client integration but adds operational complexity and potential single point of failure.

8. **Implementation Options**: Choose between self-hosted (Kong, NGINX), cloud-managed (AWS, Azure), or service mesh (Istio) based on requirements.

9. **Performance Considerations**: Adds latency but can improve overall performance through caching, aggregation, and connection pooling.

10. **Essential for Microservices**: In a microservices architecture with dozens or hundreds of services, an API Gateway is practically essential for manageable client integration.

11. **Security Boundary**: Acts as the security perimeter, authenticating all external requests before they reach internal services.

12. **Resilience Patterns**: Implement circuit breakers, retries, timeouts, and rate limiting to protect both the gateway and backend services.

13. **Observability**: Provides a centralized point for logging, monitoring, and distributed tracing across all API calls.

14. **Evolution Path**: Start simple with basic routing, then add features incrementally as needs grow (authentication → rate limiting → caching → aggregation).

15. **Complementary Patterns**: Works with Circuit Breaker, Microservices, Event-Driven Architecture, Saga Pattern, and Serverless to build robust distributed systems.

---

**Related Topics:**

- [Microservices Architecture](microservices.md)
- [Circuit Breaker Pattern](../resilience/circuit-breaker.md)
- [CAP Theorem](../fundamentals/cap-theorem.md) - Understanding distributed system trade-offs
- [Saga Pattern](../distributed-transactions/saga.md) - Distributed transactions
- [Event-Driven Architecture](../event-driven/event-driven-architecture.md)
- [Serverless Architecture](serverless.md)
- [Message Queues: RabbitMQ vs Kafka](../messaging/rabbitmq-vs-kafka.md)

**References:**

- "Building Microservices" by Sam Newman (Chapter on API Gateway)
- "System Design Interview" by Alex Xu (API Gateway section)
- Netflix Tech Blog: Zuul gateway
- AWS API Gateway Documentation
- Kong Gateway Documentation
- Martin Fowler: "Gateway Aggregation Pattern"
