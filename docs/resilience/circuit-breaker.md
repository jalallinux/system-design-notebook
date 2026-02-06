# Circuit Breaker Pattern

## 1. Introduction

The Circuit Breaker pattern is a design pattern used in modern distributed systems to prevent cascading failures and improve system resilience. Just as an electrical circuit breaker protects your home's electrical system from overload by cutting power when too much current flows, a software circuit breaker protects your application from making calls to services that are likely to fail.

**Origin and Evolution:**

The Circuit Breaker pattern was popularized by Michael Nygard in his influential 2007 book "Release It! Design and Deploy Production-Ready Software." Martin Fowler later expanded on this pattern in his 2014 blog post, making it a cornerstone of resilient [microservices architecture](../architecture/microservices.md).

**Why It Matters:**

In distributed systems, services depend on other services. When a downstream service becomes slow or unresponsive, upstream services can waste resources (threads, memory, connections) waiting for responses that may never come. This can lead to cascading failures where one service's problems ripple through the entire system, potentially taking down the entire application.

**Core Principle:**

Instead of repeatedly trying to call a failing service (which wastes resources and delays failure detection), a circuit breaker monitors for failures and "opens the circuit" to fail fast, giving the failing service time to recover while preventing resource exhaustion in the calling service.

## 2. Problem Statement

### The Cascading Failure Scenario

Consider an e-commerce application with the following service dependencies:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Frontend   │────▶│   Order      │────▶│  Inventory   │
│   Service    │     │   Service    │     │   Service    │
└──────────────┘     └──────────────┘     └──────────────┘
                              │
                              ▼
                     ┌──────────────┐     ┌──────────────┐
                     │   Payment    │────▶│   External   │
                     │   Service    │     │   Payment    │
                     └──────────────┘     │   Gateway    │
                                          └──────────────┘
```

**Without Circuit Breaker:**

1. The external payment gateway becomes unresponsive (network issue, overload, outage)
2. Payment Service keeps trying to reach it, with each request timing out after 30 seconds
3. Order Service threads are blocked waiting for Payment Service responses
4. Thread pool in Order Service becomes exhausted (all threads waiting)
5. Frontend Service requests to Order Service start timing out
6. Frontend Service threads become blocked
7. The entire application becomes unresponsive, even for operations that don't need the payment gateway

**Problems Without Circuit Breaker:**

- **Resource Exhaustion:** Threads, connections, and memory are tied up waiting for responses
- **Slow Failure Detection:** It takes many timeout cycles to realize a service is down
- **User Experience Degradation:** Users experience long waits before getting error messages
- **Wasted Compute:** CPU cycles spent on requests destined to fail
- **Difficulty in Recovery:** Even when the downstream service recovers, the upstream services may be so overloaded they can't take advantage of it

According to the [CAP theorem](../fundamentals/cap-theorem.md), during network partitions, we must choose between consistency and availability. Circuit breakers help us maintain availability by failing fast when consistency with a downstream service cannot be achieved.

## 3. The Three States

The Circuit Breaker pattern operates as a state machine with three states:

```
                    ┌─────────────────────────────────┐
                    │                                 │
                    │  Failure threshold reached      │
                    ▼                                 │
         ┌──────────────────┐                        │
         │                  │                        │
    ┌───▶│      CLOSED      │                        │
    │    │  (Normal Flow)   │                        │
    │    │                  │                        │
    │    └──────────────────┘                        │
    │                                                 │
    │    ┌──────────────────┐     Timeout expires    │
    │    │                  │    (try recovery)      │
    │    │       OPEN       │────────────────────────┤
    │    │   (Fail Fast)    │                        │
    │    │                  │                        │
    │    └──────────────────┘                        │
    │             │                                   │
    │             │                                   │
    │             ▼                                   │
    │    ┌──────────────────┐                        │
    │    │                  │                        │
    └────│    HALF-OPEN     │                        │
         │  (Test Recovery) │                        │
         │                  │                        │
         └──────────────────┘                        │
                  │                                   │
                  │ Success threshold not met         │
                  └───────────────────────────────────┘
```

### 3.1 CLOSED State (Normal Operation)

**Characteristics:**
- All requests pass through to the downstream service
- Circuit breaker monitors failures (timeouts, exceptions, 5xx errors)
- Maintains a counter or sliding window of recent failures
- This is the default and desired state

**Behavior:**
```
Client ───request──▶ Circuit Breaker ───request──▶ Service
       ◀──response─                    ◀─response──

Success: Increment success counter, reset failure counter
Failure: Increment failure counter
```

**State Transition:**
When the failure count or failure rate exceeds the configured threshold within a time window, the circuit transitions to OPEN.

**Example Threshold Configuration:**
- Fail after 5 consecutive failures, OR
- Fail when 50% of requests fail in a 10-second window, OR
- Fail when 10 failures occur in a 30-second sliding window

### 3.2 OPEN State (Failing Fast)

**Characteristics:**
- All requests fail immediately without calling the downstream service
- No resources are wasted on doomed requests
- A timeout timer starts (e.g., 60 seconds)
- The failing service gets time to recover without additional load

**Behavior:**
```
Client ───request──▶ Circuit Breaker ──╳──▶ Service (not called)
       ◀───error───

Immediate failure with CircuitBreakerOpenException
```

**State Transition:**
After the timeout period expires, the circuit transitions to HALF-OPEN to test if the downstream service has recovered.

**Why Fail Fast?**
- Immediate user feedback (no waiting for timeouts)
- Resources freed for other operations
- Reduced load on failing service
- Allows for fallback mechanisms

### 3.3 HALF-OPEN State (Testing Recovery)

**Characteristics:**
- A limited number of test requests are allowed through
- Acts as a "canary" to check if the service has recovered
- Most requests still fail fast during this period
- Brief transitional state

**Behavior:**
```
Test Request 1 ───request──▶ Circuit Breaker ───request──▶ Service
               ◀──response─                    ◀─response──

Test Request 2 ───request──▶ Circuit Breaker ───request──▶ Service
               ◀──response─                    ◀─response──

Other Requests ───request──▶ Circuit Breaker ──╳──▶ Service
               ◀───error───                     (still blocked)
```

**State Transitions:**

1. **To CLOSED (Success Path):**
   - If the test requests succeed (e.g., 3 consecutive successes)
   - Circuit fully closes and normal operation resumes
   - Counters are reset

2. **To OPEN (Failure Path):**
   - If test requests fail
   - Circuit immediately reopens
   - Timeout timer restarts
   - The service needs more recovery time

**Configuration Examples:**
- Allow 3 test requests in HALF-OPEN
- If 3/3 succeed → CLOSED
- If any fail → OPEN

## 4. How It Works

### 4.1 Success Path (Normal Operation)

```
┌────────┐         ┌─────────────────┐         ┌─────────┐
│ Client │         │ Circuit Breaker │         │ Service │
└───┬────┘         └────────┬────────┘         └────┬────┘
    │                       │                       │
    │  1. Request           │                       │
    ├──────────────────────▶│                       │
    │                       │                       │
    │                       │ 2. Check State        │
    │                       │    (CLOSED)           │
    │                       │                       │
    │                       │ 3. Forward Request    │
    │                       ├──────────────────────▶│
    │                       │                       │
    │                       │ 4. Service Response   │
    │                       │◀──────────────────────┤
    │                       │                       │
    │                       │ 5. Record Success     │
    │                       │    Reset Fail Counter │
    │                       │                       │
    │  6. Return Response   │                       │
    │◀──────────────────────┤                       │
    │                       │                       │
```

### 4.2 Failure Path (Detecting Failures)

```
┌────────┐         ┌─────────────────┐         ┌─────────┐
│ Client │         │ Circuit Breaker │         │ Service │
└───┬────┘         └────────┬────────┘         └────┬────┘
    │                       │                       │
    │  1. Request           │                       │
    ├──────────────────────▶│                       │
    │                       │                       │
    │                       │ 2. Check State        │
    │                       │    (CLOSED)           │
    │                       │                       │
    │                       │ 3. Forward Request    │
    │                       ├──────────────────────▶│
    │                       │                       │
    │                       │ 4. Timeout/Error      │
    │                       │        ╳              │
    │                       │                       │
    │                       │ 5. Increment Failures │
    │                       │    Check Threshold    │
    │                       │    (5 failures)       │
    │                       │    → Trip to OPEN     │
    │                       │                       │
    │  6. Return Error      │                       │
    │◀──────────────────────┤                       │
    │                       │                       │
    │                       │                       │
    │  7. Next Request      │                       │
    ├──────────────────────▶│                       │
    │                       │                       │
    │                       │ 8. Check State (OPEN) │
    │  9. Fail Fast         │    ╳ Don't call       │
    │◀──────────────────────┤      service          │
    │                       │                       │
```

### 4.3 Recovery Path (Service Healing)

```
┌────────┐         ┌─────────────────┐         ┌─────────┐
│ Client │         │ Circuit Breaker │         │ Service │
└───┬────┘         └────────┬────────┘         └────┬────┘
    │                       │                       │
    │                       │ [Circuit is OPEN]     │
    │                       │ [60s timeout expires] │
    │                       │                       │
    │                       │ Transition to         │
    │                       │ HALF-OPEN             │
    │                       │                       │
    │  1. Test Request      │                       │
    ├──────────────────────▶│                       │
    │                       │                       │
    │                       │ 2. Check State        │
    │                       │    (HALF-OPEN)        │
    │                       │    Allow test request │
    │                       │                       │
    │                       │ 3. Forward Test Req   │
    │                       ├──────────────────────▶│
    │                       │                       │
    │                       │ 4. Success Response   │
    │                       │◀──────────────────────┤
    │                       │                       │
    │                       │ 5. Record Success     │
    │                       │    (1 of 3 needed)    │
    │                       │                       │
    │  6. Return Response   │                       │
    │◀──────────────────────┤                       │
    │                       │                       │
    │  [More test requests  │                       │
    │   succeed...]         │                       │
    │                       │                       │
    │                       │ After 3 successes:    │
    │                       │ Transition to CLOSED  │
    │                       │ Resume normal ops     │
    │                       │                       │
```

## 5. Implementation Details

### 5.1 Key Configuration Parameters

```
┌─────────────────────────────────────────────────────────┐
│            Circuit Breaker Configuration                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Failure Threshold                                      │
│  ├─ Count Based: 5 consecutive failures                │
│  ├─ Rate Based: 50% failure rate                       │
│  └─ Time Window: 10 seconds                            │
│                                                         │
│  Open State Duration                                    │
│  └─ Sleep Window: 60 seconds                           │
│                                                         │
│  Half-Open Test Requests                               │
│  └─ Permitted Calls: 3 requests                        │
│                                                         │
│  Success Threshold (Half-Open → Closed)                │
│  └─ Required Successes: 3 out of 3                     │
│                                                         │
│  What Counts as Failure?                               │
│  ├─ Exceptions: IOException, TimeoutException          │
│  ├─ HTTP Status: 500, 502, 503, 504                   │
│  ├─ Timeouts: > 3 seconds                             │
│  └─ Custom Predicates: Based on business logic         │
│                                                         │
│  What Does NOT Count as Failure?                       │
│  ├─ HTTP 4xx (client errors)                          │
│  ├─ Expected exceptions (ValidationException)          │
│  └─ Business logic failures                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 5.2 Sliding Window Implementation

There are two common approaches to tracking failures:

**Count-Based Sliding Window:**
```
Time: ──────────────────────────────────────▶
       [S] [F] [F] [S] [F] [F] [F]

Window Size: Last 10 requests
Current: 5 failures out of 7 requests = 71% failure rate
Threshold: 50%
Action: TRIP TO OPEN
```

**Time-Based Sliding Window:**
```
Time: 0s────10s────20s────30s────40s────50s────60s────▶
      [SSF] [FFF] [FSF] [FFS] [SSS] [SSS] [FFF]
                                    ↑
                                    Now (50s)

Window: Last 30 seconds
Requests in window (20s-50s): [FFS][SSS][SSS] = 2 failures / 9 requests
Failure Rate: 22%
Threshold: 50%
Action: REMAIN CLOSED
```

### 5.3 Pseudocode Implementation

```java
class CircuitBreaker {
    enum State { CLOSED, OPEN, HALF_OPEN }

    private State state = CLOSED
    private int failureCount = 0
    private int successCount = 0
    private long openedAt = 0

    // Configuration
    private final int failureThreshold = 5
    private final long openDuration = 60000  // 60 seconds
    private final int halfOpenSuccessThreshold = 3

    function call(operation) {
        if (state == OPEN) {
            if (currentTime() - openedAt >= openDuration) {
                state = HALF_OPEN
                successCount = 0
            } else {
                throw CircuitBreakerOpenException()
            }
        }

        try {
            result = operation.execute()
            onSuccess()
            return result
        } catch (Exception e) {
            onFailure()
            throw e
        }
    }

    function onSuccess() {
        failureCount = 0

        if (state == HALF_OPEN) {
            successCount++
            if (successCount >= halfOpenSuccessThreshold) {
                state = CLOSED
                successCount = 0
            }
        }
    }

    function onFailure() {
        successCount = 0
        failureCount++

        if (state == HALF_OPEN) {
            state = OPEN
            openedAt = currentTime()
        } else if (failureCount >= failureThreshold) {
            state = OPEN
            openedAt = currentTime()
        }
    }
}
```

### 5.4 Thread Safety Considerations

Circuit breakers must be thread-safe as they're shared across multiple threads:

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Thread 1   │    │  Thread 2   │    │  Thread 3   │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                          ▼
              ┌────────────────────────┐
              │   Circuit Breaker      │
              │   (Shared Instance)    │
              │                        │
              │  - Atomic counters     │
              │  - Synchronized state  │
              │  - Lock-free if possible
              └────────────────────────┘
```

**Implementation Requirements:**
- Use atomic operations for counters (AtomicInteger, AtomicLong)
- Synchronize state transitions
- Consider lock-free data structures for high throughput
- Use concurrent data structures for sliding windows (ConcurrentLinkedQueue)

## 6. Fallback Strategies

When a circuit breaker opens, the calling service needs a strategy to handle the failure gracefully. Simply returning an error is often not the best user experience.

### 6.1 Fallback Options Comparison

| Strategy | Use Case | Pros | Cons | Example |
|----------|----------|------|------|---------|
| **Default/Static Value** | Non-critical data | Simple, fast, reliable | May provide stale/generic data | Return empty list, default avatar |
| **Cached Response** | Data that changes slowly | Provides real data, better UX | Cache may be stale or invalid | Product catalog, user profiles |
| **Alternative Service** | Critical operations | Maintains functionality | Complexity, potential cost | Secondary payment gateway |
| **Graceful Degradation** | Features with dependencies | Core features still work | Reduced functionality | Disable recommendations |
| **Queue for Later** | Async operations | No data loss | Delayed processing, queue management | Background jobs, emails |
| **User Notification** | Transparent failures | Honest communication | Poor UX if overused | "Feature temporarily unavailable" |

### 6.2 Fallback Implementation Example

```
Request Flow with Fallback:

Client Request
     │
     ▼
┌─────────────────┐
│ Circuit Breaker │
└────────┬────────┘
         │
    ┌────┴────┐
    │ Closed? │
    └────┬────┘
         │
    ┌────┴──────────┐
    │               │
   Yes             No (Open)
    │               │
    ▼               ▼
┌─────────┐   ┌──────────────┐
│ Primary │   │   Fallback   │
│ Service │   │   Strategy   │
└────┬────┘   └──────┬───────┘
     │               │
     │          ┌────┴─────┐
     │          │          │
     │     ┌────▼───┐  ┌───▼────┐
     │     │ Cache  │  │Default │
     │     │Response│  │ Value  │
     │     └────┬───┘  └───┬────┘
     │          │          │
     └──────────┴──────────┘
                │
                ▼
          Return to Client
```

### 6.3 Real-World Fallback Examples

**E-commerce Product Recommendations:**
```
Primary: Real-time ML recommendation service
Fallback 1: Pre-computed popular items from cache
Fallback 2: Category-based bestsellers
Fallback 3: Hide recommendation section
```

**User Profile Service:**
```
Primary: User service database
Fallback 1: Redis cache (5-minute TTL)
Fallback 2: Default anonymous user profile
Fallback 3: Allow limited functionality without profile
```

**Payment Processing:**
```
Primary: Preferred payment gateway (Stripe)
Fallback 1: Secondary payment gateway (PayPal)
Fallback 2: Queue payment for retry
Fallback 3: Notify user, save cart for later
```

**Image Service:**
```
Primary: CDN with user uploads
Fallback 1: Cached version from edge
Fallback 2: Placeholder image
Fallback 3: Text-only mode
```

## 7. Circuit Breaker vs Retry Pattern

Both patterns handle failures in distributed systems, but they serve different purposes and should often be used together.

### 7.1 Comparison Table

| Aspect | Circuit Breaker | Retry Pattern |
|--------|----------------|---------------|
| **Purpose** | Prevent cascading failures | Overcome transient failures |
| **When to Use** | Sustained/persistent failures | Temporary network glitches |
| **Behavior** | Fails fast after threshold | Keeps trying with backoff |
| **Resource Impact** | Conserves resources | Can waste resources if misused |
| **Recovery Time** | Gives service time to heal | Immediate retry attempts |
| **Failure Detection** | Monitors failure patterns | Reacts to individual failures |
| **User Impact** | Immediate error/fallback | Adds latency (retries take time) |
| **Best For** | Protecting against overload | Handling transient errors |

### 7.2 When to Use Which

```
Decision Tree:

Is the service completely down or unresponsive?
│
├─ Yes → Circuit Breaker
│         (Don't waste resources retrying)
│
└─ No → Is this a transient error?
        │
        ├─ Yes → Retry Pattern
        │         (Temporary network blip, rate limit)
        │
        └─ No → Is this a client error (4xx)?
                │
                ├─ Yes → No retry, no circuit breaker
                │         (Fix the client request)
                │
                └─ No → Use both patterns together
                        (Retry with circuit breaker)
```

### 7.3 Combining Circuit Breaker and Retry

The most resilient systems use both patterns together:

```
┌────────────────────────────────────────────────┐
│              Request Flow                      │
├────────────────────────────────────────────────┤
│                                                │
│  1. Request arrives                            │
│     ↓                                          │
│  2. Circuit Breaker Check                      │
│     ├─ Open? → Fail fast (no retry)           │
│     └─ Closed? → Continue                      │
│     ↓                                          │
│  3. Call service with Retry                    │
│     ├─ Attempt 1 → Timeout                     │
│     ├─ Wait 100ms (exponential backoff)        │
│     ├─ Attempt 2 → Timeout                     │
│     ├─ Wait 200ms                              │
│     ├─ Attempt 3 → Timeout                     │
│     └─ Give up                                 │
│     ↓                                          │
│  4. Report failure to Circuit Breaker          │
│     (Increment failure counter)                │
│     ↓                                          │
│  5. If threshold exceeded → Open circuit       │
│     ↓                                          │
│  6. Future requests fail fast                  │
│                                                │
└────────────────────────────────────────────────┘
```

**Configuration Example:**
```yaml
resilience:
  circuitBreaker:
    failureThreshold: 5        # Open after 5 failed attempts
    openDuration: 60s          # Stay open for 60 seconds
    halfOpenRequests: 3        # Test with 3 requests

  retry:
    maxAttempts: 3             # Try up to 3 times
    waitDuration: 100ms        # Initial wait
    exponentialBackoff: 2      # Double wait each time
    retryOnExceptions:
      - TimeoutException
      - ConnectException
```

### 7.4 Anti-Patterns to Avoid

**Don't:**
- Retry indefinitely (use max attempts)
- Retry without exponential backoff (hammers the service)
- Use aggressive retries without circuit breaker (amplifies failures)
- Retry on all errors (don't retry 4xx client errors)
- Set retry timeout longer than circuit breaker threshold

## 8. Circuit Breaker in Practice

### 8.1 Popular Libraries and Frameworks

#### Resilience4j (Modern Java)

```java
// Configuration
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)                  // 50% failure rate
    .slidingWindowSize(10)                     // Last 10 calls
    .waitDurationInOpenState(Duration.ofSeconds(60))
    .permittedNumberOfCallsInHalfOpenState(3)
    .build();

// Create circuit breaker
CircuitBreaker circuitBreaker = CircuitBreaker.of("paymentService", config);

// Use with service call
Supplier<PaymentResponse> supplier = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> paymentService.processPayment(request));

// Execute
Try<PaymentResponse> result = Try.ofSupplier(supplier)
    .recover(CallNotPermittedException.class, fallbackPaymentResponse());
```

#### Netflix Hystrix (Legacy, now in maintenance mode)

```java
// Hystrix Command (legacy but educational)
public class PaymentCommand extends HystrixCommand<PaymentResponse> {

    private PaymentRequest request;

    public PaymentCommand(PaymentRequest request) {
        super(HystrixCommandGroupKey.Factory.asKey("Payment"));
        this.request = request;
    }

    @Override
    protected PaymentResponse run() throws Exception {
        return paymentService.processPayment(request);
    }

    @Override
    protected PaymentResponse getFallback() {
        // Fallback logic
        return PaymentResponse.queued();
    }
}

// Usage
PaymentResponse response = new PaymentCommand(request).execute();
```

**Note on Hystrix:** Netflix announced in 2018 that Hystrix is in maintenance mode. They recommend using Resilience4j or service mesh solutions like Istio for new projects.

#### Polly (.NET)

```csharp
// Configure circuit breaker policy
var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .OrResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(60),
        onBreak: (result, duration) => {
            Console.WriteLine($"Circuit opened for {duration}");
        },
        onReset: () => {
            Console.WriteLine("Circuit reset");
        }
    );

// Use the policy
var response = await circuitBreakerPolicy
    .ExecuteAsync(() => httpClient.GetAsync("https://api.example.com/data"));
```

#### Spring Cloud Circuit Breaker

```java
@Service
public class PaymentService {

    @Autowired
    private CircuitBreakerFactory circuitBreakerFactory;

    public PaymentResponse processPayment(PaymentRequest request) {
        CircuitBreaker circuitBreaker = circuitBreakerFactory.create("payment");

        return circuitBreaker.run(
            () -> paymentGateway.charge(request),
            throwable -> fallbackPayment(request)
        );
    }

    private PaymentResponse fallbackPayment(PaymentRequest request) {
        // Queue for later processing
        return PaymentResponse.queued();
    }
}
```

### 8.2 Service Mesh Implementation (Istio)

In [microservices architecture](../architecture/microservices.md), circuit breakers can be implemented at the infrastructure level using a service mesh like Istio:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-circuit-breaker
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 50
      minHealthPercent: 40
```

**Benefits of Service Mesh Approach:**
- Language-agnostic (works with any service)
- Centralized configuration
- Built-in observability
- No code changes required
- Consistent behavior across all services

**Comparison: Library vs Service Mesh**

| Aspect | Library (Resilience4j) | Service Mesh (Istio) |
|--------|------------------------|----------------------|
| **Code Changes** | Required | Not required |
| **Language Support** | Single language | Any language |
| **Granularity** | Fine-grained control | Coarser control |
| **Configuration** | In application | External YAML |
| **Learning Curve** | Moderate | Steep |
| **Overhead** | Minimal | Additional infrastructure |
| **Observability** | Custom integration | Built-in |
| **Best For** | Single service optimization | Polyglot microservices |

### 8.3 Integration with API Gateway

At the [API Gateway](../architecture/api-gateway.md) level, circuit breakers can protect against downstream service failures:

```
┌──────────┐         ┌─────────────────────────────────┐
│  Client  │────────▶│         API Gateway             │
└──────────┘         │                                 │
                     │  ┌────────────────────────────┐ │
                     │  │ Circuit Breaker: Auth      │ │
                     │  └──────────┬─────────────────┘ │
                     │             │                   │
                     │  ┌──────────▼─────────────────┐ │
                     │  │ Circuit Breaker: Orders    │ │
                     │  └──────────┬─────────────────┘ │
                     │             │                   │
                     │  ┌──────────▼─────────────────┐ │
                     │  │ Circuit Breaker: Payments  │ │
                     │  └──────────┬─────────────────┘ │
                     └─────────────┼───────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
        ┌──────────┐         ┌──────────┐        ┌──────────┐
        │   Auth   │         │  Orders  │        │ Payments │
        │  Service │         │  Service │        │  Service │
        └──────────┘         └──────────┘        └──────────┘
```

**Kong API Gateway Example:**

```yaml
plugins:
  - name: circuit-breaker
    config:
      failure_threshold: 5
      window_duration: 30
      state_open_duration: 60
      notify_interval: 10
```

## 9. Monitoring and Alerting

Effective circuit breaker monitoring is crucial for understanding system health and responding to incidents.

### 9.1 Key Metrics to Track

```
┌─────────────────────────────────────────────────────────┐
│              Circuit Breaker Metrics                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  State Metrics                                          │
│  ├─ Current State (Closed/Open/Half-Open)              │
│  ├─ Time in Current State                              │
│  ├─ State Transition Count                             │
│  └─ Last State Change Timestamp                        │
│                                                         │
│  Call Metrics                                           │
│  ├─ Total Calls                                        │
│  ├─ Successful Calls                                   │
│  ├─ Failed Calls                                       │
│  ├─ Rejected Calls (when open)                        │
│  └─ Call Duration (p50, p95, p99)                     │
│                                                         │
│  Failure Metrics                                        │
│  ├─ Failure Rate (%)                                   │
│  ├─ Consecutive Failures                               │
│  ├─ Failure Types (timeout, exception, 5xx)           │
│  └─ Time Since Last Failure                           │
│                                                         │
│  Health Metrics                                         │
│  ├─ Half-Open Test Success Rate                       │
│  ├─ Recovery Time (open → closed duration)            │
│  └─ Circuit Stability (flapping detection)            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 9.2 Dashboard Example

```
Circuit Breaker Dashboard - Payment Service
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Current State: CLOSED ✓              Last Changed: 2 hours ago

Success Rate: ████████████████░░ 89%   (Target: >95%)
Call Volume:  1,247 req/min            (Normal range: 1000-1500)

State History (Last 24h):
┌─────────────────────────────────────────────────────────┐
│ Closed: ████████████████████████░░░░░░░░░░░░░░░░░ 70%  │
│ Open:   ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 15%  │
│ Half:   ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 5%   │
│ N/A:    ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 10%  │
└─────────────────────────────────────────────────────────┘

Call Latency (ms):
  p50: 42ms   p95: 125ms   p99: 380ms   Max: 2,156ms

Recent Incidents:
• 14:23 - Circuit OPENED (5 consecutive timeouts)
• 14:24 - Circuit HALF-OPEN (timeout elapsed)
• 14:24 - Circuit CLOSED (3 test requests succeeded)

Failure Types (Last hour):
• Timeouts:        12  ████░░░░░░
• 5xx Errors:      5   ██░░░░░░░░
• Network:         3   █░░░░░░░░░
```

### 9.3 Prometheus Metrics

```yaml
# Circuit breaker state (gauge: 0=closed, 1=open, 2=half-open)
circuit_breaker_state{service="payment"} 0

# Total calls counter
circuit_breaker_calls_total{service="payment",result="success"} 45234
circuit_breaker_calls_total{service="payment",result="failure"} 892
circuit_breaker_calls_total{service="payment",result="rejected"} 145

# Call duration histogram
circuit_breaker_call_duration_seconds_bucket{service="payment",le="0.1"} 42000
circuit_breaker_call_duration_seconds_bucket{service="payment",le="0.5"} 44500
circuit_breaker_call_duration_seconds_bucket{service="payment",le="1.0"} 45800

# State transition counter
circuit_breaker_state_transitions_total{service="payment",from="closed",to="open"} 12
circuit_breaker_state_transitions_total{service="payment",from="open",to="half_open"} 12
circuit_breaker_state_transitions_total{service="payment",from="half_open",to="closed"} 10
```

### 9.4 Alert Definitions

**Critical Alerts:**

```yaml
# Circuit is open for too long
alert: CircuitBreakerOpenTooLong
expr: circuit_breaker_state{state="open"} == 1 for 5m
severity: critical
message: "Circuit breaker for {{$labels.service}} has been open for >5 minutes"

# High rejection rate
alert: CircuitBreakerHighRejectionRate
expr: rate(circuit_breaker_calls_total{result="rejected"}[5m]) > 10
severity: critical
message: "Circuit breaker rejecting >10 req/s for {{$labels.service}}"
```

**Warning Alerts:**

```yaml
# Circuit is flapping (opening/closing frequently)
alert: CircuitBreakerFlapping
expr: rate(circuit_breaker_state_transitions_total[10m]) > 0.5
severity: warning
message: "Circuit breaker for {{$labels.service}} is flapping (>5 transitions in 10min)"

# Elevated failure rate approaching threshold
alert: CircuitBreakerFailureRateHigh
expr: rate(circuit_breaker_calls_total{result="failure"}[5m]) > 5
severity: warning
message: "Failure rate elevated for {{$labels.service}} (approaching circuit breaker threshold)"
```

### 9.5 Distributed Tracing Integration

Circuit breaker events should be visible in distributed traces:

```
Trace: POST /api/orders/{id}/checkout
├─ api-gateway: 245ms
│  ├─ auth-service: 23ms [OK]
│  ├─ order-service: 45ms [OK]
│  └─ payment-service: 0ms [CIRCUIT_BREAKER_OPEN]
│     └─ fallback: cache_fallback [OK]
└─ Response: 200 OK (with cached payment status)

Span Attributes:
  circuit_breaker.state: open
  circuit_breaker.fallback_used: true
  circuit_breaker.fallback_type: cache
```

## 10. Trade-offs

### 10.1 Benefits

**Prevents Cascading Failures:**
- Stops failure propagation through the system
- Protects upstream services from resource exhaustion
- Maintains partial availability during outages

**Improves User Experience:**
- Fast failures instead of long timeouts
- Enables fallback responses
- Reduces perceived latency during failures

**Resource Protection:**
- Frees up threads/connections that would be wasted
- Reduces load on failing services (gives time to recover)
- Prevents thread pool exhaustion

**Observability:**
- Clear signal when services are unhealthy
- Metrics for monitoring system health
- Early warning system for cascading issues

**Graceful Degradation:**
- System remains partially functional
- Critical paths can continue working
- Better than complete system failure

### 10.2 Drawbacks

**Complexity:**
- Additional component to configure and maintain
- State machine adds complexity to debugging
- Need to understand when circuit is open vs actual service failures

**Configuration Challenges:**
- Difficult to choose right thresholds
- Too sensitive = unnecessary fail-fast
- Too lenient = doesn't prevent cascading failures
- Different services may need different configurations

**Fallback Responsibility:**
- Circuit breaker alone isn't enough
- Need to implement meaningful fallbacks
- Fallback logic can be complex

**Testing Complexity:**
- Hard to test all state transitions
- Need to simulate various failure scenarios
- Integration testing becomes more complex

**Potential for False Positives:**
- Brief spike in errors can open circuit unnecessarily
- During deployment, circuit may open temporarily
- Can mask real issues if fallbacks always succeed

**Coordination Overhead:**
- In distributed systems, multiple circuit breakers may trip
- No global view of system health
- Can lead to cascading circuit breaker openings

### 10.3 When NOT to Use Circuit Breaker

**Synchronous User Flows:**
```
Don't use for: User clicks "Save Profile" → Must save immediately
Use instead: Optimistic UI updates, local persistence, retry with feedback
```

**Database Connections:**
```
Don't use circuit breaker between app and primary database
Why: Database failure usually means app can't function
Use instead: Connection pooling, read replicas, database clustering
```

**Internal Service Calls (Same Data Center):**
```
May not need for: Service A → Service B (same data center, low latency)
Why: Network is reliable, failures are typically not transient
Consider: Direct failure handling may be sufficient
```

**Low Traffic Services:**
```
Don't use for: Admin panel with 1 req/minute
Why: Not enough traffic to establish failure patterns
State transitions would be erratic
```

## 11. Related Patterns

Circuit breakers work best as part of a comprehensive resilience strategy.

### 11.1 Bulkhead Pattern

Isolates resources to prevent total system failure:

```
Without Bulkhead:              With Bulkhead:
┌──────────────────────┐       ┌──────────────────────┐
│   Thread Pool        │       │  Thread Pools        │
│   (50 threads)       │       │                      │
│                      │       │ ┌────────────────┐   │
│  [All threads used   │       │ │ Payment (10)   │   │
│   by slow payment    │       │ └────────────────┘   │
│   service calls]     │       │ ┌────────────────┐   │
│                      │       │ │ Orders (20)    │   │
│  Other services      │       │ └────────────────┘   │
│  can't get threads   │       │ ┌────────────────┐   │
│                      │       │ │ Other (20)     │   │
└──────────────────────┘       │ └────────────────┘   │
                               │                      │
                               │ Payment fails, but   │
                               │ others still work    │
                               └──────────────────────┘
```

**Use Circuit Breaker + Bulkhead:**
- Bulkhead limits resource consumption per service
- Circuit breaker fails fast when service is down
- Together: Maximum isolation and protection

### 11.2 Retry Pattern

Covered in section 7, but key points:

```
Request → Retry → Circuit Breaker → Service
          (3 attempts)  (fail fast if open)
```

**Integration Strategy:**
1. Retry handles transient failures (network blips)
2. Circuit breaker prevents retry storms during outages
3. Use exponential backoff with jitter in retries
4. Circuit breaker monitors retry outcomes

### 11.3 Timeout Pattern

Circuit breakers depend on timeouts:

```
Request ──────────────────────────────────▶ Service
        │                                  │
        │ Timeout: 3 seconds               │
        │                                  ╳ (slow/hung)
        ╳
        │
        └─────▶ Report timeout to Circuit Breaker
```

**Timeout Configuration:**
- Set request timeout shorter than circuit breaker evaluation window
- Example: 3-second timeout, 10-second circuit breaker window
- Without timeouts, circuit breaker can't detect slow services

### 11.4 Health Check Pattern

External health monitoring for circuit breaker decisions:

```
┌─────────────────┐         ┌─────────────────┐
│  Health Check   │────────▶│    Service      │
│  Endpoint       │         │  /health        │
└────────┬────────┘         └─────────────────┘
         │
         │ 200 OK = healthy
         │ 503 = unhealthy
         │
         ▼
┌─────────────────┐
│ Circuit Breaker │
│  (can use       │
│   health status │
│   as input)     │
└─────────────────┘
```

### 11.5 Fallback Pattern

See Section 6 for details. Key integration:

```
Circuit Breaker ──open──▶ Fallback Strategy
                           ├─ Cache
                           ├─ Default value
                           ├─ Alternative service
                           └─ Graceful degradation
```

### 11.6 Integration with Distributed Transactions

When implementing [Saga Pattern](../distributed-transactions/saga.md), circuit breakers affect compensation logic:

```
Saga Step 1: Reserve Inventory ✓
Saga Step 2: Process Payment ──▶ Circuit Open!
             │
             ├─ Fallback: Queue payment for later
             └─ Compensate: Release inventory reservation
```

**Considerations:**
- Circuit breaker open doesn't always mean saga failure
- May queue saga steps for later execution
- Compensation logic must handle circuit breaker states

### 11.7 Event-Driven Architecture Integration

In [event-driven architecture](../event-driven/event-driven-architecture.md), circuit breakers protect event publishers:

```
Service A ──publish event──▶ Circuit Breaker ──▶ Message Queue

If queue is down:
  ├─ Circuit opens
  ├─ Queue events locally
  └─ Retry when circuit closes
```

## 12. Real-World Examples

### 12.1 Netflix Hystrix Story

Netflix pioneered circuit breakers at scale, processing billions of requests per day.

**The Problem:**
In 2011, Netflix experienced a major outage when a single service failure cascaded through their entire streaming platform. When their recommendation service became slow, it consumed all available threads in the API gateway, bringing down the entire site.

**The Solution:**
Netflix created Hystrix, a latency and fault tolerance library:

```
Netflix API Gateway (Zuul)
├─ Hystrix: User Service
├─ Hystrix: Recommendation Service  ← Circuit opened during failure
├─ Hystrix: Playback Service
└─ Hystrix: Rating Service

When recommendation service failed:
✓ Circuit opened after 20 failures in 10 seconds
✓ Fell back to popular/trending content
✓ Other services continued working normally
✓ Prevented 3-hour outage, reduced to 3-minute degradation
```

**Key Metrics:**
- 200+ billion Hystrix commands per day
- Average circuit breaker trip time: 45 seconds
- Prevented estimated 10+ major outages per year
- Enabled 4-nines availability (99.99%)

**Lessons Learned:**
1. Default all service calls to fail-fast with circuit breakers
2. Monitor circuit breaker metrics in real-time dashboards
3. Plan meaningful fallbacks for all critical services
4. Test circuit breaker behavior in production with chaos engineering

**Evolution:**
In 2018, Netflix moved from Hystrix to service mesh (Envoy/Istio) for circuit breaking, as they scaled to thousands of microservices.

### 12.2 Amazon Retail Platform

Amazon uses circuit breakers extensively in their retail platform:

**Product Page Example:**
```
Product Page Composition:
┌────────────────────────────────────────┐
│ Product Details (CB: primary DB)      │ ← Fallback: Cache
├────────────────────────────────────────┤
│ Price & Availability (CB: inventory)   │ ← Fallback: "Check availability"
├────────────────────────────────────────┤
│ Recommendations (CB: ML service)       │ ← Fallback: Bestsellers
├────────────────────────────────────────┤
│ Reviews (CB: review service)           │ ← Fallback: Hide section
├────────────────────────────────────────┤
│ Q&A (CB: Q&A service)                 │ ← Fallback: Hide section
└────────────────────────────────────────┘

If recommendation service fails:
- Circuit breaker opens
- Page still loads in <200ms
- Shows bestsellers instead of personalized
- User can still purchase
```

**Black Friday Resilience:**
During peak shopping events, Amazon's circuit breakers:
- Automatically degrade non-essential features
- Maintain core checkout functionality
- Prevent cascading failures from traffic spikes
- Enable gradual recovery as load decreases

### 12.3 Uber Ride Matching

Uber uses circuit breakers in their real-time ride matching system:

```
Rider Request Flow:
┌────────────┐
│   Rider    │
│   App      │
└─────┬──────┘
      │
      ▼
┌─────────────────────────────────────┐
│  API Gateway (Circuit Breakers)     │
├─────────────────────────────────────┤
│  CB: Dispatch Service (Primary)     │──▶ Find closest drivers
│    Fallback: Region-wide dispatch   │
│                                     │
│  CB: Pricing Service                │──▶ Calculate fare
│    Fallback: Default pricing table  │
│                                     │
│  CB: ETA Service                    │──▶ Estimate arrival
│    Fallback: Geographic approximation│
└─────────────────────────────────────┘
```

**Failure Scenario:**
1. Dispatch service becomes slow (database issue)
2. Circuit breaker trips after 5 consecutive slow responses
3. Falls back to region-wide dispatch (less optimal, but works)
4. Rides continue to be matched (slightly longer wait times)
5. Service recovers, circuit closes, optimal matching resumes

**Impact:**
- Prevented complete matching system outage
- Maintained 80% normal capacity during incident
- Recovered in 2 minutes instead of 20+ minutes

### 12.4 Twitter Timeline Service

Twitter's timeline service uses circuit breakers to handle celebrity tweet storms:

```
Timeline Composition Service:
├─ CB: Tweet Fetch Service
│  └─ Fallback: Cached tweets
├─ CB: Media Service  ← Opens during viral media spikes
│  └─ Fallback: Placeholder images
├─ CB: Analytics Service
│  └─ Fallback: Static counts
└─ CB: Ad Service
   └─ Fallback: No ads (revenue loss, but timeline works)

When celebrity tweets go viral:
- Media service gets overwhelmed
- Circuit breaker opens
- Timelines show placeholder images
- Core reading experience preserved
- Media service scales up
- Circuit closes when healthy
```

## 13. Key Takeaways / Interview Framework

### 13.1 Essential Concepts to Remember

**Definition in 30 Seconds:**
"Circuit Breaker is a design pattern that prevents cascading failures in distributed systems by monitoring for failures and temporarily blocking requests to failing services. Like an electrical circuit breaker, it 'opens' when failures exceed a threshold, fails fast instead of wasting resources, and periodically tests if the service has recovered."

**The Three States:**
1. **CLOSED:** Normal operation, requests pass through
2. **OPEN:** Failure threshold exceeded, fail fast
3. **HALF-OPEN:** Testing recovery with limited requests

**Key Parameters:**
- Failure threshold (when to open)
- Timeout duration (how long to stay open)
- Success threshold (when to close from half-open)

### 13.2 Interview Discussion Framework

**When Asked "Design a Resilient Service":**

1. **Start with the problem:**
   - "In distributed systems, one failing service can cascade and take down the entire system"
   - "Circuit breakers prevent this by failing fast and giving services time to recover"

2. **Draw the state machine:**
   ```
   CLOSED ──failures──▶ OPEN ──timeout──▶ HALF-OPEN
     ▲                                       │
     └───────────success────────────────────┘
   ```

3. **Discuss configuration:**
   - "I'd configure it to open after 5 consecutive failures or 50% failure rate in a 10-second window"
   - "Stay open for 60 seconds to allow recovery"
   - "Test with 3 requests in half-open state"

4. **Plan fallback strategies:**
   - "For read operations, return cached data"
   - "For writes, queue for later processing"
   - "For critical operations, use alternative service"

5. **Add monitoring:**
   - "Track circuit state, failure rates, and state transitions"
   - "Alert when circuit is open for >5 minutes"
   - "Dashboard showing circuit health across all services"

### 13.3 Common Interview Questions

**Q: "How does Circuit Breaker differ from Retry pattern?"**

A: "Retry pattern handles transient failures by attempting the operation multiple times with backoff. Circuit Breaker prevents repeated attempts when a service is consistently failing. They complement each other: use retries within a circuit breaker to handle transient issues, but let the circuit breaker stop retries when the service is truly down."

**Q: "How do you choose the failure threshold?"**

A: "It depends on the service's normal error rate and latency:
- For critical services with low normal error rate: 5 consecutive failures or 20% error rate
- For services with higher variability: 10 failures or 50% error rate in a time window
- Use sliding window (last 10 requests or last 30 seconds) rather than consecutive failures
- Monitor in production and tune based on false positive rate"

**Q: "What happens if the circuit opens during a write operation?"**

A: "Several strategies:
1. Queue the write for later processing (eventual consistency)
2. Write to local storage and sync when service recovers
3. Return error with retry-after header
4. Use alternative service if available
The choice depends on business requirements around data consistency and user experience."

**Q: "How do you prevent thundering herd when circuit closes?"**

A: "The HALF-OPEN state solves this. Instead of immediately sending all traffic when the circuit closes, we:
1. Allow only a few test requests (e.g., 3) in half-open state
2. If tests succeed, gradually increase traffic
3. If tests fail, immediately reopen and wait longer
4. Can also add jitter to timeout durations across instances"

**Q: "Circuit breaker at service level or instance level?"**

A: "Depends on deployment model:
- **Service level** (shared): All instances share circuit state via Redis/distributed cache
  - Pros: Consistent behavior, faster failure detection
  - Cons: Additional dependency, coordination overhead
- **Instance level** (independent): Each instance has own circuit
  - Pros: Simple, no external dependency
  - Cons: May take longer to detect failures, inconsistent user experience

I'd use instance-level for most cases, and service-level only for critical paths where consistent behavior is essential."

### 13.4 Design Considerations Checklist

When implementing circuit breakers, consider:

**Configuration:**
- [ ] Failure threshold appropriate for service SLA
- [ ] Timeout duration allows service to recover
- [ ] Half-open test requests sufficient but not overwhelming
- [ ] Sliding window size balances responsiveness and stability

**Failure Detection:**
- [ ] What counts as failure? (timeouts, exceptions, 5xx)
- [ ] What doesn't count? (4xx client errors)
- [ ] How to handle slow responses? (timeout-based)

**Fallback Strategy:**
- [ ] Meaningful fallback exists for each circuit breaker
- [ ] Fallback tested and maintained
- [ ] Fallback doesn't depend on same failing service
- [ ] Cache invalidation strategy if using cached fallbacks

**Monitoring:**
- [ ] Circuit state exposed as metric
- [ ] Failure rates tracked and alerted
- [ ] State transitions logged
- [ ] Dashboard showing circuit health
- [ ] Distributed tracing includes circuit breaker spans

**Testing:**
- [ ] Unit tests for state transitions
- [ ] Integration tests for failure scenarios
- [ ] Chaos engineering to test production behavior
- [ ] Load tests with circuit breakers enabled

**Operational Concerns:**
- [ ] How to manually open/close circuits?
- [ ] How to adjust thresholds without redeployment?
- [ ] How to test circuit breaker without affecting users?
- [ ] Runbook for circuit breaker incidents

### 13.5 Advanced Topics for Senior Interviews

**Adaptive Circuit Breakers:**
- Automatically adjust thresholds based on historical failure rates
- Machine learning to predict failures before they cascade
- Context-aware circuit breaking (different thresholds for different users/regions)

**Multi-Level Circuit Breakers:**
- Circuit breakers at multiple layers (gateway, service, client)
- Coordination between circuit breakers
- Hierarchical failure management

**Global Circuit Breakers:**
- Shared state across data centers using distributed consensus
- Eventual consistency in circuit breaker state
- Handling network partitions in distributed circuit breakers

**Circuit Breakers in Event-Driven Systems:**
- How circuit breakers work with async messaging
- Circuit breaking for event producers vs consumers
- Backpressure and circuit breakers

---

## Summary

The Circuit Breaker pattern is essential for building resilient distributed systems. By monitoring failures and failing fast when services are unhealthy, circuit breakers:

- Prevent cascading failures
- Conserve resources (threads, connections, memory)
- Improve user experience with fast failures and fallbacks
- Give failing services time to recover
- Provide clear signals for monitoring and alerting

Combined with retry patterns, timeouts, and fallbacks, circuit breakers form the foundation of resilient [microservices architecture](../architecture/microservices.md).

**Remember:** Circuit breakers are not just about handling failures—they're about graceful degradation and maintaining system availability even when components fail.

---

## Practical Implementation

For a hands-on PHP/Laravel implementation of this pattern (gold price service with Cache-based state management), see [Circuit Breaker in Laravel](../../examples/circuit-breaker.md).

---

**References:**
- Michael Nygard, "Release It! Design and Deploy Production-Ready Software" (2007)
- Martin Fowler, "CircuitBreaker" blog post (2014)
- Netflix Tech Blog: Hystrix documentation and case studies
- Resilience4j documentation
- Microsoft Azure Architecture: Circuit Breaker pattern
