# الگوی Circuit Breaker - پیاده‌سازی Laravel

## مقدمه

این سند یک پیاده‌سازی عملی PHP/Laravel از [الگوی Circuit Breaker](../docs/resilience/circuit-breaker.md) ارائه می‌دهد. این مثال یک سرویس دریافت قیمت طلا می‌سازد که یک API خارجی (`https://call5.tgju.org/ajax.json`) را فراخوانی می‌کند و از Circuit Breaker برای مدیریت خطاها به صورت ظریف استفاده می‌کند.

اگر با تئوری الگوی Circuit Breaker (سه وضعیت، آستانه خطا، مکانیزم‌های بازیابی) آشنا نیستید، ابتدا [مستندات تئوری Circuit Breaker](../docs/resilience/circuit-breaker.md) را بخوانید.

---

## نمای کلی معماری

```
┌──────────────┐     ┌────────────────────┐     ┌────────────────┐     ┌──────────────────┐
│   کلاینت     │────▶│ GoldPriceController │────▶│ GoldPriceService│────▶│ CircuitBreaker   │
│  (مرورگر/   │     │                    │     │                │     │   (ماشین حالت)   │
│  فراخوانی API)│     └────────────────────┘     └────────┬───────┘     └──────────────────┘
└──────────────┘                                         │
                                                         ▼
                                                ┌──────────────────┐
                                                │  API خارجی       │
                                                │  (TGJU Gold API) │
                                                └──────────────────┘
```

**اجزا:**

| جزء | مسئولیت |
|-----|---------|
| `CircuitBreaker` | ماشین حالت مدیریت‌کننده انتقال بین وضعیت‌های CLOSED/OPEN/HALF_OPEN با استفاده از Laravel Cache |
| `GoldPriceService` | کلاینت HTTP که Circuit Breaker را دور فراخوانی‌های API خارجی قرار می‌دهد |
| `CircuitBreakerOpenException` | استثنای سفارشی که هنگام OPEN بودن مدار پرتاب می‌شود |
| `GoldPriceController` | نقطه پایانی REST برای مدیریت پاسخ‌های HTTP و قالب‌بندی خطا |

---

## پیاده‌سازی

### 1. سرویس CircuitBreaker

ماشین حالت اصلی. تمام وضعیت‌ها در Cache لاراول (Redis، Memcached و غیره) ذخیره می‌شوند، بنابراین در چندین worker و process PHP-FPM کار می‌کند.

**فایل:** `app/Services/CircuitBreaker.php`

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
        private int $recoveryTimeout = 30,    // ثانیه
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

        // HALF_OPEN — اجازه درخواست‌های محدود
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
            // در وضعیت CLOSED، شمارنده خطا را با موفقیت ریست کن
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

**تصمیمات طراحی کلیدی:**

- **Cache لاراول به عنوان ذخیره‌ساز وضعیت:** وضعیت مشترک بین چندین worker PHP-FPM و queue worker را ممکن می‌سازد. با Redis، Memcached یا هر درایور Cache کار می‌کند.
- **فضای نام کلید Cache:** هر سرویس کلیدهای ایزوله از طریق `circuit_breaker:{service}:{suffix}` دارد.
- **TTL روی تمام ورودی‌های cache:** تمام کلیدها بعد از ۱۰ دقیقه منقضی می‌شوند تا از وضعیت قدیمی در صورت راه‌اندازی مجدد برنامه جلوگیری شود.

---

### 2. سرویس GoldPriceService

کلاینت HTTP که Circuit Breaker را دور API خارجی TGJU قرار می‌دهد.

**فایل:** `app/Services/GoldPriceService.php`

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

**پیکربندی:**

| پارامتر | مقدار | معنی |
|---------|-------|------|
| `failureThreshold` | 5 | باز کردن مدار بعد از ۵ خطا |
| `recoveryTimeout` | 30 ثانیه | ۳۰ ثانیه صبر قبل از بررسی مجدد |
| `successThreshold` | 2 | نیاز به ۲ بررسی موفق برای بستن مدار |
| HTTP `timeout` | 5 ثانیه | حداکثر انتظار برای پاسخ API |
| HTTP `connectTimeout` | 3 ثانیه | حداکثر انتظار برای برقراری اتصال |

---

### 3. CircuitBreakerOpenException

**فایل:** `app/Exceptions/CircuitBreakerOpenException.php`

```php
<?php

namespace App\Exceptions;

use RuntimeException;

class CircuitBreakerOpenException extends RuntimeException {}
```

---

### 4. کنترلر GoldPriceController

**فایل:** `app/Http/Controllers/GoldPriceController.php`

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

**کدهای پاسخ HTTP:**

| کد | معنی |
|----|------|
| `200` | موفقیت — داده از API دریافت شد |
| `502` | Bad Gateway — API بالادستی خطا برگرداند |
| `503` | سرویس در دسترس نیست — Circuit Breaker باز (OPEN) است |

---

### 5. مسیرها

**فایل:** `routes/api.php`

```php
use App\Http\Controllers\GoldPriceController;

Route::get('/gold-price', [GoldPriceController::class, 'index']);
Route::get('/gold-price/circuit-status', [GoldPriceController::class, 'status']);
```

| نقطه پایانی | توضیح |
|-------------|-------|
| `GET /api/gold-price` | دریافت قیمت فعلی طلا (یا خطا با اطلاعات Circuit Breaker) |
| `GET /api/gold-price/circuit-status` | بررسی وضعیت Circuit Breaker بدون فراخوانی API |

---

## نحوه کار

### نمودار وضعیت

```
CLOSED ──[5 خطا]──▶ OPEN ──[30 ثانیه]──▶ HALF_OPEN
  ▲                    ▲                      │
  │                    │       [خطا]           │
  │                    +──────────────────────+
  │
  └────────[2 موفقیت]──── HALF_OPEN
```

### جدول انتقال وضعیت

| وضعیت فعلی | رویداد | وضعیت بعدی | عمل |
|------------|--------|-----------|-----|
| CLOSED | درخواست موفق | CLOSED | ریست شمارنده خطا |
| CLOSED | خطای درخواست (تعداد < 5) | CLOSED | افزایش شمارنده خطا |
| CLOSED | خطای درخواست (تعداد >= 5) | OPEN | فعال کردن مدار، ثبت زمان |
| OPEN | درخواست وارد می‌شود (زمان انتظار سپری نشده) | OPEN | پرتاب فوری `CircuitBreakerOpenException` |
| OPEN | درخواست وارد می‌شود (زمان انتظار سپری شده) | HALF_OPEN | اجازه عبور درخواست به عنوان بررسی |
| HALF_OPEN | بررسی موفق (تعداد < 2) | HALF_OPEN | افزایش شمارنده موفقیت |
| HALF_OPEN | بررسی موفق (تعداد >= 2) | CLOSED | ریست تمام شمارنده‌ها |
| HALF_OPEN | خطای بررسی | OPEN | فعال کردن مجدد مدار |

### نمونه جریان درخواست

**عملکرد عادی (CLOSED):**
```
Client → GET /api/gold-price
       → GoldPriceController::index()
       → GoldPriceService::getPrice()
       → CircuitBreaker::isAvailable() → true
       → Http::get('https://call5.tgju.org/ajax.json')
       → 200 OK, داده JSON
       → CircuitBreaker::recordSuccess()
       → برگشت { success: true, data: { gold_ounce: "2,345.67", ... } }
```

**بعد از ۵ خطا (OPEN):**
```
Client → GET /api/gold-price
       → GoldPriceController::index()
       → GoldPriceService::getPrice()
       → CircuitBreaker::isAvailable() → false
       → throw CircuitBreakerOpenException
       → برگشت 503 { success: false, error: "Service temporarily unavailable" }
```

**بعد از ۳۰ ثانیه زمان بازیابی (HALF_OPEN):**
```
Client → GET /api/gold-price
       → CircuitBreaker::isAvailable() → true (انتقال به HALF_OPEN)
       → Http::get('https://call5.tgju.org/ajax.json')
       → 200 OK → recordSuccess() → (۱ از ۲ مورد نیاز)
       → درخواست موفق بعدی → recordSuccess() → (۲ از ۲) → ریست به CLOSED
```

---

## کلیدهای Cache

تمام وضعیت‌ها تحت این کلیدهای cache ذخیره می‌شوند:

| کلید | نوع | توضیح |
|------|-----|-------|
| `circuit_breaker:tgju_gold_price:state` | string | وضعیت فعلی (`closed`، `open`، `half_open`) |
| `circuit_breaker:tgju_gold_price:failures` | int | تعداد خطای فعلی |
| `circuit_breaker:tgju_gold_price:last_failed_at` | int | مُهر زمانی Unix آخرین خطا |
| `circuit_breaker:tgju_gold_price:half_open_successes` | int | تعداد موفقیت در حین بررسی HALF_OPEN |

---

## مطالعه بیشتر

- [الگوی Circuit Breaker - تئوری و طراحی](../docs/resilience/circuit-breaker.md) - تئوری جامع شامل استراتژی‌های بازگشتی، نظارت و مثال‌های واقعی از Netflix، Amazon و Uber
- [قضیه CAP](../docs/fundamentals/cap-theorem.md) - مصالحه بنیادی بین سازگاری و دسترس‌پذیری که انگیزه Circuit Breaker است
- [الگوی Saga](../docs/distributed-transactions/saga.md) - نحوه تعامل Circuit Breaker با منطق جبرانی تراکنش‌های توزیع‌شده
