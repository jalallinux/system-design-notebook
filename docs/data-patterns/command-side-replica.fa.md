# الگوی Command-side Replica

## ۱. مقدمه

**Command-side Replica** یک الگوی مدیریت داده است که در سیستم‌های توزیع‌شده و [معماری‌های میکروسرویس](../architecture/microservices.md) استفاده می‌شود. این الگو شامل نگهداری یک **نسخه محلی و فقط-خواندنی** از داده‌هایی است که متعلق به سرویس دیگری هستند، به طور خاص برای پشتیبانی از **پردازش Command** -- یعنی اعتبارسنجی، مجوزدهی و اجرای منطق کسب‌وکار در سمت نوشتن.

### ریشه و انگیزه

در [معماری میکروسرویس](../architecture/microservices.md) که از اصل Database per Service پیروی می‌کند، هر سرویس به طور انحصاری مالک داده‌های خود است. با این حال، پردازش یک Command اغلب به داده‌هایی نیاز دارد که در سرویس دیگری وجود دارند. الگوی Command-side Replica این مشکل را با نگهداری یک **projection محلی** از داده‌های راه دور، که به صورت ناهمزمان از طریق Domain Event‌ها به‌روزرسانی می‌شود، حل می‌کند.

### تفاوت با Read Model در CQRS

یک منبع رایج سردرگمی، رابطه بین Command-side Replica و Read Model در [CQRS](./cqrs.md) است. در حالی که هر دو projection‌های مشتق‌شده از داده هستند، اهداف اساساً متفاوتی را دنبال می‌کنند:

| جنبه | Read Model در CQRS | Command-side Replica |
|------|---------------------|----------------------|
| **هدف** | پاسخ به Query‌ها (سمت خواندن) | پشتیبانی از اعتبارسنجی Command (سمت نوشتن) |
| **مصرف‌کننده** | Query Handler‌ها، پاسخ‌های API | Command Handler‌ها، منطق دامنه |
| **منبع داده** | Write Model همان سرویس | Domain Event‌های سرویس دیگر |
| **نیاز به سازگاری** | نهایی (برای نمایش) | نهایی (برای اعتبارسنجی) |
| **مکان** | سمت خواندن همان سرویس | سمت نوشتن یک سرویس *دیگر* |

**نکته کلیدی**: Command-side Replica در **مسیر نوشتن** قرار دارد، نه مسیر خواندن. این الگو به یک سرویس امکان می‌دهد بدون انجام فراخوانی‌های همزمان به سرویس‌های خارجی، Command‌ها را اعتبارسنجی و پردازش کند.

---

## ۲. زمینه و مسئله

### زمینه

سیستمی را در نظر بگیرید که با [میکروسرویس‌ها](../architecture/microservices.md) ساخته شده، جایی که هر سرویس پایگاه داده خود را دارد (الگوی Database per Service). سرویس‌ها از طریق رویدادها به صورت ناهمزمان یا از طریق API‌ها به صورت همزمان ارتباط برقرار می‌کنند.

```
┌──────────────────┐          ┌──────────────────┐
│  Order Service   │          │ Product Service   │
│                  │          │                   │
│  Orders DB       │          │  Products DB      │
│  ┌────────────┐  │          │  ┌─────────────┐  │
│  │ orders     │  │          │  │ products    │  │
│  │ order_items│  │          │  │ inventory   │  │
│  └────────────┘  │          │  │ pricing     │  │
│                  │          │  └─────────────┘  │
└──────────────────┘          └──────────────────┘
```

### مسئله

وقتی Order Service یک Command به نام `PlaceOrder` دریافت می‌کند، باید:

1. **اعتبارسنجی** کند که محصولات درخواستی وجود دارند
2. **بررسی** کند که محصولات در انبار موجود هستند
3. **تأیید** کند قیمت‌های فعلی را برای محاسبه مجموع سفارش
4. **اعمال** کند قوانین کسب‌وکار (مثلاً حداکثر تعداد سفارش، واجد شرایط بودن مشتری)

تمام این داده‌ها در **Product Service** قرار دارند. Order Service چگونه باید به آن‌ها دسترسی پیدا کند؟

### رویکرد ساده: فراخوانی‌های همزمان API

```
┌──────────┐     PlaceOrder      ┌──────────────┐
│  Client   │ ──────────────────▶│ Order Service │
└──────────┘                     └──────┬───────┘
                                        │
                          GET /products/123
                          GET /products/456
                          GET /inventory/123
                          GET /inventory/456
                                        │
                                        ▼
                                 ┌──────────────┐
                                 │Product Service│
                                 └──────────────┘
```

**مشکلات این رویکرد:**

- **تأخیر**: فراخوانی‌های همزمان HTTP متعدد جمع می‌شوند، به خصوص تحت بار
- **وابستگی**: Order Service به در دسترس بودن Product Service در زمان پردازش Command وابسته است
- **شکنندگی**: اگر Product Service از کار بیفتد، Order Service نمی‌تواند هیچ Command‌ای را پردازش کند
- **توان عملیاتی**: رفت‌وبرگشت‌های شبکه‌ای برای پردازش Command‌های با حجم بالا تبدیل به گلوگاه می‌شوند
- **شکست‌های آبشاری**: پاسخ‌های کند از Product Service می‌توانند باعث تایم‌اوت و شکست‌های آبشاری شوند (ر.ک. [Circuit Breaker](../resilience/circuit-breaker.md))

---

## ۳. نیروها

نیروهای زیر به سمت پذیرش الگوی Command-side Replica سوق می‌دهند:

| نیرو | توضیح |
|------|-------|
| **پردازش Command با تأخیر کم** | Command‌ها باید سریع اعتبارسنجی شوند؛ فراخوانی‌های شبکه‌ای به سرویس‌های دیگر تأخیر غیرقابل قبولی اضافه می‌کنند |
| **استقلال و تاب‌آوری** | یک سرویس باید بتواند Command‌ها را حتی وقتی سرویس‌های دیگر موقتاً در دسترس نیستند پردازش کند |
| **کاهش وابستگی** | فراخوانی‌های همزمان بین سرویسی وابستگی‌های زمان اجرا ایجاد می‌کنند که تاب‌آوری سیستم را کاهش می‌دهند |
| **سازگاری نهایی قابل قبول** | داده‌های تکرارشده نیازی به به‌روز بودن کامل ندارند؛ کمی تأخیر برای اهداف اعتبارسنجی قابل تحمل است |
| **توان عملیاتی بالای Command** | سرویس حجم زیادی از Command‌ها را مدیریت می‌کند و نمی‌تواند هزینه رفت‌وبرگشت شبکه برای هر کدام را بپذیرد |
| **مرزهای مالکیت داده** | داده متعلق به سرویس دیگری است و نباید مستقیماً از پایگاه داده آن دسترسی پیدا شود |

### تنش‌ها

- **تازگی در مقابل عملکرد**: داده محلی سریع‌تر است اما ممکن است قدیمی باشد
- **سادگی در مقابل تاب‌آوری**: فراخوانی‌های همزمان ساده‌تر هستند اما وابستگی‌های زمان اجرا ایجاد می‌کنند
- **تکرار در مقابل استقلال**: تکرار داده یعنی نگهداری آن در دو مکان، اما عملکرد مستقل را ممکن می‌سازد

---

## ۴. راه‌حل

### ایده اصلی

به **Domain Event‌های** منتشرشده توسط سرویسی که مالک داده است مشترک شوید. یک **نسخه محلی فقط-خواندنی** از زیرمجموعه مرتبط آن داده را نگهداری کنید. از این نسخه در حین پردازش Command برای اعتبارسنجی و منطق کسب‌وکار استفاده کنید.

### معماری

```
┌─────────────────────────────────────────────────────────────────┐
│                     Product Service (Data Owner)                 │
│                                                                  │
│  ┌─────────────┐     Publishes Events     ┌──────────────────┐  │
│  │ Products DB │ ────────────────────────▶ │  Event Bus /     │  │
│  └─────────────┘                           │  Message Broker  │  │
│                                            └────────┬─────────┘  │
└─────────────────────────────────────────────────────┼────────────┘
                                                      │
                    ProductCreated                     │
                    ProductPriceChanged                │
                    InventoryUpdated                   │
                    ProductDiscontinued                │
                                                      │
┌─────────────────────────────────────────────────────┼────────────┐
│                     Order Service (Consumer)         │            │
│                                                      ▼            │
│  ┌──────────────────┐    ┌──────────────────────────────────┐    │
│  │  Command Handler │    │  Event Handler                    │    │
│  │  (PlaceOrder)    │    │  (Subscribes to Product events)  │    │
│  │                  │    │                                    │    │
│  │  Uses replica ──▶│    │  Updates replica                  │    │
│  │  for validation  │    │  ┌────────────────────────────┐   │    │
│  └──────────────────┘    │  │ product_replica table      │   │    │
│                          │  │ ─────────────────────────  │   │    │
│                          │  │ product_id  (PK)           │   │    │
│                          │  │ name                       │   │    │
│                          │  │ price                      │   │    │
│                          │  │ stock_quantity              │   │    │
│                          │  │ is_available                │   │    │
│                          │  │ last_updated_at             │   │    │
│                          │  └────────────────────────────┘   │    │
│                          └──────────────────────────────────┘    │
│                                                                  │
│  ┌──────────────┐                                                │
│  │  Orders DB   │   (Orders + Product Replica in same DB)        │
│  └──────────────┘                                                │
└──────────────────────────────────────────────────────────────────┘
```

### جریان گام‌به‌گام

```
Step 1: Product Service publishes domain events
─────────────────────────────────────────────────

Product Service ──▶ ProductCreated {
                        productId: "P-001",
                        name: "Wireless Keyboard",
                        price: 49.99,
                        stockQuantity: 500
                    }

Step 2: Order Service subscribes and updates local replica
──────────────────────────────────────────────────────────

Event Handler receives ProductCreated
  ──▶ INSERT INTO product_replica
       (product_id, name, price, stock_quantity, is_available, last_updated_at)
       VALUES ('P-001', 'Wireless Keyboard', 49.99, 500, true, NOW())

Step 3: Client sends PlaceOrder command
───────────────────────────────────────

Client ──▶ PlaceOrder {
               customerId: "C-100",
               items: [
                   { productId: "P-001", quantity: 2 },
                   { productId: "P-042", quantity: 1 }
               ]
           }

Step 4: Command handler validates using local replica
─────────────────────────────────────────────────────

PlaceOrderHandler:
  1. Load products from product_replica        ← LOCAL query, no network call
     SELECT * FROM product_replica
     WHERE product_id IN ('P-001', 'P-042')

  2. Validate all products exist               ← LOCAL check
  3. Validate all products are available        ← LOCAL check
  4. Validate sufficient stock                  ← LOCAL check
  5. Calculate total using replica prices       ← LOCAL calculation
  6. Create order                               ← Write to orders table
  7. Publish OrderPlaced event                  ← For downstream consumers

Step 5: Order created successfully
──────────────────────────────────

No synchronous calls to Product Service were needed.
```

### چه داده‌هایی را تکرار کنیم

شما باید فقط **حداقل زیرمجموعه** داده‌های مورد نیاز برای اعتبارسنجی Command را تکرار کنید:

```
Product Service owns:                Order Service replicates:
───────────────────                  ─────────────────────────
product_id              ──────────▶  product_id
name                    ──────────▶  name
description             ✗ (not needed for validation)
price                   ──────────▶  price
stock_quantity          ──────────▶  stock_quantity
is_available            ──────────▶  is_available
images[]                ✗ (not needed for validation)
detailed_specs          ✗ (not needed for validation)
reviews[]               ✗ (not needed for validation)
created_at              ✗ (not needed for validation)
last_updated_at         ──────────▶  last_updated_at
```

**اصل**: حداقل داده مورد نیاز را تکرار کنید. داده کمتر یعنی رویدادهای کمتر برای پردازش، فضای ذخیره‌سازی کمتر و پنجره سازگاری کوچکتر.

### مدیریت رویداد: به‌روز نگه داشتن Replica

```
┌─────────────────────────────────────────────────────┐
│               Event Handlers in Order Service        │
├─────────────────────────────────────────────────────┤
│                                                      │
│  on ProductCreated(event):                           │
│      INSERT INTO product_replica                     │
│      VALUES (event.productId, event.name,            │
│              event.price, event.stock, true, NOW())  │
│                                                      │
│  on ProductPriceChanged(event):                      │
│      UPDATE product_replica                          │
│      SET price = event.newPrice,                     │
│          last_updated_at = NOW()                     │
│      WHERE product_id = event.productId              │
│                                                      │
│  on InventoryUpdated(event):                         │
│      UPDATE product_replica                          │
│      SET stock_quantity = event.newQuantity,          │
│          last_updated_at = NOW()                     │
│      WHERE product_id = event.productId              │
│                                                      │
│  on ProductDiscontinued(event):                      │
│      UPDATE product_replica                          │
│      SET is_available = false,                       │
│          last_updated_at = NOW()                     │
│      WHERE product_id = event.productId              │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### مدیریت رویداد Idempotent

از آنجایی که رویدادها ممکن است بیش از یک بار تحویل داده شوند (تحویل حداقل یک‌بار)، Handler‌ها باید **Idempotent** باشند:

```
on ProductPriceChanged(event):
    // Check if we have already processed this event
    if (event.eventId already in processed_events table):
        return   // Skip duplicate

    UPDATE product_replica
    SET price = event.newPrice,
        last_updated_at = NOW()
    WHERE product_id = event.productId

    INSERT INTO processed_events (event_id, processed_at)
    VALUES (event.eventId, NOW())
```

برای اطلاعات بیشتر درباره تضمین‌های تحویل رویداد، به [معماری رویداد-محور](../event-driven/event-driven-architecture.md) مراجعه کنید.

---

## ۵. مثال

### سناریو: پردازش سفارش تجارت الکترونیک

یک پلتفرم تجارت الکترونیک سرویس‌های زیر را دارد:

```
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│ Product Service│  │Customer Service│  │  Order Service  │
│                │  │                │  │                 │
│ - Products     │  │ - Customers    │  │ - Orders        │
│ - Inventory    │  │ - Memberships  │  │ - Order Items   │
│ - Pricing      │  │ - Credit Limits│  │                 │
│                │  │                │  │ Replicas:       │
│                │  │                │  │ - Products (R/O)│
│                │  │                │  │ - Customers(R/O)│
└───────┬────────┘  └───────┬────────┘  └────────────────┘
        │                   │
        │   Domain Events   │
        └───────────────────┘
                │
                ▼
        ┌───────────────┐
        │  Message Broker│
        │  (e.g., Kafka) │
        └───────┬───────┘
                │
                ▼
        Order Service subscribes
        and maintains replicas
```

### Command: PlaceOrder

```
PlaceOrderCommand {
    customerId: "C-100"
    items: [
        { productId: "P-001", quantity: 2 },
        { productId: "P-042", quantity: 1 }
    ]
    shippingAddress: "123 Main St"
}
```

### اعتبارسنجی با استفاده از Command-side Replica‌ها

```
PlaceOrderHandler.handle(command):

    // Step 1: Validate customer using CUSTOMER replica
    customer = customerReplica.findById(command.customerId)
    if (customer == null)
        throw CustomerNotFoundException

    if (customer.status != ACTIVE)
        throw CustomerNotActiveException

    // Step 2: Validate products using PRODUCT replica
    products = productReplica.findByIds(command.items.map(i => i.productId))

    if (products.size != command.items.size)
        throw ProductNotFoundException

    for each item in command.items:
        product = products.get(item.productId)

        if (!product.isAvailable)
            throw ProductNotAvailableException

        if (product.stockQuantity < item.quantity)
            throw InsufficientStockException

    // Step 3: Calculate total using replica prices
    total = 0
    for each item in command.items:
        product = products.get(item.productId)
        total += product.price * item.quantity

    // Step 4: Check customer credit limit
    if (customer.membershipType == CREDIT)
        if (total > customer.creditLimit)
            throw CreditLimitExceededException

    // Step 5: Create order (write to Order Service's own DB)
    order = Order.create(command.customerId, command.items, total)
    orderRepository.save(order)

    // Step 6: Publish domain event
    eventBus.publish(OrderPlaced {
        orderId: order.id,
        customerId: command.customerId,
        items: command.items,
        total: total
    })

    return order.id
```

### نمودار کامل جریان داده

```
     Product Service              Message Broker              Order Service
     ────────────────             ──────────────              ─────────────

     Admin updates price
     for product P-001
            │
            ▼
     ProductPriceChanged ────────▶  Topic:          ────────▶  Event Handler
     {                              product.events              │
       productId: "P-001",                                      ▼
       oldPrice: 49.99,                                  UPDATE product_replica
       newPrice: 44.99                                   SET price = 44.99
     }                                                   WHERE product_id = 'P-001'
                                                                │
                                                                ▼
                                                         Replica is now current
                                                                │
     ───────── Time passes ─────────                            │
                                                                │
     Client sends PlaceOrder ───────────────────────────▶ PlaceOrderHandler
     for product P-001                                          │
                                                                ▼
                                                         SELECT * FROM
                                                         product_replica
                                                         WHERE product_id = 'P-001'
                                                                │
                                                                ▼
                                                         price = 44.99 (current!)
                                                         Validates and creates order
```

---

## ۶. مزایا و معایب

### مزایا

| مزیت | توضیح |
|------|-------|
| **پردازش Command با تأخیر کم** | تمام داده‌های اعتبارسنجی محلی هستند -- بدون رفت‌وبرگشت شبکه در حین پردازش Command |
| **استقلال سرویس** | سرویس می‌تواند Command‌ها را حتی زمانی که سرویس مبدأ موقتاً از کار افتاده پردازش کند |
| **کاهش وابستگی زمان اجرا** | هیچ وابستگی همزمانی به سرویس‌های دیگر در زمان Command وجود ندارد |
| **تاب‌آوری** | عدم دسترسی موقت به سرویس مالک داده، پردازش Command را مسدود نمی‌کند |
| **عملکرد قابل پیش‌بینی** | کوئری‌های پایگاه داده محلی تأخیر ثابت و قابل پیش‌بینی دارند |
| **مقیاس‌پذیری** | توان عملیاتی Command توسط ظرفیت سرویس خارجی محدود نمی‌شود |
| **تناسب طبیعی با سیستم‌های رویداد-محور** | از زیرساخت [معماری رویداد-محور](../event-driven/event-driven-architecture.md) موجود بهره می‌برد |

### معایب

| عیب | توضیح | راه‌حل |
|-----|-------|--------|
| **سازگاری نهایی** | داده Replica ممکن است قدیمی باشد؛ قیمت محصول ممکن است بین به‌روزرسانی Replica و پردازش Command تغییر کند | به عنوان مصالحه بپذیرید؛ از تراکنش‌های جبرانی ([الگوی Saga](../distributed-transactions/saga.md)) برای موارد بحرانی استفاده کنید |
| **تکرار داده** | همان داده در چندین سرویس ذخیره می‌شود که هزینه ذخیره‌سازی و نگهداری را افزایش می‌دهد | فقط حداقل فیلدهای ضروری را تکرار کنید |
| **پیچیدگی مدیریت رویداد** | باید ترتیب رویدادها، Idempotency و شکست‌ها را مدیریت کنید | از Handler‌های Idempotent، حذف تکرار رویداد و صف‌های Dead Letter استفاده کنید |
| **تکامل Schema** | تغییرات در Schema رویدادهای سرویس مبدأ نیاز به به‌روزرسانی در تمام سرویس‌های مصرف‌کننده دارد | از نسخه‌بندی رویداد و Schema‌های backward-compatible استفاده کنید |
| **بارگذاری اولیه داده** | وقتی یک سرویس از صفر شروع می‌کند، باید Replica را از ابتدا پر کند | از رویدادهای Snapshot یا API‌های دسته‌ای برای بارگذاری اولیه استفاده کنید |
| **خطر داده قدیمی** | اگر رویدادها با تأخیر یا از دست رفته باشند، Replica ممکن است داده‌های منقضی داشته باشد | تازگی Replica را مانیتور کنید؛ بررسی‌های قدیمی بودن را در اعتبارسنجی‌های حیاتی اضافه کنید |
| **پیچیدگی اشکال‌زدایی** | ردیابی جریان داده در سرویس‌ها و رویدادها سخت‌تر از یک فراخوانی ساده API است | از Correlation ID‌ها، ردیابی توزیع‌شده و لاگ‌های رویداد استفاده کنید |

### وقتی قدیمی بودن داده اهمیت دارد

همه قدیمی بودن‌ها یکسان نیستند. این سناریوها را در نظر بگیرید:

```
Scenario 1: Price changed 2 seconds ago, replica not yet updated
─────────────────────────────────────────────────────────────────
Risk Level: LOW
Impact: Customer charged old price ($49.99 instead of $44.99)
Mitigation: Saga compensates or business absorbs small difference

Scenario 2: Product discontinued, replica not yet updated
─────────────────────────────────────────────────────────────────
Risk Level: MEDIUM
Impact: Order placed for unavailable product
Mitigation: Downstream inventory check fails, Saga triggers compensation

Scenario 3: Stock depleted, replica shows stock available
─────────────────────────────────────────────────────────────────
Risk Level: MEDIUM
Impact: Overselling
Mitigation: Final stock reservation at inventory service;
            Saga handles out-of-stock compensation
```

**اصل کلیدی**: Command-side Replica یک **اعتبارسنجی بهترین-تلاش** ارائه می‌دهد. برای محدودیت‌های حیاتی (مانند رزرو واقعی موجودی)، یک مرحله پایین‌دستی در یک [Saga](../distributed-transactions/saga.md) باید بررسی معتبر را در برابر منبع حقیقت انجام دهد.

---

## ۷. الگوهای مرتبط

### مستقیماً مرتبط

| الگو | رابطه |
|------|-------|
| [Database per Service](./database-per-service.md) | محدودیتی که انگیزه Command-side Replica‌ها را ایجاد می‌کند -- هر سرویس مالک داده‌های خود است |
| [CQRS](./cqrs.md) | Command-side Replica در سمت نوشتن است؛ Read Model‌های CQRS در سمت خواندن. هر دو projection هستند اما برای اهداف متفاوت |
| [Domain Event](../event-driven/domain-event.md) | مکانیزمی که برای همگام‌سازی Replica با منبع حقیقت استفاده می‌شود |
| [الگوی Saga](../distributed-transactions/saga.md) | برای مدیریت مواردی استفاده می‌شود که سازگاری نهایی Replica منجر به Command‌های نامعتبر می‌شود که نیاز به جبران دارند |

### الگوهای مکمل

| الگو | نحوه تکمیل |
|------|-----------|
| [معماری رویداد-محور](../event-driven/event-driven-architecture.md) | زیرساخت (Event Bus، موضوعات، اشتراک‌ها) را برای انتشار رویداد فراهم می‌کند |
| [Circuit Breaker](../resilience/circuit-breaker.md) | در برابر شکست‌های آبشاری محافظت می‌کند اگر بازگشت به فراخوانی‌های همزمان نیاز باشد |
| [API Gateway](../architecture/api-gateway.md) | Command‌ها را به سرویس مناسب جایی که Command-side Replica‌ها استفاده می‌شوند هدایت می‌کند |
| [قضیه CAP](../fundamentals/cap-theorem.md) | توضیح می‌دهد چرا سازگاری نهایی یک مصالحه قابل قبول در سیستم‌های توزیع‌شده است |

### مقایسه الگوها

```
┌───────────────────────────────────────────────────────────────┐
│          How services get data they don't own                  │
├────────────────────┬──────────────────────────────────────────┤
│                    │                                           │
│  1. Sync API Call  │  Service A ──HTTP──▶ Service B           │
│     (Request/      │  + Simple                                │
│      Response)     │  - Coupled, fragile, slow                │
│                    │                                           │
├────────────────────┼──────────────────────────────────────────┤
│                    │                                           │
│  2. Command-side   │  Service B ──Events──▶ Service A         │
│     Replica        │                        (local replica)   │
│     (This pattern) │  + Fast, decoupled, resilient            │
│                    │  - Eventually consistent, data duplication│
│                    │                                           │
├────────────────────┼──────────────────────────────────────────┤
│                    │                                           │
│  3. API Composition│  Gateway ──▶ Service A + Service B       │
│     (at gateway)   │              (aggregate responses)       │
│                    │  + No duplication                         │
│                    │  - Gateway complexity, still coupled      │
│                    │                                           │
├────────────────────┼──────────────────────────────────────────┤
│                    │                                           │
│  4. Shared Database│  Service A and B share same DB           │
│     (anti-pattern) │  + Simple                                │
│                    │  - Defeats microservices purpose          │
│                    │                                           │
└────────────────────┴──────────────────────────────────────────┘
```

---

## ۸. استفاده در دنیای واقعی

### چه زمانی از Command-side Replica استفاده کنیم

| سناریو | مثال |
|---------|------|
| **اعتبارسنجی Command نیازمند داده خارجی** | Order Service اعتبارسنجی قیمت و موجودی محصول از Product Service |
| **پردازش Command با توان عملیاتی بالا** | Payment Service بررسی موجودی حساب از Account Service |
| **Command‌های حیاتی از نظر تاب‌آوری** | Booking Service اعتبارسنجی در دسترس بودن اتاق از Hotel Service، حتی اگر Hotel Service موقتاً از کار افتاده باشد |
| **قوانین کسب‌وکار بین سرویسی** | Lending Service بررسی امتیاز اعتباری مشتری از Credit Service برای تأیید وام |
| **تجمیع چند سرویسی برای Command‌ها** | Shipping Service نیاز به آدرس مشتری (از Customer Service) و ابعاد بسته (از Product Service) برای محاسبه نرخ |

### چه زمانی استفاده نکنیم

| سناریو | چرا | جایگزین |
|---------|-----|---------|
| **داده خیلی مکرر تغییر می‌کند** | Replica دائماً قدیمی است؛ رویدادها سیستم را غرق می‌کنند | از API همزمان با کش و [Circuit Breaker](../resilience/circuit-breaker.md) استفاده کنید |
| **سازگاری کامل مورد نیاز است** | هیچ قدیمی بودنی در اعتبارسنجی قابل تحمل نیست | از [Two-Phase Commit](../distributed-transactions/two-phase-commit.md) یا فراخوانی همزمان استفاده کنید |
| **مجموعه داده‌های خیلی بزرگ** | تکرار میلیون‌ها رکورد غیرعملی است | از تکرار انتخابی (فقط داده داغ) یا فراخوانی API استفاده کنید |
| **سیستم‌های ساده با ترافیک کم** | سربار رویدادها و Replica‌ها توجیه ندارد | فراخوانی‌های مستقیم همزمان API کافی هستند |

### مثال‌های صنعتی

**۱. Amazon -- پردازش سفارش**

سرویس سفارش Amazon نسخه‌های محلی از داده‌های کاتالوگ محصول (قیمت‌گذاری، موجودی) را نگهداری می‌کند تا سفارشات را بدون فراخوانی همزمان به سرویس کاتالوگ اعتبارسنجی کند. تأیید نهایی قیمت به عنوان بخشی از یک [Saga](../distributed-transactions/saga.md) قبل از شارژ مشتری انجام می‌شود.

**۲. Uber -- قیمت‌گذاری سفر**

سرویس سفر یک نسخه محلی از داده‌های قیمت‌گذاری Surge و در دسترس بودن راننده نگهداری می‌کند. وقتی یک مسافر درخواست سفر می‌دهد، نسخه محلی یک تخمین قیمت فوری ارائه می‌دهد. کرایه واقعی بعداً با استفاده از سرویس قیمت‌گذاری معتبر تأیید می‌شود.

**۳. Netflix -- مجوزدهی محتوا**

وقتی کاربر دکمه پخش را فشار می‌دهد، سرویس پخش مجوز محتوا و وضعیت اشتراک کاربر را با استفاده از نسخه‌های محلی اعتبارسنجی می‌کند نه با فراخوانی همزمان به سرویس اشتراک و سرویس مجوز. این امکان تصمیمات مجوزدهی زیر ۱۰۰ میلی‌ثانیه را فراهم می‌کند.

**۴. بانکداری -- اعتبارسنجی انتقال**

سرویس انتقال یک نسخه از وضعیت‌های حساب و اطلاعات اولیه موجودی را نگهداری می‌کند. وقتی یک Command انتقال می‌رسد، اعتبارسنجی می‌کند که حساب فعال است و موجودی تقریبی کافی دارد. کسر واقعی موجودی توسط سرویس حساب در یک مرحله Saga انجام می‌شود.

### فناوری‌های پیاده‌سازی

| مؤلفه | گزینه‌های فناوری |
|-------|----------------|
| **Event Bus** | Apache Kafka، RabbitMQ، AWS SNS/SQS، Azure Service Bus |
| **ذخیره‌سازی Replica** | همان پایگاه داده سرویس (جدول جداگانه)، Redis، کش در حافظه |
| **فرمت رویداد** | CloudEvents، Avro، Protobuf، JSON |
| **بارگذاری اولیه** | رویدادهای Snapshot، REST API دسته‌ای، dump پایگاه داده |
| **مانیتورینگ** | معیارهای تأخیر Replica، بررسی‌های تازگی، داشبوردهای پردازش رویداد |

برای مقایسه فناوری‌های Event Bus، به [مقایسه RabbitMQ و Kafka](../messaging/rabbitmq-vs-kafka.md) مراجعه کنید.

---

## ۹. خلاصه

### نکات کلیدی

1. **Command-side Replica یک نسخه محلی فقط-خواندنی** از داده‌های متعلق به سرویس دیگری است که از طریق Domain Event‌ها نگهداری می‌شود و در **سمت نوشتن** برای اعتبارسنجی Command و منطق کسب‌وکار استفاده می‌شود.

2. **مسئله‌ای را حل می‌کند** که نیاز به داده خارجی در حین پردازش Command بدون انجام فراخوانی‌های همزمان بین سرویسی که تأخیر، وابستگی و شکنندگی را افزایش می‌دهند.

3. **یک Read Model در CQRS نیست**. در حالی که هر دو projection هستند، Command-side Replica در مسیر نوشتن وجود دارد، نه مسیر خواندن. [Read Model در CQRS](./cqrs.md) Query‌ها را سرو می‌کند؛ Command-side Replica اعتبارسنجی Command را سرو می‌کند.

4. **سازگاری نهایی مصالحه اصلی است**. Replica ممکن است کمی قدیمی باشد که برای اکثر سناریوهای اعتبارسنجی قابل قبول است. برای بررسی‌های حیاتی، از یک مرحله پایین‌دستی [Saga](../distributed-transactions/saga.md) در برابر منبع معتبر استفاده کنید.

5. **فقط آنچه نیاز دارید را تکرار کنید**. Replica را حداقل نگه دارید -- فقط فیلدهایی که برای اعتبارسنجی Command لازم هستند. این حجم رویدادها، ذخیره‌سازی و پنجره سازگاری را کاهش می‌دهد.

6. **Event Handler‌های Idempotent ضروری هستند**. رویدادها ممکن است بیش از یک بار تحویل داده شوند؛ Handler‌ها باید تکراری‌ها را با ظرافت مدیریت کنند.

7. **تازگی Replica را مانیتور کنید**. تأخیر بین منبع حقیقت و Replica محلی را ردیابی کنید. در صورت قدیمی بودن بیش از حد هشدار دهید.

### چک‌لیست تصمیم‌گیری

```
Should you use a Command-side Replica?

 [ ] Your service needs data from another service to validate commands
 [ ] The data can be eventually consistent for validation purposes
 [ ] You need low-latency command processing
 [ ] You want your service to be resilient to other services being down
 [ ] You have an event-driven infrastructure in place (or plan to build one)
 [ ] The dataset to replicate is manageable in size
 [ ] Your team is comfortable with eventual consistency and event handling

If most boxes are checked ──▶ Command-side Replica is a good fit.
If few boxes are checked  ──▶ Consider synchronous API calls or other patterns.
```

### مرجع سریع

```
┌─────────────────────────────────────────────────────────────────┐
│                    Command-side Replica Pattern                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  WHAT:  Local read-only copy of another service's data           │
│  WHY:   Low-latency command validation without sync calls        │
│  HOW:   Subscribe to domain events, maintain local projection    │
│  WHERE: On the write side of the consuming service               │
│                                                                  │
│  ┌──────────┐   Events    ┌──────────┐   Local     ┌─────────┐ │
│  │ Source   │ ──────────▶ │ Replica  │ ──────────▶ │ Command │ │
│  │ Service  │             │ (R/O)    │   Query     │ Handler │ │
│  └──────────┘             └──────────┘             └─────────┘ │
│                                                                  │
│  Trade-off: Eventual consistency for performance & resilience    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### مطالعه بیشتر

- [CQRS](./cqrs.md) -- جداسازی مدل‌های خواندن و نوشتن در یک سرویس
- [الگوی Saga](../distributed-transactions/saga.md) -- هماهنگی تراکنش‌های توزیع‌شده با جبران
- [معماری رویداد-محور](../event-driven/event-driven-architecture.md) -- الگوهای رویداد و تضمین‌های تحویل
- [قضیه CAP](../fundamentals/cap-theorem.md) -- درک مصالحه‌های سازگاری در سیستم‌های توزیع‌شده
- [معماری میکروسرویس‌ها](../architecture/microservices.md) -- سبک معماری که انگیزه این الگو را ایجاد می‌کند
- [مقایسه RabbitMQ و Kafka](../messaging/rabbitmq-vs-kafka.md) -- انتخاب فناوری پیام‌رسانی برای انتشار رویداد

---

**به خاطر داشته باشید**: الگوی Command-side Replica تازگی داده را با عملکرد و تاب‌آوری مبادله می‌کند. این یک انتخاب عمل‌گرایانه در سیستم‌هایی است که فراخوانی‌های همزمان بین سرویسی در زمان Command هزینه زیادی دارند و سازگاری نهایی یک مصالحه قابل قبول برای اعتبارسنجی Command است.
