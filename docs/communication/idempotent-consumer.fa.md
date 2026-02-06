# مصرف‌کننده توان‌وار (Idempotent Consumer)

## ۱. مقدمه

الگوی مصرف‌کننده توان‌وار (Idempotent Consumer) تضمین می‌کند که یک پردازشگر پیام می‌تواند همان پیام را چندین بار به صورت ایمن پردازش کند و همیشه همان نتیجه را تولید کند، صرف‌نظر از تعداد دفعات تحویل پیام. در سیستم‌های توزیع‌شده مبتنی بر [پیام‌رسانی](./messaging.md)، دستیابی به پردازش دقیقاً-یک‌بار بسیار دشوار است. واسطه‌های پیام معمولاً **تحویل حداقل-یک‌بار** را تضمین می‌کنند، به این معنی که مصرف‌کنندگان باید آماده مدیریت پیام‌های تکراری به صورت شایسته باشند.

**ریشه و زمینه:**

اصطلاح «توان‌وار» (idempotent) از ریاضیات می‌آید، جایی که یک عملیات توان‌وار است اگر اعمال آن چندین بار همان نتیجه‌ای را داشته باشد که اعمال آن یک بار. در زمینه پیام‌رسانی، یک مصرف‌کننده توان‌وار کسی است که می‌تواند همان پیام را دو، سه یا صد بار دریافت و پردازش کند بدون اینکه اثرات جانبی نادرستی مانند شارژ مضاعف، سفارش‌های تکراری یا اعلان‌های مکرر ایجاد کند.

**چرا اهمیت دارد:**

در [معماری میکروسرویس‌ها](../architecture/microservices.md)، سرویس‌ها از طریق واسطه‌های پیام مانند [RabbitMQ یا Kafka](../messaging/rabbitmq-vs-kafka.md) به صورت ناهمزمان ارتباط برقرار می‌کنند. خرابی‌های شبکه، سقوط مصرف‌کننده، بازتوازن واسطه، و مکانیزم‌های تلاش مجدد همگی به احتمال تحویل تکراری پیام کمک می‌کنند. بدون مصرف‌کنندگان توان‌وار، این تکرارها می‌توانند منجر به فساد داده و خطاهای منطق کسب‌وکار شوند که شناسایی و حل آن‌ها دشوار است.

**اصل اساسی:**

پردازشگرهای پیام را طوری طراحی کنید که پردازش همان پیام بیش از یک بار دقیقاً همان اثری را داشته باشد که پردازش آن یک بار دارد. این کار از طریق ردیابی حذف تکرار، عملیات‌های ذاتاً توان‌وار، یا کلیدهای توان‌واری ارائه شده توسط کلاینت قابل دستیابی است.

---

## ۲. زمینه و مسئله

### سناریوی تحویل تکراری

یک پلتفرم تجارت الکترونیک را در نظر بگیرید که سرویس پرداخت سفارشات را از طریق یک خط لوله رویداد-محور پردازش می‌کند:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Order      │────▶│   Message    │────▶│   Payment    │
│   Service    │     │   Broker     │     │   Service    │
└──────────────┘     └──────────────┘     └──────────────┘
                           │
                    Message: PaymentRequested
                    {orderId: 1234, amount: $99.99}
```

**چه مشکلاتی می‌تواند رخ دهد:**

1. سرویس پرداخت پیام `PaymentRequested` را دریافت و پردازش را آغاز می‌کند
2. با موفقیت کارت اعتباری مشتری را شارژ می‌کند
3. قبل از اینکه بتواند پیام را تایید (ACK) کند، سرویس سقوط می‌کند یا اتصال شبکه قطع می‌شود
4. واسطه پیام که ACK دریافت نکرده، همان پیام را مجدداً تحویل می‌دهد
5. سرویس پرداخت پیام را دوباره پردازش می‌کند و **شارژ مضاعف** رخ می‌دهد

```
Timeline of Duplicate Delivery:

  Order Service          Message Broker         Payment Service
       │                      │                       │
       │  PaymentRequested    │                       │
       ├─────────────────────▶│                       │
       │                      │  Deliver Message      │
       │                      ├──────────────────────▶│
       │                      │                       │
       │                      │              Process: Charge $99.99
       │                      │                       │
       │                      │                  ╳ CRASH / No ACK
       │                      │                       │
       │                      │  Redeliver Message    │
       │                      ├──────────────────────▶│
       │                      │                       │
       │                      │              Process: Charge $99.99
       │                      │                  (DUPLICATE!)
       │                      │                       │
       │                      │         ACK           │
       │                      │◀──────────────────────┤
       │                      │                       │
```

### دلایل رایج پیام‌های تکراری

| علت | توضیحات | فراوانی |
|-----|---------|---------|
| **سقوط مصرف‌کننده** | مصرف‌کننده پیام را پردازش می‌کند اما قبل از ACK سقوط می‌کند | گاهگاهی |
| **وقفه شبکه** | ACK ارسال شده اما در مسیر گم می‌شود؛ واسطه مجدداً تحویل می‌دهد | گاهگاهی |
| **بازتوازن واسطه** | بازتوازن پارتیشن Kafka باعث تحویل مجدد می‌شود | هنگام مقیاس‌گذاری |
| **تلاش مجدد تولیدکننده** | تولیدکننده پس از وقفه تلاش مجدد می‌کند و پیام تکراری ارسال می‌کند | گاهگاهی |
| **راه‌اندازی مجدد زیرساخت** | واسطه یا مصرف‌کننده در حین پردازش راه‌اندازی مجدد می‌شود | هنگام استقرار |
| **بازپخش دستی** | تیم عملیات پیام‌ها را برای بازیابی بازپخش می‌کند | عمدی |

### تأثیر تجاری

بدون مصرف‌کنندگان توان‌وار، پردازش تکراری می‌تواند باعث شود:

- **ضرر مالی**: شارژ مضاعف، بازپرداخت تکراری، موجودی نادرست حساب
- **فساد داده**: رکوردهای تکراری، شمارنده‌های نادرست، وضعیت ناسازگار
- **نارضایتی مشتری**: سفارش‌های تکراری، اعلان‌های مکرر، ارسال‌های متعدد
- **نقض مقررات**: تراکنش‌های مالی تکراری ممکن است قوانین نظارتی را نقض کنند

---

## ۳. نیروها

نیروهای زیر نیاز به مصرف‌کنندگان توان‌وار را شکل می‌دهند:

1. **تحویل حداقل-یک‌بار هنجار است**: بیشتر واسطه‌های پیام (Kafka، RabbitMQ، SQS) تحویل حداقل-یک‌بار را تضمین می‌کنند. تحویل دقیقاً-یک‌بار یا غیرممکن است یا دستیابی به آن در سیستم‌های توزیع‌شده بسیار پرهزینه است.

2. **عملیات‌های کسب‌وکار نباید به اشتباه تکرار شوند**: شارژ کارت اعتباری، ثبت سفارش، یا ارسال اعلان باید از دیدگاه کسب‌وکار دقیقاً یک بار اتفاق بیفتد، حتی اگر پیام زیربنایی چندین بار تحویل داده شود.

3. **شبکه غیرقابل اعتماد است**: طبق [قضیه CAP](../fundamentals/cap-theorem.md)، پارتیشن‌های شبکه اجتناب‌ناپذیر هستند. پیام‌ها، تاییدیه‌ها و پاسخ‌ها می‌توانند گم شوند یا تأخیر داشته باشند.

4. **عملکرد اهمیت دارد**: مکانیزم حذف تکرار نباید به گلوگاه تبدیل شود. سیستم‌های با توان عملیاتی بالا میلیون‌ها پیام در ثانیه پردازش می‌کنند.

5. **فضای ذخیره‌سازی محدود است**: ردیابی هر شناسه پیام پردازش شده برای همیشه عملی نیست. یک استراتژی پاکسازی لازم است.

6. **ترتیب ممکن است تضمین نشود**: پیام‌ها ممکن است بی‌ترتیب برسند، که حذف تکرار ساده مبتنی بر دنباله را ناکافی می‌سازد.

---

## ۴. راه‌حل

پردازشگرهای پیامی پیاده‌سازی کنید که بتوانند همان پیام را چندین بار به صورت ایمن پردازش کنند بدون ایجاد اثرات جانبی نادرست. چندین استراتژی برای دستیابی به این هدف وجود دارد که هر کدام برای سناریوهای مختلف مناسب هستند.

### استراتژی ۱: حذف تکرار پیام (ردیابی پیام‌های پردازش شده)

شناسه‌های پیام‌های پردازش شده را در یک پایگاه داده ردیابی کنید. قبل از پردازش یک پیام جدید، بررسی کنید آیا شناسه آن قبلاً ثبت شده است. اگر چنین بود، از پردازش صرف‌نظر کنید و نتیجه قبلی را برگردانید.

```
Idempotent Consumer Flow (Deduplication):

  Message Arrives
       │
       ▼
  ┌─────────────────────────┐
  │  Look up message_id in  │
  │  processed_messages DB  │
  └────────────┬────────────┘
               │
        ┌──────┴──────┐
        │  Already    │
        │  processed? │
        └──────┬──────┘
               │
       ┌───────┴───────┐
       │               │
      Yes              No
       │               │
       ▼               ▼
  ┌──────────┐   ┌──────────────────────────┐
  │  Return   │   │  BEGIN TRANSACTION       │
  │  cached   │   │                          │
  │  result   │   │  1. Insert message_id    │
  │  (skip)   │   │     into processed_msgs  │
  └──────────┘   │                          │
                  │  2. Execute business     │
                  │     logic                │
                  │                          │
                  │  3. Store result         │
                  │                          │
                  │  COMMIT TRANSACTION      │
                  └──────────────────────────┘
```

**جدول ردیابی پیام:**

```sql
CREATE TABLE processed_messages (
    message_id   VARCHAR(255) PRIMARY KEY,
    processed_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    result       JSONB,
    expires_at   TIMESTAMP    NOT NULL
);

CREATE INDEX idx_processed_messages_expires
    ON processed_messages (expires_at);
```

**شبه‌کد:**

```python
def handle_message(message):
    message_id = message.id

    # بررسی آیا قبلاً پردازش شده
    existing = db.query(
        "SELECT result FROM processed_messages WHERE message_id = ?",
        message_id
    )

    if existing:
        log.info(f"Duplicate message {message_id}, returning cached result")
        return existing.result

    # پردازش در یک تراکنش
    with db.transaction():
        # ابتدا شناسه پیام را درج کن (به عنوان قفل عمل می‌کند)
        db.execute(
            "INSERT INTO processed_messages (message_id, result, expires_at) "
            "VALUES (?, NULL, NOW() + INTERVAL '7 days')",
            message_id
        )

        # اجرای منطق کسب‌وکار
        result = process_business_logic(message)

        # به‌روزرسانی با نتیجه
        db.execute(
            "UPDATE processed_messages SET result = ? WHERE message_id = ?",
            json.dumps(result), message_id
        )

    return result
```

### استراتژی ۲: توان‌واری طبیعی

عملیات‌ها را طوری طراحی کنید که ذاتاً توان‌وار باشند با استفاده از مقادیر مطلق به جای تغییرات نسبی.

```
NON-IDEMPOTENT (Dangerous):
  UPDATE accounts SET balance = balance + 100 WHERE id = 123;
  -- اجرای دو بار: موجودی 200 افزایش می‌یابد (اشتباه!)

IDEMPOTENT (Safe):
  UPDATE accounts SET balance = 500 WHERE id = 123 AND version = 4;
  -- اجرای دو بار: موجودی هر دو بار 500 است (صحیح)

NON-IDEMPOTENT:
  INSERT INTO orders (customer_id, product_id, quantity) VALUES (1, 42, 2);
  -- اجرای دو بار: دو سفارش تکراری ایجاد می‌شود (اشتباه!)

IDEMPOTENT:
  INSERT INTO orders (order_id, customer_id, product_id, quantity)
  VALUES ('ord-abc-123', 1, 42, 2)
  ON CONFLICT (order_id) DO NOTHING;
  -- اجرای دو بار: فقط یک سفارش وجود دارد (صحیح)
```

### استراتژی ۳: کلیدهای توان‌واری

کلاینت یک کلید منحصر به فرد برای هر عملیات منطقی تولید می‌کند. سرور از این کلید برای شناسایی و جلوگیری از پردازش تکراری استفاده می‌کند.

```
Idempotency Key Flow:

  Client                       Server
    │                            │
    │  POST /payments            │
    │  Idempotency-Key: ik_abc1  │
    │  {amount: 99.99}           │
    ├───────────────────────────▶│
    │                            │
    │                   ┌────────┴────────┐
    │                   │ Check ik_abc1   │
    │                   │ in key store    │
    │                   └────────┬────────┘
    │                            │
    │                   Key not found:
    │                   Process payment,
    │                   store result with key
    │                            │
    │  201 Created               │
    │  {payment_id: pay_xyz}     │
    │◀───────────────────────────┤
    │                            │
    │  POST /payments (retry)    │
    │  Idempotency-Key: ik_abc1  │
    │  {amount: 99.99}           │
    ├───────────────────────────▶│
    │                            │
    │                   ┌────────┴────────┐
    │                   │ Check ik_abc1   │
    │                   │ FOUND! Return   │
    │                   │ cached response │
    │                   └────────┬────────┘
    │                            │
    │  201 Created (cached)      │
    │  {payment_id: pay_xyz}     │
    │◀───────────────────────────┤
```

### استراتژی ۴: قفل خوش‌بینانه با شماره نسخه

از شماره‌های نسخه روی موجودیت‌ها برای شناسایی و رد به‌روزرسانی‌های تکراری استفاده کنید.

```python
def handle_update_message(message):
    entity = db.find(message.entity_id)

    if entity.version != message.expected_version:
        log.info("Version mismatch, message already applied or stale")
        return  # از پردازش صرف‌نظر کن

    entity.apply_changes(message.data)
    entity.version += 1
    db.save(entity)  # اگر نسخه همزمان تغییر کرده باشد شکست می‌خورد
```

### استراتژی ۵: محدودیت‌های پایگاه داده

از محدودیت‌های یکتا برای جلوگیری از درج تکراری در سطح پایگاه داده استفاده کنید.

```sql
-- محدودیت یکتا از پردازش تکراری جلوگیری می‌کند
CREATE TABLE order_events (
    event_id    VARCHAR(255) PRIMARY KEY,  -- شناسه پیام به عنوان کلید اصلی
    order_id    VARCHAR(255) NOT NULL,
    event_type  VARCHAR(100) NOT NULL,
    payload     JSONB        NOT NULL,
    created_at  TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uq_order_event UNIQUE (order_id, event_type)
);
```

```python
def handle_order_event(message):
    try:
        db.execute(
            "INSERT INTO order_events (event_id, order_id, event_type, payload) "
            "VALUES (?, ?, ?, ?)",
            message.id, message.order_id, message.event_type, message.payload
        )
        # فقط در صورت درج موفق منطق کسب‌وکار را پردازش کن
        process_order_event(message)
    except UniqueConstraintViolation:
        log.info(f"Duplicate event {message.id}, skipping")
```

### استراتژی پاکسازی شناسه‌های پیام پردازش شده

ذخیره هر شناسه پیام به صورت نامحدود عملی نیست. یک پاکسازی مبتنی بر TTL پیاده‌سازی کنید:

```
Cleanup Approaches:

┌────────────────────────────────────────────────────────┐
│               Message ID Cleanup                       │
├────────────────────────────────────────────────────────┤
│                                                        │
│  1. TTL-Based Expiry                                   │
│     - Set expires_at = processed_at + 7 days           │
│     - Cron job deletes expired records                 │
│     - Balance: longer TTL = more storage, safer        │
│                                                        │
│  2. Sliding Window                                     │
│     - Keep only last N hours of message IDs            │
│     - Partition table by time for efficient cleanup    │
│                                                        │
│  3. Redis with TTL                                     │
│     - Store message_id in Redis with auto-expiry       │
│     - SET message:{id} "processed" EX 604800           │
│     - Fast lookups, automatic cleanup                  │
│                                                        │
│  4. Compacted Log (Kafka)                              │
│     - Use Kafka compacted topic for processed IDs      │
│     - Automatic cleanup via log compaction             │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**کار زمان‌بندی شده پاکسازی دوره‌ای:**

```sql
-- اجرای روزانه توسط cron
DELETE FROM processed_messages
WHERE expires_at < CURRENT_TIMESTAMP;
```

**ردیابی مبتنی بر Redis:**

```python
def is_processed(message_id):
    return redis.exists(f"processed:{message_id}")

def mark_processed(message_id, ttl_seconds=604800):  # 7 روز
    redis.set(f"processed:{message_id}", "1", ex=ttl_seconds)
```

---

## ۵. مثال

### مثال ۱: پردازش پرداخت با حذف تکرار

یک سرویس پرداخت رویدادهای `PaymentRequested` را دریافت می‌کند. همان رویداد می‌تواند به دلیل تلاش مجدد واسطه یا سقوط مصرف‌کننده دو بار تحویل داده شود.

```
Scenario: Duplicate PaymentRequested Event

  Event: PaymentRequested
  {
    "messageId":  "msg-7f3a-4b2c",
    "orderId":    "ord-1234",
    "customerId": "cust-5678",
    "amount":     99.99,
    "currency":   "USD"
  }
```

**تحویل اول (پردازش عادی):**

```
  Message Broker              Payment Service              Database
       │                           │                          │
       │  msg-7f3a-4b2c            │                          │
       ├──────────────────────────▶│                          │
       │                           │  SELECT * FROM           │
       │                           │  processed_messages      │
       │                           │  WHERE id = 'msg-7f3a'   │
       │                           ├─────────────────────────▶│
       │                           │                          │
       │                           │  Result: NOT FOUND       │
       │                           │◀─────────────────────────┤
       │                           │                          │
       │                           │  BEGIN TRANSACTION       │
       │                           │  1. INSERT processed_msg │
       │                           │  2. Charge credit card   │
       │                           │  3. INSERT payment       │
       │                           │  COMMIT                  │
       │                           ├─────────────────────────▶│
       │                           │                          │
       │         ACK               │  Committed               │
       │◀──────────────────────────┤◀─────────────────────────┤
```

**تحویل دوم (تکراری، به صورت ایمن مدیریت شده):**

```
  Message Broker              Payment Service              Database
       │                           │                          │
       │  msg-7f3a-4b2c (retry)    │                          │
       ├──────────────────────────▶│                          │
       │                           │  SELECT * FROM           │
       │                           │  processed_messages      │
       │                           │  WHERE id = 'msg-7f3a'   │
       │                           ├─────────────────────────▶│
       │                           │                          │
       │                           │  Result: FOUND           │
       │                           │  (already processed)     │
       │                           │◀─────────────────────────┤
       │                           │                          │
       │         ACK               │  Skip processing         │
       │◀──────────────────────────┤  Log: "Duplicate msg"    │
       │                           │                          │
```

**نتیجه:** مشتری دقیقاً یک بار شارژ می‌شود، حتی اگر پیام دو بار تحویل داده شده باشد.

### مثال ۲: ایجاد سفارش با کلید توان‌واری

یک کلاینت با استفاده از کلید توان‌واری سفارش ایجاد می‌کند تا از سفارش‌های تکراری در هنگام تلاش مجدد جلوگیری شود.

```python
# سمت کلاینت
import uuid

idempotency_key = str(uuid.uuid4())  # "ik-9a8b7c6d"

response = http.post(
    "/api/orders",
    headers={"Idempotency-Key": idempotency_key},
    json={
        "product_id": "prod-42",
        "quantity": 2,
        "shipping_address": "123 Main St"
    }
)

# اگر وقفه شبکه رخ دهد، کلاینت با همان کلید تلاش مجدد می‌کند
response = http.post(
    "/api/orders",
    headers={"Idempotency-Key": idempotency_key},  # همان کلید
    json={
        "product_id": "prod-42",
        "quantity": 2,
        "shipping_address": "123 Main St"
    }
)
# سرور همان سفارش درخواست اول را برمی‌گرداند
```

```python
# سمت سرور
def create_order(request):
    idempotency_key = request.headers.get("Idempotency-Key")

    # بررسی نتیجه موجود
    cached = db.query(
        "SELECT response FROM idempotency_keys WHERE key = ?",
        idempotency_key
    )
    if cached:
        return cached.response  # برگرداندن همان پاسخ

    with db.transaction():
        # قفل کلید توان‌واری
        db.execute(
            "INSERT INTO idempotency_keys (key, status) VALUES (?, 'processing')",
            idempotency_key
        )

        # ایجاد سفارش
        order = Order.create(
            product_id=request.json["product_id"],
            quantity=request.json["quantity"],
            shipping_address=request.json["shipping_address"]
        )

        # ذخیره پاسخ
        response = {"order_id": order.id, "status": "created"}
        db.execute(
            "UPDATE idempotency_keys SET status = 'completed', response = ? WHERE key = ?",
            json.dumps(response), idempotency_key
        )

    return response
```

---

## ۶. مزایا و معایب

### مزایا

| مزیت | توضیحات |
|------|---------|
| **مدیریت ایمن تکرار** | مصرف‌کنندگان از دیدگاه کسب‌وکار پیام‌ها را دقیقاً یک بار پردازش می‌کنند، صرف‌نظر از تعداد تحویل |
| **فعال‌سازی تحویل حداقل-یک‌بار** | سیستم‌ها می‌توانند به جای معنای پیچیده دقیقاً-یک‌بار، به معنای ساده‌تر و مقاوم‌تر حداقل-یک‌بار تکیه کنند |
| **ساده‌سازی منطق تلاش مجدد** | تولیدکنندگان و واسطه‌ها می‌توانند آزادانه تلاش مجدد کنند بدون ترس از ایجاد اثرات جانبی نادرست |
| **تاب‌آوری در برابر خرابی** | سقوط مصرف‌کننده، وقفه شبکه و راه‌اندازی مجدد واسطه باعث فساد داده نمی‌شود |
| **انعطاف‌پذیری عملیاتی** | تیم‌های عملیات می‌توانند با اطمینان پیام‌ها را برای بازیابی یا پردازش مجدد بازپخش کنند |
| **سازگاری با Event Sourcing** | به طور طبیعی با [معماری‌های رویداد-محور](../event-driven/event-driven-architecture.md) کار می‌کند جایی که رویدادها ممکن است بازپخش شوند |

### معایب

| عیب | توضیحات | راه‌حل |
|-----|---------|--------|
| **فضای ذخیره‌سازی اضافی** | ردیابی شناسه‌های پیام پردازش شده به فضای پایگاه داده نیاز دارد | پاکسازی مبتنی بر TTL، Redis با انقضای خودکار |
| **سربار عملکرد** | هر پیام قبل از پردازش نیاز به جستجو دارد | استفاده از ذخایر سریع (Redis)، ایندکس‌گذاری، کش |
| **پیچیدگی پیاده‌سازی** | پیچیدگی کد به هر مصرف‌کننده اضافه می‌شود | انتزاع در میان‌افزار یا فریم‌ورک |
| **همگام‌سازی ساعت** | پاکسازی مبتنی بر TTL به مُهرهای زمانی سازگار بستگی دارد | استفاده از ساعت‌های منطقی یا مُهرهای زمانی واسطه |
| **محدوده تراکنش** | بررسی حذف تکرار و منطق کسب‌وکار باید اتمی باشند | استفاده از تراکنش‌های پایگاه داده یا الگوهای دو مرحله‌ای |
| **بار تولید کلید** | کلیدهای توان‌واری نیاز به هماهنگی سمت کلاینت دارند | ارائه SDK/کتابخانه برای تولید کلید |

### ماتریس تصمیم‌گیری: کدام استراتژی را استفاده کنیم

| استراتژی | بهترین برای | پیچیدگی | عملکرد | هزینه ذخیره‌سازی |
|----------|------------|---------|--------|-----------------|
| **حذف تکرار پیام** | عمومی، هر مصرف‌کننده‌ای | متوسط | متوسط | متوسط |
| **توان‌واری طبیعی** | به‌روزرسانی ساده وضعیت (SET در مقابل INCREMENT) | کم | بالا | هیچ |
| **کلیدهای توان‌واری** | نقاط پایانی API، درخواست‌های آغاز شده توسط کلاینت | متوسط | متوسط | متوسط |
| **قفل خوش‌بینانه** | به‌روزرسانی موجودیت با نسخه‌بندی | کم | بالا | کم |
| **محدودیت‌های پایگاه داده** | عملیات‌های سنگین درج | کم | بالا | کم |

---

## ۷. الگوهای مرتبط

- **[پیام‌رسانی](./messaging.md)** -- زمینه اصلی این الگو. مصرف‌کنندگان توان‌وار در هر معماری مبتنی بر پیام‌رسانی که از تحویل حداقل-یک‌بار استفاده می‌کند ضروری هستند.

- **[صندوق خروجی تراکنشی (Transactional Outbox)](../messaging/transactional-outbox.md)** -- یک الگوی مرتبط پیام‌رسانی قابل اعتماد که اطمینان می‌دهد پیام‌ها به صورت اتمی با تغییرات پایگاه داده منتشر شوند. حتی با صندوق خروجی تراکنشی که انتشار قابل اعتماد را تضمین می‌کند، مصرف‌کنندگان همچنان به توان‌واری نیاز دارند زیرا فرآیند رله صندوق خروجی ممکن است تکراری تحویل دهد.

- **[معماری رویداد-محور](../event-driven/event-driven-architecture.md)** -- مصرف‌کنندگان توان‌وار در سیستم‌های رویداد-محور حیاتی هستند جایی که رویدادها می‌توانند بازپخش شوند، مجدداً تحویل داده شوند، یا بی‌ترتیب پردازش شوند.

- **[الگوی Saga](../distributed-transactions/saga.md)** -- در جریان‌های کاری مبتنی بر Saga، تراکنش‌های جبرانی و پردازشگرهای مرحله‌ای باید توان‌وار باشند زیرا ارکستراتور یا کوریوگرافی ممکن است مراحل ناموفق را تلاش مجدد کند.

- **[CQRS](../data-patterns/cqrs.md)** -- در معماری‌های CQRS، پردازشگرهای رویداد که مدل‌های خواندن را به‌روزرسانی می‌کنند باید توان‌وار باشند زیرا رویدادها می‌توانند برای بازسازی پروجکشن‌ها بازپخش شوند.

- **[Circuit Breaker](../resilience/circuit-breaker.md)** -- وقتی یک Circuit Breaker باز شده و سپس بسته می‌شود، پیام‌های تلاش مجدد شده ممکن است به عنوان تکراری برسند. مصرف‌کنندگان توان‌وار اطمینان می‌دهند که این تلاش‌های مجدد ایمن هستند.

---

## ۸. استفاده در دنیای واقعی

### Stripe: کلیدهای توان‌واری

Stripe یکی از شناخته‌شده‌ترین پیاده‌سازی‌های الگوی کلید توان‌واری است. هر درخواست API تغییردهنده یک هدر `Idempotency-Key` می‌پذیرد.

```
POST /v1/charges
Idempotency-Key: ik_KG5LxwFBepaKHyUD
Content-Type: application/json

{
  "amount": 2000,
  "currency": "usd",
  "source": "tok_visa"
}
```

**جزئیات پیاده‌سازی Stripe:**
- کلیدها به مدت ۲۴ ساعت ذخیره می‌شوند و سپس به طور خودکار منقضی می‌شوند
- اگر یک درخواست در حال انجام باشد، یک تکراری همزمان `409 Conflict` برمی‌گرداند
- پاسخ توان‌وار شامل دقیقاً همان کد وضعیت و بدنه است
- کلیدها به ازای هر کلید API محدود هستند (حساب‌های مختلف می‌توانند از همان کلید استفاده کنند)

### AWS SQS: مهلت نمایش و حذف تکرار

Amazon SQS پشتیبانی داخلی برای پردازش توان‌وار ارائه می‌دهد:

- **صف‌های استاندارد**: تحویل حداقل-یک‌بار؛ مصرف‌کنندگان باید حذف تکرار خود را پیاده‌سازی کنند
- **صف‌های FIFO**: حذف تکرار داخلی با استفاده از `MessageDeduplicationId` (پنجره ۵ دقیقه‌ای)

```
SQS FIFO Deduplication:

  Producer                    SQS FIFO Queue             Consumer
     │                             │                        │
     │  Send(DeduplicationId=X)    │                        │
     ├────────────────────────────▶│                        │
     │                             │  Store message         │
     │                             │                        │
     │  Send(DeduplicationId=X)    │                        │
     ├────────────────────────────▶│                        │
     │                             │  Reject (duplicate     │
     │                             │  within 5-min window)  │
     │                             │                        │
     │                             │  Deliver message       │
     │                             ├───────────────────────▶│
     │                             │                        │
```

### گروه‌های مصرف‌کننده Kafka: مدیریت آفست

مصرف‌کنندگان Apache Kafka موقعیت (آفست) خود را در هر پارتیشن ردیابی می‌کنند. پردازش توان‌وار لازم است زیرا آفست‌ها ممکن است همیشه قبل از سقوط ثبت نشده باشند.

```
Kafka Offset and Idempotency:

  Partition: [msg-0] [msg-1] [msg-2] [msg-3] [msg-4]
                                 ^
                         committed offset = 2

  Consumer processes msg-2, msg-3
  Consumer crashes before committing offset to 4

  On restart:
  Consumer re-reads from offset 2 (msg-2 is redelivered)
  Idempotent handler detects msg-2 is already processed → skip
  Processes msg-3 (again, idempotent) and msg-4
  Commits offset to 5
```

**تولیدکننده توان‌وار Kafka:**

Kafka همچنین از تولیدکنندگان توان‌وار (از طریق `enable.idempotence=true`) پشتیبانی می‌کند که از ارسال پیام‌های تکراری توسط تولیدکننده به واسطه جلوگیری می‌کند. با این حال، این فقط تکرارهای تولیدکننده-به-واسطه را پوشش می‌دهد؛ توان‌واری سمت مصرف‌کننده همچنان ضروری است.

### PayPal: الگوی شناسه درخواست

PayPal از هدر `PayPal-Request-Id` برای درخواست‌های توان‌وار استفاده می‌کند:

```
POST /v2/checkout/orders
PayPal-Request-Id: req-unique-7a8b9c
Content-Type: application/json

{
  "intent": "CAPTURE",
  "purchase_units": [{
    "amount": {"currency_code": "USD", "value": "100.00"}
  }]
}
```

اگر همان `PayPal-Request-Id` دوباره ارسال شود، PayPal پاسخ اصلی را بدون ایجاد سفارش جدید برمی‌گرداند.

---

## ۹. خلاصه

الگوی مصرف‌کننده توان‌وار برای ساخت سیستم‌های قابل اعتماد مبتنی بر پیام در [معماری میکروسرویس‌ها](../architecture/microservices.md) ضروری است. از آنجایی که واسطه‌های پیام تحویل حداقل-یک‌بار را تضمین می‌کنند، مصرف‌کنندگان باید برای مدیریت ایمن تکرارها طراحی شوند.

**نکات کلیدی:**

| جنبه | جزئیات |
|------|--------|
| **مسئله** | پیام‌های تکراری باعث اثرات جانبی نادرست می‌شوند (شارژ مضاعف، رکوردهای تکراری) |
| **راه‌حل** | طراحی پردازشگرهایی که صرف‌نظر از تعداد تحویل همان نتیجه را تولید می‌کنند |
| **استراتژی‌ها** | حذف تکرار پیام، توان‌واری طبیعی، کلیدهای توان‌واری، قفل خوش‌بینانه، محدودیت‌های پایگاه داده |
| **ذخیره‌سازی** | ردیابی شناسه‌های پیام پردازش شده با پاکسازی مبتنی بر TTL |
| **دنیای واقعی** | Stripe (کلیدهای توان‌واری)، AWS SQS (حذف تکرار FIFO)، Kafka (آفست + تولیدکنندگان توان‌وار) |

**به یاد داشته باشید:** توان‌واری در سیستم‌های توزیع‌شده اختیاری نیست. هر مصرف‌کننده‌ای که پیام‌ها را از یک واسطه پردازش می‌کند باید فرض کند که تکرارها خواهند رسید و آن‌ها را به درستی مدیریت کند. الگوی [صندوق خروجی تراکنشی (Transactional Outbox)](../messaging/transactional-outbox.md) انتشار قابل اعتماد را تضمین می‌کند، اما مصرف‌کنندگان توان‌وار پردازش قابل اعتماد را تضمین می‌کنند.

---

**منابع:**
- Chris Richardson, "Microservices Patterns" (2018) - فصل پیام‌رسانی و توان‌واری
- مستندات API Stripe: درخواست‌های توان‌وار
- مستندات AWS: حذف تکرار پیام Amazon SQS
- مستندات Apache Kafka: طراحی تولیدکننده و مصرف‌کننده توان‌وار
- Martin Kleppmann, "Designing Data-Intensive Applications" (2017) - فصل پردازش جریان
