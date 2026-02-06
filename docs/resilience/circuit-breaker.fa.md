# الگوی Circuit Breaker

## 1. مقدمه

الگوی Circuit Breaker یک الگوی طراحی است که در سیستم‌های توزیع‌شده مدرن برای جلوگیری از شکست‌های آبشاری و بهبود انعطاف‌پذیری سیستم استفاده می‌شود. درست مانند یک circuit breaker الکتریکی که سیستم برق خانه شما را از بار اضافی محافظت می‌کند و هنگام عبور جریان بیش از حد، برق را قطع می‌کند، یک circuit breaker نرم‌افزاری نیز برنامه شما را از فراخوانی سرویس‌هایی که احتمالاً با شکست مواجه می‌شوند محافظت می‌کند.

**منشأ و تکامل:**

الگوی Circuit Breaker توسط Michael Nygard در کتاب تاثیرگذار خود در سال 2007 به نام "Release It! Design and Deploy Production-Ready Software" محبوب شد. Martin Fowler بعداً در پست وبلاگ خود در سال 2014 این الگو را گسترش داد و آن را به یکی از پایه‌های معماری [microservices](../architecture/microservices.fa.md) مقاوم تبدیل کرد.

**چرا مهم است:**

در سیستم‌های توزیع‌شده، سرویس‌ها به سرویس‌های دیگر وابسته هستند. زمانی که یک سرویس downstream کند یا پاسخگو نباشد، سرویس‌های upstream می‌توانند منابع (thread، حافظه، اتصالات) را هدر دهند در حالی که منتظر پاسخ‌هایی هستند که ممکن است هرگز نرسند. این می‌تواند منجر به شکست‌های آبشاری شود که در آن مشکلات یک سرویس در کل سیستم موج می‌زند و احتمالاً کل برنامه را از کار می‌اندازد.

**اصل اصلی:**

به جای تلاش مکرر برای فراخوانی یک سرویس خراب (که منابع را هدر می‌دهد و تشخیص شکست را به تاخیر می‌اندازد)، یک circuit breaker شکست‌ها را رصد می‌کند و "مدار را باز می‌کند" تا درخواست‌ها به سرویس‌های خراب را موقتاً مسدود کند، به سرویس خراب زمان برای بازیابی می‌دهد و در عین حال از اتلاف منابع در سرویس فراخوانی‌کننده جلوگیری می‌کند.

## 2. بیان مسئله

### سناریوی شکست آبشاری

یک برنامه تجارت الکترونیک با وابستگی‌های سرویس زیر را در نظر بگیرید:

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

**بدون Circuit Breaker:**

1. درگاه پرداخت خارجی پاسخگو نمی‌شود (مشکل شبکه، بار زیاد، قطعی)
2. Payment Service همچنان تلاش می‌کند به آن دسترسی پیدا کند، با هر درخواست که پس از 30 ثانیه timeout می‌شود
3. Thread های Order Service در انتظار پاسخ Payment Service مسدود می‌شوند
4. Thread pool در Order Service تمام می‌شود (همه thread ها در انتظار)
5. درخواست‌های Frontend Service به Order Service شروع به timeout می‌کنند
6. Thread های Frontend Service مسدود می‌شوند
7. کل برنامه پاسخگو نمی‌شود، حتی برای عملیاتی که به درگاه پرداخت نیاز ندارند

**مشکلات بدون Circuit Breaker:**

- **اتمام منابع:** Thread ها، اتصالات و حافظه در انتظار پاسخ‌ها گرفتار می‌شوند
- **تشخیص کند شکست:** چندین چرخه timeout طول می‌کشد تا متوجه شوید سرویس از کار افتاده است
- **کاهش تجربه کاربری:** کاربران پیش از دریافت پیام خطا، انتظار طولانی را تجربه می‌کنند
- **هدر رفتن محاسبات:** چرخه‌های CPU صرف درخواست‌هایی می‌شود که محکوم به شکست هستند
- **دشواری در بازیابی:** حتی زمانی که سرویس downstream بازیابی می‌شود، سرویس‌های upstream ممکن است آنقدر تحت بار باشند که نتوانند از آن استفاده کنند

طبق [قضیه CAP](../fundamentals/cap-theorem.fa.md)، در طول پارتیشن‌های شبکه، باید بین consistency و availability انتخاب کنیم. Circuit breaker ها به ما کمک می‌کنند تا availability را با fail fast زمانی که consistency با یک سرویس downstream قابل دستیابی نیست، حفظ کنیم.

## 3. سه حالت

الگوی Circuit Breaker به عنوان یک state machine با سه حالت عمل می‌کند:

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

### 3.1 حالت CLOSED (عملیات عادی)

**ویژگی‌ها:**
- همه درخواست‌ها به سرویس downstream منتقل می‌شوند
- Circuit breaker شکست‌ها را رصد می‌کند (timeout، exception، خطاهای 5xx)
- یک شمارنده یا sliding window از شکست‌های اخیر را نگه می‌دارد
- این حالت پیش‌فرض و مطلوب است

**رفتار:**
```
Client ───request──▶ Circuit Breaker ───request──▶ Service
       ◀──response─                    ◀─response──

Success: افزایش شمارنده موفقیت، reset شمارنده شکست
Failure: افزایش شمارنده شکست
```

**انتقال حالت:**
زمانی که تعداد شکست یا نرخ شکست از آستانه پیکربندی‌شده در یک بازه زمانی فراتر رود، مدار به OPEN منتقل می‌شود.

**مثال پیکربندی آستانه:**
- شکست پس از 5 شکست متوالی، یا
- شکست زمانی که 50٪ از درخواست‌ها در یک بازه 10 ثانیه‌ای شکست بخورند، یا
- شکست زمانی که 10 شکست در یک sliding window 30 ثانیه‌ای رخ دهد

### 3.2 حالت OPEN (Fail Fast)

**ویژگی‌ها:**
- همه درخواست‌ها بلافاصله بدون فراخوانی سرویس downstream شکست می‌خورند
- هیچ منبعی روی درخواست‌های محکوم به شکست هدر نمی‌رود
- یک تایمر timeout شروع می‌شود (مثلاً 60 ثانیه)
- سرویس خراب زمان برای بازیابی بدون بار اضافی دریافت می‌کند

**رفتار:**
```
Client ───request──▶ Circuit Breaker ──╳──▶ Service (فراخوانی نمی‌شود)
       ◀───error───

شکست فوری با CircuitBreakerOpenException
```

**انتقال حالت:**
پس از گذشت دوره timeout، مدار به HALF-OPEN منتقل می‌شود تا بررسی کند آیا سرویس downstream بازیابی شده است یا خیر.

**چرا Fail Fast؟**
- بازخورد فوری به کاربر (بدون انتظار برای timeout)
- آزادسازی منابع برای عملیات دیگر
- کاهش بار روی سرویس خراب
- امکان استفاده از مکانیزم‌های fallback

### 3.3 حالت HALF-OPEN (تست بازیابی)

**ویژگی‌ها:**
- تعداد محدودی از درخواست‌های تست اجازه عبور دارند
- به عنوان "canary" عمل می‌کند تا بررسی کند آیا سرویس بازیابی شده است
- بیشتر درخواست‌ها در این دوره هنوز به سرعت شکست می‌خورند
- حالت انتقالی کوتاه

**رفتار:**
```
Test Request 1 ───request──▶ Circuit Breaker ───request──▶ Service
               ◀──response─                    ◀─response──

Test Request 2 ───request──▶ Circuit Breaker ───request──▶ Service
               ◀──response─                    ◀─response──

Other Requests ───request──▶ Circuit Breaker ──╳──▶ Service
               ◀───error───                     (هنوز مسدود)
```

**انتقال حالت‌ها:**

1. **به CLOSED (مسیر موفقیت):**
   - اگر درخواست‌های تست موفق باشند (مثلاً 3 موفقیت متوالی)
   - Circuit به طور کامل بسته می‌شود و عملیات عادی از سر گرفته می‌شود
   - شمارنده‌ها reset می‌شوند

2. **به OPEN (مسیر شکست):**
   - اگر درخواست‌های تست شکست بخورند
   - Circuit بلافاصله دوباره باز می‌شود
   - تایمر timeout دوباره شروع می‌شود
   - سرویس به زمان بازیابی بیشتری نیاز دارد

**مثال‌های پیکربندی:**
- اجازه 3 درخواست تست در HALF-OPEN
- اگر 3/3 موفق باشند → CLOSED
- اگر هر کدام شکست بخورند → OPEN

## 4. چگونه کار می‌کند

### 4.1 مسیر موفقیت (عملیات عادی)

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

### 4.2 مسیر شکست (تشخیص شکست‌ها)

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

### 4.3 مسیر بازیابی (بهبود سرویس)

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

## 5. جزئیات پیاده‌سازی

### 5.1 پارامترهای کلیدی پیکربندی

```
┌─────────────────────────────────────────────────────────┐
│            Circuit Breaker Configuration                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Failure Threshold                                      │
│  ├─ Count Based: 5 شکست متوالی                         │
│  ├─ Rate Based: 50% نرخ شکست                           │
│  └─ Time Window: 10 ثانیه                              │
│                                                         │
│  Open State Duration                                    │
│  └─ Sleep Window: 60 ثانیه                             │
│                                                         │
│  Half-Open Test Requests                               │
│  └─ Permitted Calls: 3 درخواست                         │
│                                                         │
│  Success Threshold (Half-Open → Closed)                │
│  └─ Required Successes: 3 از 3                         │
│                                                         │
│  چه چیزی به عنوان شکست محسوب می‌شود؟                 │
│  ├─ Exceptions: IOException, TimeoutException          │
│  ├─ HTTP Status: 500, 502, 503, 504                   │
│  ├─ Timeouts: > 3 ثانیه                                │
│  └─ Custom Predicates: بر اساس منطق کسب‌وکار          │
│                                                         │
│  چه چیزی به عنوان شکست محسوب نمی‌شود؟                │
│  ├─ HTTP 4xx (خطاهای کلاینت)                          │
│  ├─ Expected exceptions (ValidationException)          │
│  └─ شکست‌های منطق کسب‌وکار                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 5.2 پیاده‌سازی Sliding Window

دو رویکرد رایج برای ردیابی شکست‌ها وجود دارد:

**Sliding Window مبتنی بر شمارش:**
```
Time: ──────────────────────────────────────▶
       [S] [F] [F] [S] [F] [F] [F]

اندازه Window: 10 درخواست آخر
فعلی: 5 شکست از 7 درخواست = 71% نرخ شکست
آستانه: 50%
عمل: TRIP TO OPEN
```

**Sliding Window مبتنی بر زمان:**
```
Time: 0s────10s────20s────30s────40s────50s────60s────▶
      [SSF] [FFF] [FSF] [FFS] [SSS] [SSS] [FFF]
                                    ↑
                                    Now (50s)

Window: 30 ثانیه آخر
درخواست‌ها در window (20s-50s): [FFS][SSS][SSS] = 2 شکست / 9 درخواست
نرخ شکست: 22%
آستانه: 50%
عمل: REMAIN CLOSED
```

### 5.3 Pseudocode پیاده‌سازی

```java
class CircuitBreaker {
    enum State { CLOSED, OPEN, HALF_OPEN }

    private State state = CLOSED
    private int failureCount = 0
    private int successCount = 0
    private long openedAt = 0

    // Configuration
    private final int failureThreshold = 5
    private final long openDuration = 60000  // 60 ثانیه
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

### 5.4 ملاحظات Thread Safety

Circuit breaker ها باید thread-safe باشند زیرا بین چندین thread به اشتراک گذاشته می‌شوند:

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

**الزامات پیاده‌سازی:**
- استفاده از عملیات atomic برای شمارنده‌ها (AtomicInteger, AtomicLong)
- همگام‌سازی انتقال‌های حالت
- در نظر گرفتن ساختارهای داده lock-free برای throughput بالا
- استفاده از ساختارهای داده همزمان برای sliding window ها (ConcurrentLinkedQueue)

## 6. استراتژی‌های Fallback

زمانی که یک circuit breaker باز می‌شود، سرویس فراخوانی‌کننده به یک استراتژی برای مدیریت شکست به صورت مناسب نیاز دارد. صرفاً برگرداندن یک خطا اغلب بهترین تجربه کاربری نیست.

### 6.1 مقایسه گزینه‌های Fallback

| استراتژی | مورد استفاده | مزایا | معایب | مثال |
|----------|----------|------|------|---------|
| **مقدار پیش‌فرض/ثابت** | داده‌های غیرحیاتی | ساده، سریع، قابل اطمینان | ممکن است داده‌های قدیمی/عمومی ارائه دهد | برگرداندن لیست خالی، آواتار پیش‌فرض |
| **پاسخ Cache شده** | داده‌هایی که به آرامی تغییر می‌کنند | داده واقعی ارائه می‌دهد، UX بهتر | Cache ممکن است قدیمی یا نامعتبر باشد | کاتالوگ محصول، پروفایل کاربر |
| **سرویس جایگزین** | عملیات حیاتی | عملکرد را حفظ می‌کند | پیچیدگی، هزینه احتمالی | درگاه پرداخت ثانویه |
| **کاهش تدریجی** | ویژگی‌ها با وابستگی‌ها | ویژگی‌های اصلی هنوز کار می‌کنند | عملکرد کاهش یافته | غیرفعال کردن توصیه‌ها |
| **صف برای بعد** | عملیات async | بدون از دست دادن داده | پردازش تاخیری، مدیریت صف | کارهای پس‌زمینه، ایمیل‌ها |
| **اطلاع‌رسانی کاربر** | شکست‌های شفاف | ارتباط صادقانه | UX ضعیف در صورت استفاده بیش از حد | "ویژگی موقتاً در دسترس نیست" |

### 6.2 مثال پیاده‌سازی Fallback

```
جریان درخواست با Fallback:

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

### 6.3 مثال‌های واقعی Fallback

**توصیه‌های محصول تجارت الکترونیک:**
```
اولیه: سرویس توصیه ML بلادرنگ
Fallback 1: آیتم‌های محبوب از پیش محاسبه شده از cache
Fallback 2: پرفروش‌ترین‌های مبتنی بر دسته‌بندی
Fallback 3: مخفی کردن بخش توصیه
```

**سرویس پروفایل کاربر:**
```
اولیه: پایگاه داده سرویس کاربر
Fallback 1: Cache ی Redis (TTL 5 دقیقه)
Fallback 2: پروفایل کاربر ناشناس پیش‌فرض
Fallback 3: اجازه عملکرد محدود بدون پروفایل
```

**پردازش پرداخت:**
```
اولیه: درگاه پرداخت ترجیحی (Stripe)
Fallback 1: درگاه پرداخت ثانویه (PayPal)
Fallback 2: صف پرداخت برای تلاش مجدد
Fallback 3: اطلاع‌رسانی به کاربر، ذخیره سبد خرید برای بعد
```

**سرویس تصویر:**
```
اولیه: CDN با آپلودهای کاربر
Fallback 1: نسخه Cache شده از edge
Fallback 2: تصویر placeholder
Fallback 3: حالت فقط متن
```

## 7. Circuit Breaker در مقابل الگوی Retry

هر دو الگو شکست‌ها را در سیستم‌های توزیع‌شده مدیریت می‌کنند، اما اهداف متفاوتی دارند و اغلب باید با هم استفاده شوند.

### 7.1 جدول مقایسه

| جنبه | Circuit Breaker | الگوی Retry |
|--------|----------------|---------------|
| **هدف** | جلوگیری از شکست‌های آبشاری | غلبه بر شکست‌های موقت |
| **چه زمانی استفاده شود** | شکست‌های پایدار/مداوم | مشکلات شبکه موقت |
| **رفتار** | پس از آستانه به سرعت شکست می‌خورد | با backoff به تلاش ادامه می‌دهد |
| **تأثیر منابع** | منابع را حفظ می‌کند | در صورت استفاده نادرست می‌تواند منابع را هدر دهد |
| **زمان بازیابی** | به سرویس زمان برای بهبود می‌دهد | تلاش‌های فوری مجدد |
| **تشخیص شکست** | الگوهای شکست را رصد می‌کند | به شکست‌های فردی واکنش نشان می‌دهد |
| **تأثیر کاربر** | خطا/fallback فوری | تأخیر را اضافه می‌کند (retry ها زمان می‌برند) |
| **بهترین برای** | محافظت در برابر بار بیش از حد | مدیریت خطاهای موقت |

### 7.2 چه زمانی از کدام استفاده کنیم

```
درخت تصمیم:

آیا سرویس کاملاً از کار افتاده یا پاسخگو نیست؟
│
├─ بله → Circuit Breaker
│         (منابع را روی retry هدر ندهید)
│
└─ خیر → آیا این یک خطای موقت است؟
        │
        ├─ بله → الگوی Retry
        │         (مشکل شبکه موقت، محدودیت نرخ)
        │
        └─ خیر → آیا این یک خطای کلاینت است (4xx)؟
                │
                ├─ بله → بدون retry، بدون circuit breaker
                │         (درخواست کلاینت را اصلاح کنید)
                │
                └─ خیر → از هر دو الگو با هم استفاده کنید
                        (Retry با circuit breaker)
```

### 7.3 ترکیب Circuit Breaker و Retry

مقاوم‌ترین سیستم‌ها از هر دو الگو با هم استفاده می‌کنند:

```
┌────────────────────────────────────────────────┐
│              جریان درخواست                     │
├────────────────────────────────────────────────┤
│                                                │
│  1. درخواست می‌رسد                            │
│     ↓                                          │
│  2. بررسی Circuit Breaker                      │
│     ├─ باز؟ → Fail fast (بدون retry)           │
│     └─ بسته؟ → ادامه                           │
│     ↓                                          │
│  3. فراخوانی سرویس با Retry                    │
│     ├─ تلاش 1 → Timeout                        │
│     ├─ صبر 100ms (exponential backoff)         │
│     ├─ تلاش 2 → Timeout                        │
│     ├─ صبر 200ms                               │
│     ├─ تلاش 3 → Timeout                        │
│     └─ تسلیم شدن                               │
│     ↓                                          │
│  4. گزارش شکست به Circuit Breaker              │
│     (افزایش شمارنده شکست)                      │
│     ↓                                          │
│  5. اگر از آستانه فراتر رفت → باز کردن circuit  │
│     ↓                                          │
│  6. درخواست‌های آینده به سرعت شکست می‌خورند   │
│                                                │
└────────────────────────────────────────────────┘
```

**مثال پیکربندی:**
```yaml
resilience:
  circuitBreaker:
    failureThreshold: 5        # باز شدن پس از 5 تلاش ناموفق
    openDuration: 60s          # 60 ثانیه باز بماند
    halfOpenRequests: 3        # تست با 3 درخواست

  retry:
    maxAttempts: 3             # تا 3 بار تلاش کنید
    waitDuration: 100ms        # انتظار اولیه
    exponentialBackoff: 2      # دو برابر کردن انتظار هر بار
    retryOnExceptions:
      - TimeoutException
      - ConnectException
```

### 7.4 ضد الگوهایی که باید از آنها اجتناب کرد

**انجام ندهید:**
- Retry بی‌نهایت (از max attempts استفاده کنید)
- Retry بدون exponential backoff (سرویس را می‌کوبد)
- استفاده از retry های تهاجمی بدون circuit breaker (شکست‌ها را تقویت می‌کند)
- Retry روی همه خطاها (خطاهای 4xx کلاینت را retry نکنید)
- تنظیم timeout ی retry طولانی‌تر از آستانه circuit breaker

## 8. Circuit Breaker در عمل

### 8.1 کتابخانه‌ها و فریمورک‌های محبوب

#### Resilience4j (Java مدرن)

```java
// Configuration
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)                  // 50% نرخ شکست
    .slidingWindowSize(10)                     // 10 فراخوانی آخر
    .waitDurationInOpenState(Duration.ofSeconds(60))
    .permittedNumberOfCallsInHalfOpenState(3)
    .build();

// ایجاد circuit breaker
CircuitBreaker circuitBreaker = CircuitBreaker.of("paymentService", config);

// استفاده با فراخوانی سرویس
Supplier<PaymentResponse> supplier = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> paymentService.processPayment(request));

// اجرا
Try<PaymentResponse> result = Try.ofSupplier(supplier)
    .recover(CallNotPermittedException.class, fallbackPaymentResponse());
```

#### Netflix Hystrix (قدیمی، اکنون در حالت نگهداری)

```java
// دستور Hystrix (قدیمی اما آموزنده)
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
        // منطق Fallback
        return PaymentResponse.queued();
    }
}

// استفاده
PaymentResponse response = new PaymentCommand(request).execute();
```

**نکته در مورد Hystrix:** Netflix در سال 2018 اعلام کرد که Hystrix در حالت نگهداری است. آنها توصیه می‌کنند برای پروژه‌های جدید از Resilience4j یا راه‌حل‌های service mesh مانند Istio استفاده شود.

#### Polly (.NET)

```csharp
// پیکربندی policy ی circuit breaker
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

// استفاده از policy
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
        // صف برای پردازش بعدی
        return PaymentResponse.queued();
    }
}
```

### 8.2 پیاده‌سازی Service Mesh (Istio)

در [معماری microservices](../architecture/microservices.fa.md)، circuit breaker ها می‌توانند در سطح زیرساخت با استفاده از یک service mesh مانند Istio پیاده‌سازی شوند:

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

**مزایای رویکرد Service Mesh:**
- مستقل از زبان (با هر سرویسی کار می‌کند)
- پیکربندی متمرکز
- قابلیت مشاهده داخلی
- بدون نیاز به تغییرات کد
- رفتار یکنواخت در تمام سرویس‌ها

**مقایسه: کتابخانه در مقابل Service Mesh**

| جنبه | کتابخانه (Resilience4j) | Service Mesh (Istio) |
|--------|------------------------|----------------------|
| **تغییرات کد** | مورد نیاز | مورد نیاز نیست |
| **پشتیبانی زبان** | یک زبان | هر زبانی |
| **دانه‌بندی** | کنترل دقیق | کنترل درشت‌تر |
| **پیکربندی** | در برنامه | YAML خارجی |
| **منحنی یادگیری** | متوسط | شیب دار |
| **سربار** | حداقل | زیرساخت اضافی |
| **قابلیت مشاهده** | یکپارچه‌سازی سفارشی | داخلی |
| **بهترین برای** | بهینه‌سازی سرویس واحد | Microservices چندزبانه |

### 8.3 یکپارچه‌سازی با API Gateway

در سطح [API Gateway](../architecture/api-gateway.fa.md)، circuit breaker ها می‌توانند در برابر شکست‌های سرویس downstream محافظت کنند:

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

**مثال Kong API Gateway:**

```yaml
plugins:
  - name: circuit-breaker
    config:
      failure_threshold: 5
      window_duration: 30
      state_open_duration: 60
      notify_interval: 10
```

## 9. نظارت و هشدار

نظارت مؤثر بر circuit breaker برای درک سلامت سیستم و پاسخ به حوادث بسیار مهم است.

### 9.1 معیارهای کلیدی برای ردیابی

```
┌─────────────────────────────────────────────────────────┐
│              معیارهای Circuit Breaker                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  معیارهای حالت                                          │
│  ├─ حالت فعلی (Closed/Open/Half-Open)                  │
│  ├─ زمان در حالت فعلی                                  │
│  ├─ تعداد انتقال حالت                                  │
│  └─ Timestamp آخرین تغییر حالت                         │
│                                                         │
│  معیارهای فراخوانی                                      │
│  ├─ کل فراخوانی‌ها                                     │
│  ├─ فراخوانی‌های موفق                                  │
│  ├─ فراخوانی‌های ناموفق                                │
│  ├─ فراخوانی‌های رد شده (هنگام باز بودن)              │
│  └─ مدت زمان فراخوانی (p50, p95, p99)                 │
│                                                         │
│  معیارهای شکست                                          │
│  ├─ نرخ شکست (%)                                       │
│  ├─ شکست‌های متوالی                                    │
│  ├─ انواع شکست (timeout, exception, 5xx)              │
│  └─ زمان از آخرین شکست                                │
│                                                         │
│  معیارهای سلامت                                         │
│  ├─ نرخ موفقیت تست Half-Open                           │
│  ├─ زمان بازیابی (مدت زمان open → closed)             │
│  └─ پایداری Circuit (تشخیص flapping)                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 9.2 مثال داشبورد

```
داشبورد Circuit Breaker - سرویس پرداخت
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

حالت فعلی: CLOSED ✓              آخرین تغییر: 2 ساعت پیش

نرخ موفقیت: ████████████████░░ 89%   (هدف: >95%)
حجم فراخوانی:  1,247 req/min            (محدوده عادی: 1000-1500)

تاریخچه حالت (24 ساعت گذشته):
┌─────────────────────────────────────────────────────────┐
│ Closed: ████████████████████████░░░░░░░░░░░░░░░░░ 70%  │
│ Open:   ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 15%  │
│ Half:   ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 5%   │
│ N/A:    ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 10%  │
└─────────────────────────────────────────────────────────┘

تأخیر فراخوانی (ms):
  p50: 42ms   p95: 125ms   p99: 380ms   Max: 2,156ms

حوادث اخیر:
• 14:23 - Circuit باز شد (5 timeout متوالی)
• 14:24 - Circuit نیمه‌باز شد (timeout منقضی شد)
• 14:24 - Circuit بسته شد (3 درخواست تست موفق شد)

انواع شکست (ساعت گذشته):
• Timeout ها:        12  ████░░░░░░
• خطاهای 5xx:        5   ██░░░░░░░░
• شبکه:              3   █░░░░░░░░░
```

### 9.3 معیارهای Prometheus

```yaml
# حالت circuit breaker (gauge: 0=closed, 1=open, 2=half-open)
circuit_breaker_state{service="payment"} 0

# شمارنده کل فراخوانی‌ها
circuit_breaker_calls_total{service="payment",result="success"} 45234
circuit_breaker_calls_total{service="payment",result="failure"} 892
circuit_breaker_calls_total{service="payment",result="rejected"} 145

# هیستوگرام مدت زمان فراخوانی
circuit_breaker_call_duration_seconds_bucket{service="payment",le="0.1"} 42000
circuit_breaker_call_duration_seconds_bucket{service="payment",le="0.5"} 44500
circuit_breaker_call_duration_seconds_bucket{service="payment",le="1.0"} 45800

# شمارنده انتقال حالت
circuit_breaker_state_transitions_total{service="payment",from="closed",to="open"} 12
circuit_breaker_state_transitions_total{service="payment",from="open",to="half_open"} 12
circuit_breaker_state_transitions_total{service="payment",from="half_open",to="closed"} 10
```

### 9.4 تعاریف هشدار

**هشدارهای حیاتی:**

```yaml
# Circuit برای مدت طولانی باز است
alert: CircuitBreakerOpenTooLong
expr: circuit_breaker_state{state="open"} == 1 for 5m
severity: critical
message: "Circuit breaker برای {{$labels.service}} بیش از 5 دقیقه باز بوده است"

# نرخ بالای رد
alert: CircuitBreakerHighRejectionRate
expr: rate(circuit_breaker_calls_total{result="rejected"}[5m]) > 10
severity: critical
message: "Circuit breaker بیش از 10 req/s برای {{$labels.service}} رد می‌کند"
```

**هشدارهای هشداردهنده:**

```yaml
# Circuit در حال flapping است (باز/بسته شدن مکرر)
alert: CircuitBreakerFlapping
expr: rate(circuit_breaker_state_transitions_total[10m]) > 0.5
severity: warning
message: "Circuit breaker برای {{$labels.service}} در حال flapping است (>5 انتقال در 10 دقیقه)"

# نرخ شکست بالا نزدیک به آستانه
alert: CircuitBreakerFailureRateHigh
expr: rate(circuit_breaker_calls_total{result="failure"}[5m]) > 5
severity: warning
message: "نرخ شکست برای {{$labels.service}} بالا است (نزدیک به آستانه circuit breaker)"
```

### 9.5 یکپارچه‌سازی Distributed Tracing

رویدادهای circuit breaker باید در trace های توزیع‌شده قابل مشاهده باشند:

```
Trace: POST /api/orders/{id}/checkout
├─ api-gateway: 245ms
│  ├─ auth-service: 23ms [OK]
│  ├─ order-service: 45ms [OK]
│  └─ payment-service: 0ms [CIRCUIT_BREAKER_OPEN]
│     └─ fallback: cache_fallback [OK]
└─ Response: 200 OK (با وضعیت پرداخت cache شده)

ویژگی‌های Span:
  circuit_breaker.state: open
  circuit_breaker.fallback_used: true
  circuit_breaker.fallback_type: cache
```

## 10. مبادلات

### 10.1 مزایا

**جلوگیری از شکست‌های آبشاری:**
- توقف انتشار شکست در سیستم
- محافظت از سرویس‌های upstream از اتمام منابع
- حفظ در دسترس بودن جزئی در طول قطعی‌ها

**بهبود تجربه کاربر:**
- شکست‌های سریع به جای timeout های طولانی
- امکان پاسخ‌های fallback
- کاهش تأخیر درک شده در طول شکست‌ها

**محافظت از منابع:**
- آزادسازی thread ها/اتصالاتی که هدر می‌رفتند
- کاهش بار روی سرویس‌های خراب (زمان برای بازیابی می‌دهد)
- جلوگیری از اتمام thread pool

**قابلیت مشاهده:**
- سیگنال واضح زمانی که سرویس‌ها ناسالم هستند
- معیارها برای نظارت بر سلامت سیستم
- سیستم هشدار اولیه برای مسائل آبشاری

**کاهش تدریجی:**
- سیستم تا حدی عملکرد می‌کند
- مسیرهای حیاتی می‌توانند به کار ادامه دهند
- بهتر از شکست کامل سیستم

### 10.2 معایب

**پیچیدگی:**
- مؤلفه اضافی برای پیکربندی و نگهداری
- State machine پیچیدگی را به اشکال‌زدایی اضافه می‌کند
- نیاز به درک زمانی که circuit باز است در مقابل شکست‌های واقعی سرویس

**چالش‌های پیکربندی:**
- انتخاب آستانه‌های مناسب دشوار است
- خیلی حساس = fail-fast غیرضروری
- خیلی نرم = جلوگیری از شکست‌های آبشاری نمی‌کند
- سرویس‌های مختلف ممکن است به پیکربندی‌های متفاوت نیاز داشته باشند

**مسئولیت Fallback:**
- Circuit breaker به تنهایی کافی نیست
- نیاز به پیاده‌سازی fallback های معنادار
- منطق Fallback می‌تواند پیچیده باشد

**پیچیدگی تست:**
- تست همه انتقال‌های حالت دشوار است
- نیاز به شبیه‌سازی سناریوهای مختلف شکست
- تست یکپارچه‌سازی پیچیده‌تر می‌شود

**احتمال False Positive:**
- افزایش کوتاه خطاها می‌تواند circuit را بی‌جهت باز کند
- در طول استقرار، circuit ممکن است موقتاً باز شود
- می‌تواند مشکلات واقعی را اگر fallback ها همیشه موفق باشند پنهان کند

**سربار هماهنگی:**
- در سیستم‌های توزیع‌شده، چندین circuit breaker ممکن است trip شوند
- نمای جهانی از سلامت سیستم وجود ندارد
- می‌تواند منجر به باز شدن آبشاری circuit breaker ها شود

### 10.3 چه زمانی از Circuit Breaker استفاده نکنیم

**جریان‌های همزمان کاربر:**
```
استفاده نکنید برای: کاربر روی "ذخیره پروفایل" کلیک می‌کند → باید فوراً ذخیره شود
به جای آن استفاده کنید: به‌روزرسانی‌های UI خوش‌بینانه، ماندگاری محلی، retry با بازخورد
```

**اتصالات پایگاه داده:**
```
استفاده نکنید circuit breaker بین برنامه و پایگاه داده اصلی
چرا: شکست پایگاه داده معمولاً به این معنی است که برنامه نمی‌تواند کار کند
به جای آن استفاده کنید: Connection pooling، read replica ها، کلاسترینگ پایگاه داده
```

**فراخوانی‌های سرویس داخلی (همان مرکز داده):**
```
ممکن است نیاز نباشد برای: سرویس A → سرویس B (همان مرکز داده، تأخیر کم)
چرا: شبکه قابل اعتماد است، شکست‌ها معمولاً موقت نیستند
در نظر بگیرید: مدیریت مستقیم شکست ممکن است کافی باشد
```

**سرویس‌های با ترافیک کم:**
```
استفاده نکنید برای: پنل ادمین با 1 req/minute
چرا: ترافیک کافی برای ایجاد الگوهای شکست وجود ندارد
انتقال‌های حالت نامنظم خواهند بود
```

## 11. الگوهای مرتبط

Circuit breaker ها به عنوان بخشی از یک استراتژی جامع انعطاف‌پذیری بهترین عملکرد را دارند.

### 11.1 الگوی Bulkhead

منابع را برای جلوگیری از شکست کامل سیستم جداسازی می‌کند:

```
بدون Bulkhead:              با Bulkhead:
┌──────────────────────┐       ┌──────────────────────┐
│   Thread Pool        │       │  Thread Pools        │
│   (50 threads)       │       │                      │
│                      │       │ ┌────────────────┐   │
│  [همه thread ها      │       │ │ Payment (10)   │   │
│   توسط فراخوانی‌های  │       │ └────────────────┘   │
│   کند سرویس پرداخت   │       │ ┌────────────────┐   │
│   استفاده شده‌اند]   │       │ │ Orders (20)    │   │
│                      │       │ └────────────────┘   │
│  سرویس‌های دیگر      │       │ ┌────────────────┐   │
│  نمی‌توانند thread   │       │ │ Other (20)     │   │
│  بگیرند              │       │ └────────────────┘   │
└──────────────────────┘       │                      │
                               │ Payment شکست می‌خورد │
                               │ اما بقیه کار می‌کنند │
                               └──────────────────────┘
```

**استفاده از Circuit Breaker + Bulkhead:**
- Bulkhead مصرف منابع را برای هر سرویس محدود می‌کند
- Circuit breaker زمانی که سرویس از کار افتاده به سرعت شکست می‌خورد
- با هم: حداکثر جداسازی و محافظت

### 11.2 الگوی Retry

در بخش 7 پوشش داده شد، اما نکات کلیدی:

```
Request → Retry → Circuit Breaker → Service
          (3 تلاش)  (fail fast اگر باز باشد)
```

**استراتژی یکپارچه‌سازی:**
1. Retry شکست‌های موقت را مدیریت می‌کند (مشکلات شبکه)
2. Circuit breaker از طوفان retry در طول قطعی‌ها جلوگیری می‌کند
3. استفاده از exponential backoff با jitter در retry ها
4. Circuit breaker نتایج retry را نظارت می‌کند

### 11.3 الگوی Timeout

Circuit breaker ها به timeout ها وابسته هستند:

```
Request ──────────────────────────────────▶ Service
        │                                  │
        │ Timeout: 3 ثانیه                 │
        │                                  ╳ (کند/متوقف شده)
        ╳
        │
        └─────▶ گزارش timeout به Circuit Breaker
```

**پیکربندی Timeout:**
- تنظیم timeout درخواست کوتاه‌تر از بازه ارزیابی circuit breaker
- مثال: timeout 3 ثانیه‌ای، بازه circuit breaker 10 ثانیه‌ای
- بدون timeout، circuit breaker نمی‌تواند سرویس‌های کند را تشخیص دهد

### 11.4 الگوی Health Check

نظارت سلامت خارجی برای تصمیمات circuit breaker:

```
┌─────────────────┐         ┌─────────────────┐
│  Health Check   │────────▶│    Service      │
│  Endpoint       │         │  /health        │
└────────┬────────┘         └─────────────────┘
         │
         │ 200 OK = سالم
         │ 503 = ناسالم
         │
         ▼
┌─────────────────┐
│ Circuit Breaker │
│  (می‌تواند از   │
│   وضعیت سلامت   │
│   به عنوان ورودی│
│   استفاده کند)  │
└─────────────────┘
```

### 11.5 الگوی Fallback

بخش 6 را برای جزئیات ببینید. یکپارچه‌سازی کلیدی:

```
Circuit Breaker ──open──▶ استراتژی Fallback
                           ├─ Cache
                           ├─ مقدار پیش‌فرض
                           ├─ سرویس جایگزین
                           └─ کاهش تدریجی
```

### 11.6 یکپارچه‌سازی با تراکنش‌های توزیع‌شده

هنگام پیاده‌سازی [الگوی Saga](../distributed-transactions/saga.fa.md)، circuit breaker ها بر منطق جبران تأثیر می‌گذارند:

```
مرحله 1 Saga: رزرو موجودی ✓
مرحله 2 Saga: پردازش پرداخت ──▶ Circuit باز!
             │
             ├─ Fallback: صف پرداخت برای بعد
             └─ جبران: آزادسازی رزرو موجودی
```

**ملاحظات:**
- باز بودن Circuit breaker همیشه به معنی شکست saga نیست
- ممکن است مراحل saga را برای اجرای بعدی به صف بزنید
- منطق جبران باید حالت‌های circuit breaker را مدیریت کند

### 11.7 یکپارچه‌سازی Event-Driven Architecture

در [معماری event-driven](../event-driven/event-driven-architecture.fa.md)، circuit breaker ها از event publisher ها محافظت می‌کنند:

```
Service A ──publish event──▶ Circuit Breaker ──▶ Message Queue

اگر صف از کار افتاده:
  ├─ Circuit باز می‌شود
  ├─ رویدادها را به صورت محلی به صف بزنید
  └─ Retry زمانی که circuit بسته می‌شود
```

## 12. مثال‌های واقعی

### 12.1 داستان Netflix Hystrix

Netflix پیشگام circuit breaker ها در مقیاس بود و میلیاردها درخواست در روز را پردازش می‌کرد.

**مشکل:**
در سال 2011، Netflix یک قطعی بزرگ را تجربه کرد زمانی که شکست یک سرویس واحد در کل پلتفرم استریم آنها آبشاری شد. وقتی سرویس توصیه آنها کند شد، همه thread های موجود در API gateway را مصرف کرد و کل سایت را از کار انداخت.

**راه‌حل:**
Netflix Hystrix را ایجاد کرد، یک کتابخانه تحمل تأخیر و خطا:

```
Netflix API Gateway (Zuul)
├─ Hystrix: User Service
├─ Hystrix: Recommendation Service  ← Circuit در طول شکست باز شد
├─ Hystrix: Playback Service
└─ Hystrix: Rating Service

زمانی که سرویس توصیه شکست خورد:
✓ Circuit پس از 20 شکست در 10 ثانیه باز شد
✓ به محتوای محبوب/ترند Fallback زد
✓ سرویس‌های دیگر به طور عادی به کار ادامه دادند
✓ از قطعی 3 ساعته جلوگیری کرد، به کاهش 3 دقیقه‌ای کاهش یافت
```

**معیارهای کلیدی:**
- بیش از 200 میلیارد دستور Hystrix در روز
- میانگین زمان trip circuit breaker: 45 ثانیه
- جلوگیری از حداقل 10+ قطعی بزرگ در سال
- امکان در دسترس بودن 4-نه (99.99٪)

**درس‌های آموخته شده:**
1. پیش‌فرض همه فراخوانی‌های سرویس به fail-fast با circuit breaker ها
2. نظارت بر معیارهای circuit breaker در داشبوردهای بلادرنگ
3. برنامه‌ریزی fallback های معنادار برای همه سرویس‌های حیاتی
4. تست رفتار circuit breaker در تولید با chaos engineering

**تکامل:**
در سال 2018، Netflix از Hystrix به service mesh (Envoy/Istio) برای circuit breaking منتقل شد، زیرا به هزاران microservice مقیاس یافتند.

### 12.2 پلتفرم خرده‌فروشی Amazon

Amazon به طور گسترده از circuit breaker ها در پلتفرم خرده‌فروشی خود استفاده می‌کند:

**مثال صفحه محصول:**
```
ترکیب صفحه محصول:
┌────────────────────────────────────────┐
│ جزئیات محصول (CB: primary DB)         │ ← Fallback: Cache
├────────────────────────────────────────┤
│ قیمت و در دسترس بودن (CB: inventory)  │ ← Fallback: "بررسی موجودی"
├────────────────────────────────────────┤
│ توصیه‌ها (CB: ML service)              │ ← Fallback: پرفروش‌ترین‌ها
├────────────────────────────────────────┤
│ نظرات (CB: review service)             │ ← Fallback: مخفی کردن بخش
├────────────────────────────────────────┤
│ Q&A (CB: Q&A service)                  │ ← Fallback: مخفی کردن بخش
└────────────────────────────────────────┘

اگر سرویس توصیه شکست بخورد:
- Circuit breaker باز می‌شود
- صفحه هنوز در <200ms بارگیری می‌شود
- به جای شخصی‌سازی شده پرفروش‌ترین‌ها را نشان می‌دهد
- کاربر هنوز می‌تواند خرید کند
```

**انعطاف‌پذیری Black Friday:**
در طول رویدادهای خرید پیک، circuit breaker های Amazon:
- به طور خودکار ویژگی‌های غیرضروری را کاهش می‌دهند
- عملکرد اصلی checkout را حفظ می‌کنند
- از شکست‌های آبشاری ناشی از افزایش ترافیک جلوگیری می‌کنند
- بازیابی تدریجی را با کاهش بار امکان‌پذیر می‌کنند

### 12.3 تطبیق سواری Uber

Uber از circuit breaker ها در سیستم تطبیق سواری بلادرنگ خود استفاده می‌کند:

```
جریان درخواست سوار:
┌────────────┐
│   Rider    │
│   App      │
└─────┬──────┘
      │
      ▼
┌─────────────────────────────────────┐
│  API Gateway (Circuit Breakers)     │
├─────────────────────────────────────┤
│  CB: Dispatch Service (اولیه)       │──▶ یافتن نزدیک‌ترین رانندگان
│    Fallback: اعزام سراسری منطقه     │
│                                     │
│  CB: Pricing Service                │──▶ محاسبه کرایه
│    Fallback: جدول قیمت‌گذاری پیش‌فرض│
│                                     │
│  CB: ETA Service                    │──▶ تخمین زمان رسیدن
│    Fallback: تقریب جغرافیایی         │
└─────────────────────────────────────┘
```

**سناریوی شکست:**
1. سرویس اعزام کند می‌شود (مشکل پایگاه داده)
2. Circuit breaker پس از 5 پاسخ کند متوالی trip می‌شود
3. به اعزام سراسری منطقه Fallback می‌زند (کمتر بهینه، اما کار می‌کند)
4. سواری‌ها همچنان تطبیق داده می‌شوند (زمان انتظار کمی طولانی‌تر)
5. سرویس بازیابی می‌شود، circuit بسته می‌شود، تطبیق بهینه از سر گرفته می‌شود

**تأثیر:**
- از قطعی کامل سیستم تطبیق جلوگیری شد
- ظرفیت عادی 80٪ را در طول حادثه حفظ کرد
- در 2 دقیقه بازیابی شد به جای 20+ دقیقه

### 12.4 سرویس Timeline توییتر

سرویس timeline توییتر از circuit breaker ها برای مدیریت طوفان توییت سلبریتی‌ها استفاده می‌کند:

```
سرویس ترکیب Timeline:
├─ CB: Tweet Fetch Service
│  └─ Fallback: توییت‌های Cache شده
├─ CB: Media Service  ← در طول افزایش رسانه ویروسی باز می‌شود
│  └─ Fallback: تصاویر placeholder
├─ CB: Analytics Service
│  └─ Fallback: شمارش‌های ثابت
└─ CB: Ad Service
   └─ Fallback: بدون تبلیغ (از دست دادن درآمد، اما timeline کار می‌کند)

زمانی که توییت سلبریتی‌ها ویروسی می‌شوند:
- سرویس رسانه تحت فشار قرار می‌گیرد
- Circuit breaker باز می‌شود
- Timeline ها تصاویر placeholder نشان می‌دهند
- تجربه اصلی خواندن حفظ می‌شود
- سرویس رسانه مقیاس می‌یابد
- Circuit زمانی که سالم است بسته می‌شود
```

## 13. نکات کلیدی / چارچوب مصاحبه

### 13.1 مفاهیم اساسی برای به خاطر سپردن

**تعریف در 30 ثانیه:**
"Circuit Breaker یک الگوی طراحی است که با نظارت بر شکست‌ها و مسدود کردن موقت درخواست‌ها به سرویس‌های خراب، از شکست‌های آبشاری در سیستم‌های توزیع‌شده جلوگیری می‌کند. مانند یک circuit breaker الکتریکی، زمانی که شکست‌ها از یک آستانه فراتر روند 'باز' می‌شود، به جای اتلاف منابع به سرعت شکست می‌خورد و به صورت دوره‌ای تست می‌کند آیا سرویس بازیابی شده است."

**سه حالت:**
1. **CLOSED:** عملیات عادی، درخواست‌ها عبور می‌کنند
2. **OPEN:** آستانه شکست فراتر رفته، fail fast
3. **HALF-OPEN:** تست بازیابی با درخواست‌های محدود

**پارامترهای کلیدی:**
- آستانه شکست (چه زمانی باز شود)
- مدت زمان timeout (چقدر باز بماند)
- آستانه موفقیت (چه زمانی از half-open بسته شود)

### 13.2 چارچوب بحث مصاحبه

**زمانی که سؤال می‌شود "یک سرویس مقاوم طراحی کنید":**

1. **با مشکل شروع کنید:**
   - "در سیستم‌های توزیع‌شده، یک سرویس خراب می‌تواند آبشاری شود و کل سیستم را از کار بیندازد"
   - "Circuit breaker ها با fail fast و دادن زمان برای بازیابی به سرویس‌ها از این جلوگیری می‌کنند"

2. **State machine را رسم کنید:**
   ```
   CLOSED ──failures──▶ OPEN ──timeout──▶ HALF-OPEN
     ▲                                       │
     └───────────success────────────────────┘
   ```

3. **در مورد پیکربندی بحث کنید:**
   - "آن را پیکربندی می‌کنم تا پس از 5 شکست متوالی یا 50٪ نرخ شکست در یک بازه 10 ثانیه‌ای باز شود"
   - "60 ثانیه باز بماند تا بازیابی را مجاز کند"
   - "با 3 درخواست در حالت half-open تست کنید"

4. **استراتژی‌های fallback را برنامه‌ریزی کنید:**
   - "برای عملیات خواندن، داده cache شده را برگردانید"
   - "برای نوشتن، برای پردازش بعدی به صف بزنید"
   - "برای عملیات حیاتی، از سرویس جایگزین استفاده کنید"

5. **نظارت را اضافه کنید:**
   - "حالت circuit، نرخ شکست و انتقال‌های حالت را ردیابی کنید"
   - "زمانی که circuit بیش از 5 دقیقه باز است هشدار دهید"
   - "داشبوردی که سلامت circuit را در تمام سرویس‌ها نشان می‌دهد"

### 13.3 سؤالات رایج مصاحبه

**س: "Circuit Breaker با الگوی Retry چه تفاوتی دارد؟"**

ج: "الگوی Retry با تلاش چندباره عملیات با backoff، شکست‌های موقت را مدیریت می‌کند. Circuit Breaker زمانی که یک سرویس به طور مداوم در حال شکست است از تلاش‌های مکرر جلوگیری می‌کند. آنها یکدیگر را تکمیل می‌کنند: از retry ها در داخل یک circuit breaker برای مدیریت مسائل موقت استفاده کنید، اما بگذارید circuit breaker زمانی که سرویس واقعاً از کار افتاده است retry ها را متوقف کند."

**س: "چگونه آستانه شکست را انتخاب می‌کنید؟"**

ج: "به نرخ خطای عادی سرویس و تأخیر بستگی دارد:
- برای سرویس‌های حیاتی با نرخ خطای عادی کم: 5 شکست متوالی یا 20٪ نرخ خطا
- برای سرویس‌ها با تنوع بیشتر: 10 شکست یا 50٪ نرخ خطا در یک بازه زمانی
- استفاده از sliding window (10 درخواست آخر یا 30 ثانیه آخر) به جای شکست‌های متوالی
- نظارت در تولید و تنظیم بر اساس نرخ false positive"

**س: "اگر circuit در طول یک عملیات نوشتن باز شود چه اتفاقی می‌افتد؟"**

ج: "چندین استراتژی:
1. نوشتن را برای پردازش بعدی به صف بزنید (consistency نهایی)
2. به storage محلی بنویسید و زمانی که سرویس بازیابی می‌شود همگام‌سازی کنید
3. خطا با header ی retry-after برگردانید
4. اگر موجود باشد از سرویس جایگزین استفاده کنید
انتخاب به نیازهای کسب‌وکار در مورد consistency داده و تجربه کاربر بستگی دارد."

**س: "چگونه از thundering herd زمانی که circuit بسته می‌شود جلوگیری می‌کنید؟"**

ج: "حالت HALF-OPEN این را حل می‌کند. به جای ارسال فوری همه ترافیک زمانی که circuit بسته می‌شود، ما:
1. فقط چند درخواست تست (مثلاً 3) را در حالت half-open اجازه می‌دهیم
2. اگر تست‌ها موفق باشند، به تدریج ترافیک را افزایش می‌دهیم
3. اگر تست‌ها شکست بخورند، فوراً دوباره باز می‌کنیم و بیشتر صبر می‌کنیم
4. همچنین می‌توانیم jitter به مدت زمان‌های timeout در نمونه‌ها اضافه کنیم"

**س: "Circuit breaker در سطح سرویس یا سطح نمونه؟"**

ج: "به مدل استقرار بستگی دارد:
- **سطح سرویس** (اشتراکی): همه نمونه‌ها حالت circuit را از طریق Redis/cache توزیع‌شده به اشتراک می‌گذارند
  - مزایا: رفتار یکنواخت، تشخیص سریع‌تر شکست
  - معایب: وابستگی اضافی، سربار هماهنگی
- **سطح نمونه** (مستقل): هر نمونه circuit خودش را دارد
  - مزایا: ساده، بدون وابستگی خارجی
  - معایب: ممکن است زمان بیشتری برای تشخیص شکست‌ها طول بکشد، تجربه کاربر ناهماهنگ

من برای بیشتر موارد از سطح نمونه استفاده می‌کنم، و فقط برای مسیرهای حیاتی که رفتار یکنواخت ضروری است از سطح سرویس استفاده می‌کنم."

### 13.4 چک لیست ملاحظات طراحی

هنگام پیاده‌سازی circuit breaker ها، در نظر بگیرید:

**پیکربندی:**
- [ ] آستانه شکست مناسب برای SLA سرویس
- [ ] مدت زمان timeout امکان بازیابی سرویس را می‌دهد
- [ ] درخواست‌های تست half-open کافی اما غرق‌کننده نیستند
- [ ] اندازه sliding window بین پاسخگویی و پایداری تعادل برقرار می‌کند

**تشخیص شکست:**
- [ ] چه چیزی به عنوان شکست محسوب می‌شود؟ (timeout ها، exception ها، 5xx)
- [ ] چه چیزی محسوب نمی‌شود؟ (خطاهای کلاینت 4xx)
- [ ] چگونه پاسخ‌های کند را مدیریت کنیم؟ (مبتنی بر timeout)

**استراتژی Fallback:**
- [ ] fallback معناداری برای هر circuit breaker وجود دارد
- [ ] Fallback تست و نگهداری شده است
- [ ] Fallback به همان سرویس خراب وابسته نیست
- [ ] استراتژی ابطال cache اگر از fallback های cache شده استفاده می‌شود

**نظارت:**
- [ ] حالت Circuit به عنوان معیار expose شده است
- [ ] نرخ شکست ردیابی و هشدار داده می‌شود
- [ ] انتقال‌های حالت log می‌شوند
- [ ] داشبوردی که سلامت circuit را نشان می‌دهد
- [ ] Distributed tracing شامل span های circuit breaker است

**تست:**
- [ ] تست‌های واحد برای انتقال‌های حالت
- [ ] تست‌های یکپارچه‌سازی برای سناریوهای شکست
- [ ] Chaos engineering برای تست رفتار تولید
- [ ] تست‌های بار با circuit breaker های فعال شده

**ملاحظات عملیاتی:**
- [ ] چگونه circuit ها را به صورت دستی باز/بسته کنیم؟
- [ ] چگونه آستانه‌ها را بدون استقرار مجدد تنظیم کنیم؟
- [ ] چگونه circuit breaker را بدون تأثیر بر کاربران تست کنیم؟
- [ ] Runbook برای حوادث circuit breaker

### 13.5 موضوعات پیشرفته برای مصاحبه‌های ارشد

**Circuit Breaker های تطبیقی:**
- تنظیم خودکار آستانه‌ها بر اساس نرخ شکست تاریخی
- یادگیری ماشین برای پیش‌بینی شکست‌ها قبل از آبشاری شدن
- Circuit breaking آگاه از زمینه (آستانه‌های متفاوت برای کاربران/مناطق مختلف)

**Circuit Breaker های چند سطحی:**
- Circuit breaker ها در لایه‌های مختلف (gateway، service، client)
- هماهنگی بین circuit breaker ها
- مدیریت سلسله‌مراتبی شکست

**Circuit Breaker های جهانی:**
- حالت اشتراکی در مراکز داده با استفاده از اجماع توزیع‌شده
- Consistency نهایی در حالت circuit breaker
- مدیریت پارتیشن‌های شبکه در circuit breaker های توزیع‌شده

**Circuit Breaker ها در سیستم‌های Event-Driven:**
- چگونه circuit breaker ها با پیام‌رسانی async کار می‌کنند
- Circuit breaking برای تولیدکنندگان رویداد در مقابل مصرف‌کنندگان
- Backpressure و circuit breaker ها

---

## خلاصه

الگوی Circuit Breaker برای ساخت سیستم‌های توزیع‌شده مقاوم ضروری است. با نظارت بر شکست‌ها و fail fast زمانی که سرویس‌ها ناسالم هستند، circuit breaker ها:

- از شکست‌های آبشاری جلوگیری می‌کنند
- منابع را حفظ می‌کنند (thread ها، اتصالات، حافظه)
- تجربه کاربر را با شکست‌های سریع و fallback ها بهبود می‌بخشند
- به سرویس‌های خراب زمان برای بازیابی می‌دهند
- سیگنال‌های واضحی برای نظارت و هشدار ارائه می‌دهند

در ترکیب با الگوهای retry، timeout ها و fallback ها، circuit breaker ها پایه [معماری microservices](../architecture/microservices.fa.md) مقاوم را تشکیل می‌دهند.

**به خاطر بسپارید:** Circuit breaker ها فقط در مورد مدیریت شکست‌ها نیستند—آنها در مورد کاهش تدریجی و حفظ در دسترس بودن سیستم حتی زمانی که اجزا شکست می‌خورند هستند.

---

**منابع:**
- Michael Nygard, "Release It! Design and Deploy Production-Ready Software" (2007)
- Martin Fowler, پست وبلاگ "CircuitBreaker" (2014)
- وبلاگ فنی Netflix: مستندات Hystrix و مطالعات موردی
- مستندات Resilience4j
- معماری Microsoft Azure: الگوی Circuit Breaker
