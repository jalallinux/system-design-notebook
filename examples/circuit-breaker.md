# Circuit Breaker Pattern - Laravel Implementation

## Introduction

This document provides a practical PHP/Laravel implementation of the [Circuit Breaker pattern](../docs/resilience/circuit-breaker.md). The example builds a gold price fetching service that calls an external API (`https://call5.tgju.org/ajax.json`) and uses a circuit breaker to handle failures gracefully.

If you're not familiar with the Circuit Breaker pattern theory (three states, failure thresholds, recovery mechanisms), read the [Circuit Breaker theory documentation](../docs/resilience/circuit-breaker.md) first.

---

## Architecture Overview

```
┌──────────────┐     ┌────────────────────┐     ┌────────────────┐     ┌──────────────────┐
│   Client     │────▶│ GoldPriceController │────▶│ GoldPriceService│────▶│ CircuitBreaker   │
│  (Browser/   │     │                    │     │                │     │   (State Machine) │
│   API call)  │     └────────────────────┘     └────────┬───────┘     └──────────────────┘
└──────────────┘                                         │
                                                         ▼
                                                ┌──────────────────┐
                                                │  External API    │
                                                │  (TGJU Gold API) │
                                                └──────────────────┘
```

**Components:**

| Component | Responsibility |
|-----------|---------------|
| `CircuitBreaker` | State machine managing CLOSED/OPEN/HALF_OPEN transitions using Laravel Cache |
| `GoldPriceService` | HTTP client that wraps the circuit breaker around external API calls |
| `CircuitBreakerOpenException` | Custom exception thrown when the circuit is OPEN |
| `GoldPriceController` | REST endpoint handling HTTP responses and error formatting |

---

## Implementation

### 1. CircuitBreaker Service

The core state machine. All state is stored in Laravel's Cache (Redis, Memcached, etc.), so it works across multiple workers/processes.

**File:** `app/Services/CircuitBreaker.php`

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Cache;

class CircuitBreaker
{
    const STATE_CLOSED = 'closed';
    const STATE_OPEN = 'open';
    const STATE_HALF_OPEN = 'half_open';

    public function __construct(
        private string $service,
        private int $failureThreshold = 5,
        private int $recoveryTimeout = 30,    // seconds
        private int $successThreshold = 2,
    ) {}

    public function getState(): string
    {
        return Cache::get($this->key('state'), self::STATE_CLOSED);
    }

    public function isAvailable(): bool
    {
        $state = $this->getState();

        if ($state === self::STATE_CLOSED) {
            return true;
        }

        if ($state === self::STATE_OPEN) {
            $lastFailedAt = Cache::get($this->key('last_failed_at'), 0);

            if (time() - $lastFailedAt >= $this->recoveryTimeout) {
                $this->transitionTo(self::STATE_HALF_OPEN);
                return true;
            }

            return false;
        }

        // HALF_OPEN — allow limited requests
        return true;
    }

    public function recordSuccess(): void
    {
        if ($this->getState() === self::STATE_HALF_OPEN) {
            $successes = Cache::increment($this->key('half_open_successes'));

            if ($successes >= $this->successThreshold) {
                $this->reset();
            }
        } else {
            // In CLOSED state, reset failure count on success
            Cache::put($this->key('failures'), 0, now()->addMinutes(10));
        }
    }

    public function recordFailure(): void
    {
        $state = $this->getState();

        if ($state === self::STATE_HALF_OPEN) {
            $this->trip();
            return;
        }

        $failures = Cache::increment($this->key('failures'));
        Cache::put($this->key('last_failed_at'), time(), now()->addMinutes(10));

        if ($failures >= $this->failureThreshold) {
            $this->trip();
        }
    }

    public function getFailureCount(): int
    {
        return (int) Cache::get($this->key('failures'), 0);
    }

    private function trip(): void
    {
        $this->transitionTo(self::STATE_OPEN);
        Cache::put($this->key('last_failed_at'), time(), now()->addMinutes(10));
    }

    private function reset(): void
    {
        $this->transitionTo(self::STATE_CLOSED);
        Cache::forget($this->key('failures'));
        Cache::forget($this->key('last_failed_at'));
        Cache::forget($this->key('half_open_successes'));
    }

    private function transitionTo(string $state): void
    {
        Cache::put($this->key('state'), $state, now()->addMinutes(10));

        if ($state === self::STATE_HALF_OPEN) {
            Cache::put($this->key('half_open_successes'), 0, now()->addMinutes(10));
        }
    }

    private function key(string $suffix): string
    {
        return "circuit_breaker:{$this->service}:{$suffix}";
    }
}
```

**Key Design Decisions:**

- **Laravel Cache as state store:** Enables shared state across multiple PHP-FPM workers and queue workers. Works with Redis, Memcached, or any Cache driver.
- **Cache key namespacing:** Each service gets isolated keys via `circuit_breaker:{service}:{suffix}`.
- **TTL on all cache entries:** All keys expire after 10 minutes to prevent stale state if the application restarts.

---

### 2. GoldPriceService

The HTTP client that wraps the circuit breaker around the external TGJU API.

**File:** `app/Services/GoldPriceService.php`

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Http;
use App\Exceptions\CircuitBreakerOpenException;

class GoldPriceService
{
    private CircuitBreaker $circuitBreaker;

    public function __construct()
    {
        $this->circuitBreaker = new CircuitBreaker(
            service: 'tgju_gold_price',
            failureThreshold: 5,
            recoveryTimeout: 30,
            successThreshold: 2,
        );
    }

    public function getPrice(): array
    {
        if (! $this->circuitBreaker->isAvailable()) {
            throw new CircuitBreakerOpenException(
                "Circuit breaker is OPEN for tgju_gold_price. "
                . "Failures: {$this->circuitBreaker->getFailureCount()}. "
                . "Try again later."
            );
        }

        try {
            $response = Http::timeout(5)
                ->connectTimeout(3)
                ->get('https://call5.tgju.org/ajax.json');

            if ($response->failed()) {
                $this->circuitBreaker->recordFailure();
                throw new \RuntimeException("TGJU API returned status {$response->status()}");
            }

            $data = $response->json();
            $this->circuitBreaker->recordSuccess();

            return [
                'gold_ounce'  => $data['ounce']['p'] ?? null,
                'gold_gram'   => $data['geram18']['p'] ?? null,
                'gold_coin'   => $data['sekee']['p'] ?? null,
                'usd'         => $data['usd']['p'] ?? null,
                'state'       => $this->circuitBreaker->getState(),
                'fetched_at'  => now()->toDateTimeString(),
            ];
        } catch (CircuitBreakerOpenException $e) {
            throw $e;
        } catch (\Throwable $e) {
            $this->circuitBreaker->recordFailure();
            throw $e;
        }
    }

    public function getStatus(): array
    {
        return [
            'state'    => $this->circuitBreaker->getState(),
            'failures' => $this->circuitBreaker->getFailureCount(),
        ];
    }
}
```

**Configuration:**

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `failureThreshold` | 5 | Open the circuit after 5 failures |
| `recoveryTimeout` | 30s | Wait 30 seconds before probing |
| `successThreshold` | 2 | Require 2 successful probes to close |
| HTTP `timeout` | 5s | Max wait for API response |
| HTTP `connectTimeout` | 3s | Max wait to establish connection |

---

### 3. CircuitBreakerOpenException

**File:** `app/Exceptions/CircuitBreakerOpenException.php`

```php
<?php

namespace App\Exceptions;

use RuntimeException;

class CircuitBreakerOpenException extends RuntimeException {}
```

---

### 4. GoldPriceController

**File:** `app/Http/Controllers/GoldPriceController.php`

```php
<?php

namespace App\Http\Controllers;

use App\Services\GoldPriceService;
use App\Exceptions\CircuitBreakerOpenException;
use Illuminate\Http\JsonResponse;

class GoldPriceController extends Controller
{
    public function __construct(
        private GoldPriceService $goldPriceService,
    ) {}

    public function index(): JsonResponse
    {
        try {
            $data = $this->goldPriceService->getPrice();

            return response()->json([
                'success' => true,
                'data'    => $data,
            ]);
        } catch (CircuitBreakerOpenException $e) {
            return response()->json([
                'success' => false,
                'error'   => 'Service temporarily unavailable',
                'message' => $e->getMessage(),
                'circuit' => $this->goldPriceService->getStatus(),
            ], 503);
        } catch (\Throwable $e) {
            return response()->json([
                'success' => false,
                'error'   => 'Failed to fetch gold price',
                'message' => $e->getMessage(),
                'circuit' => $this->goldPriceService->getStatus(),
            ], 502);
        }
    }

    public function status(): JsonResponse
    {
        return response()->json($this->goldPriceService->getStatus());
    }
}
```

**HTTP Response Codes:**

| Code | Meaning |
|------|---------|
| `200` | Success — data fetched from API |
| `502` | Bad Gateway — upstream API returned an error |
| `503` | Service Unavailable — circuit breaker is OPEN |

---

### 5. Routes

**File:** `routes/api.php`

```php
use App\Http\Controllers\GoldPriceController;

Route::get('/gold-price', [GoldPriceController::class, 'index']);
Route::get('/gold-price/circuit-status', [GoldPriceController::class, 'status']);
```

| Endpoint | Description |
|----------|-------------|
| `GET /api/gold-price` | Fetch current gold prices (or fail with circuit breaker info) |
| `GET /api/gold-price/circuit-status` | Check the circuit breaker state without making an API call |

---

## How It Works

### State Diagram

```
CLOSED ──[5 failures]──▶ OPEN ──[30s timeout]──▶ HALF_OPEN
  ▲                        ▲                         │
  │                        │        [failure]         │
  │                        +─────────────────────────+
  │
  └───────────[2 successes]──── HALF_OPEN
```

### State Transition Table

| Current State | Event | Next State | Action |
|---------------|-------|------------|--------|
| CLOSED | Request succeeds | CLOSED | Reset failure counter |
| CLOSED | Request fails (count < 5) | CLOSED | Increment failure counter |
| CLOSED | Request fails (count >= 5) | OPEN | Trip the circuit, record timestamp |
| OPEN | Request arrives (timeout not elapsed) | OPEN | Throw `CircuitBreakerOpenException` immediately |
| OPEN | Request arrives (timeout elapsed) | HALF_OPEN | Allow request through as probe |
| HALF_OPEN | Probe succeeds (count < 2) | HALF_OPEN | Increment success counter |
| HALF_OPEN | Probe succeeds (count >= 2) | CLOSED | Reset all counters |
| HALF_OPEN | Probe fails | OPEN | Trip the circuit again |

### Example Request Flow

**Normal operation (CLOSED):**
```
Client → GET /api/gold-price
       → GoldPriceController::index()
       → GoldPriceService::getPrice()
       → CircuitBreaker::isAvailable() → true
       → Http::get('https://call5.tgju.org/ajax.json')
       → 200 OK, JSON data
       → CircuitBreaker::recordSuccess()
       → Return { success: true, data: { gold_ounce: "2,345.67", ... } }
```

**After 5 failures (OPEN):**
```
Client → GET /api/gold-price
       → GoldPriceController::index()
       → GoldPriceService::getPrice()
       → CircuitBreaker::isAvailable() → false
       → throw CircuitBreakerOpenException
       → Return 503 { success: false, error: "Service temporarily unavailable" }
```

**After 30s recovery timeout (HALF_OPEN):**
```
Client → GET /api/gold-price
       → CircuitBreaker::isAvailable() → true (transitions to HALF_OPEN)
       → Http::get('https://call5.tgju.org/ajax.json')
       → 200 OK → recordSuccess() → (1 of 2 needed)
       → Next successful request → recordSuccess() → (2 of 2) → reset to CLOSED
```

---

## Cache Keys

All state is stored under these cache keys:

| Key | Type | Description |
|-----|------|-------------|
| `circuit_breaker:tgju_gold_price:state` | string | Current state (`closed`, `open`, `half_open`) |
| `circuit_breaker:tgju_gold_price:failures` | int | Current failure count |
| `circuit_breaker:tgju_gold_price:last_failed_at` | int | Unix timestamp of the last failure |
| `circuit_breaker:tgju_gold_price:half_open_successes` | int | Success count during HALF_OPEN probing |

---

## Further Reading

- [Circuit Breaker Pattern - Theory and Design](../docs/resilience/circuit-breaker.md) - Comprehensive theory including fallback strategies, monitoring, and real-world examples from Netflix, Amazon, and Uber
- [CAP Theorem](../docs/fundamentals/cap-theorem.md) - The fundamental trade-off between consistency and availability that motivates circuit breakers
- [Saga Pattern](../docs/distributed-transactions/saga.md) - How circuit breakers interact with distributed transaction compensation logic
