# الگوی Polling Publisher

## ۱. مقدمه

در معماری میکروسرویس‌ها، انتشار مطمئن رویدادهای دامنه به یک واسطه پیام (Message Broker) یک چالش اساسی است. الگوی **Polling Publisher** مکانیزمی ساده برای تحویل رویدادها از جدول outbox به واسطه پیام فراهم می‌کند، با پرس‌وجوی دوره‌ای پایگاه داده برای رویدادهای منتشر نشده.

این الگو معمولاً به عنوان **مکانیزم تحویل** برای الگوی [Transactional Outbox](./transactional-outbox.md) استفاده می‌شود. در حالی که Transactional Outbox اطمینان می‌دهد که رویدادها به صورت اتمیک همراه با داده‌های کسب‌وکار نوشته می‌شوند، Polling Publisher مسئول مرحله دوم است: انتقال آن رویدادها از پایگاه داده به واسطه پیام (مانند [Kafka یا RabbitMQ](./rabbitmq-vs-kafka.md)).

Polling Publisher به دلیل **سادگی** و **عدم وابستگی به پایگاه داده خاص** ارزشمند است. هیچ ویژگی خاص پایگاه داده یا زیرساخت خاصی فراتر از یک فرآیند پس‌زمینه و یک پایگاه داده رابطه‌ای (یا NoSQL) نیاز ندارد.

```
+------------------+       +------------------+       +------------------+
|                  |       |                  |       |                  |
|   Application    | ----> |   Outbox Table   | ----> | Message Broker   |
|   (writes to     |       |   (in database)  |       | (Kafka, RabbitMQ |
|    outbox)       |       |                  |       |  etc.)           |
|                  |       |                  |       |                  |
+------------------+       +--------+---------+       +------------------+
                                    ^
                                    |
                           +--------+---------+
                           |                  |
                           | Polling Publisher |
                           | (background job) |
                           |                  |
                           +------------------+
```

## ۲. زمینه و مسئله

هنگام استفاده از الگوی [Transactional Outbox](./transactional-outbox.md)، برنامه رویدادهای دامنه را در همان تراکنش پایگاه داده که داده‌های کسب‌وکار را تغییر می‌دهد، در جدول outbox می‌نویسد. این اتمی بودن را تضمین می‌کند — یا هم داده‌های کسب‌وکار و هم رویداد ذخیره می‌شوند، یا هیچ‌کدام.

با این حال، رویدادهایی که در جدول outbox نشسته‌اند تا زمانی که به واسطه پیام نرسند بی‌فایده هستند، جایی که مصرف‌کنندگان پایین‌دستی بتوانند آن‌ها را پردازش کنند. مسئله اصلی این است:

> **چگونه رویدادها را به طور مطمئن از جدول outbox به واسطه پیام منتقل کنیم؟**

دو رویکرد اصلی برای حل این مسئله وجود دارد:

1. **Polling Publisher** (این الگو) — یک فرآیند پس‌زمینه به صورت دوره‌ای جدول outbox را پرس‌وجو می‌کند
2. **[Transaction Log Tailing](./transaction-log-tailing.md)** — لاگ تراکنش پایگاه داده (CDC) را برای ضبط رویدادهای جدید می‌خواند

هر رویکرد مصالحه‌های متفاوتی از نظر پیچیدگی، تاخیر و نیازمندی‌های زیرساختی دارد.

## ۳. نیروهای مؤثر

چندین نیرو بر تصمیم برای استفاده از الگوی Polling Publisher تأثیر می‌گذارند:

- **سادگی بر بلادرنگ**: سیستم می‌تواند تأخیر کوچکی (ده‌ها تا صدها میلی‌ثانیه) بین نوشتن رویداد و انتشار آن به واسطه را تحمل کند. تحویل کاملاً بلادرنگ لازم نیست.

- **رویکرد مستقل از پایگاه داده**: راه‌حل باید با هر پایگاه داده‌ای که از پرس‌وجوهای پایه SQL (SELECT، UPDATE، DELETE) پشتیبانی می‌کند کار کند. بدون وابستگی به ویژگی‌های خاص پایگاه داده مانند Change Data Capture (CDC)، اسلات‌های تکرار منطقی، یا دسترسی به لاگ تراکنش.

- **بدون نیاز به زیرساخت خاص**: تیم نمی‌خواهد زیرساخت اضافی مانند Debezium، Maxwell، یا سایر کانکتورهای CDC را مستقر یا مدیریت کند. یک فرآیند پس‌زمینه ساده کافی است.

- **سادگی عملیاتی**: تیم راه‌حلی را ترجیح می‌دهد که درک، اشکال‌زدایی و اداره آن آسان باشد. هر کسی که بتواند SQL بخواند، می‌تواند Polling Publisher را درک کند.

- **نیازمندی‌های سازگاری**: رویدادها باید حداقل یک‌بار تحویل داده شوند. تحویل تکراری قابل قبول است اگر مصرف‌کنندگان ایدمپوتنت باشند، اما رویدادها هرگز نباید از دست بروند.

- **محدودیت‌های ترتیب**: رویدادها برای همان aggregate (مثلاً همان سفارش) باید به ترتیب ایجاد شده منتشر شوند.

## ۴. راه‌حل

الگوی Polling Publisher از یک **فرآیند پس‌زمینه** (وظیفه زمان‌بندی شده، cron task، یا worker اختصاصی) استفاده می‌کند که در فواصل منظم اجرا می‌شود و چرخه زیر را انجام می‌دهد:

### چرخه Polling

```
    +------------------+
    |  Wait for next   |
    |  poll interval   |
    +--------+---------+
             |
             v
    +------------------+
    |  Query outbox    |
    |  for unpublished |
    |  events          |
    +--------+---------+
             |
             v
    +------------------+
    |  Any events      |-----> No -----> (back to wait)
    |  found?          |
    +--------+---------+
             |
            Yes
             |
             v
    +------------------+
    |  Publish batch   |
    |  to message      |
    |  broker          |
    +--------+---------+
             |
             v
    +------------------+
    |  Mark events as  |
    |  published in    |
    |  outbox          |
    +--------+---------+
             |
             v
        (back to wait)
```

### الگوریتم Polling

الگوریتم اصلی شامل مراحل زیر است:

**مرحله ۱: پرس‌وجوی رویدادهای منتشر نشده**

```sql
SELECT id, aggregate_type, aggregate_id, event_type, payload, created_at
FROM outbox
WHERE published = FALSE
ORDER BY created_at ASC
LIMIT 100;
```

عبارت `ORDER BY created_at ASC` اطمینان می‌دهد که رویدادها به ترتیب ایجاد پردازش می‌شوند. عبارت `LIMIT` اندازه دسته را کنترل می‌کند تا از غلبه بر واسطه یا مصرف بیش از حد حافظه جلوگیری شود.

**مرحله ۲: انتشار به واسطه پیام**

برای هر رویداد (یا به صورت دسته‌ای)، payload رویداد را به topic یا صف مناسب در واسطه پیام منتشر کنید.

```
For each event in batch:
    topic = derive_topic(event.aggregate_type, event.event_type)
    key   = event.aggregate_id
    value = event.payload
    publish(topic, key, value)
```

استفاده از `aggregate_id` به عنوان کلید پیام اطمینان می‌دهد که رویدادهای مربوط به همان aggregate در همان پارتیشن (در Kafka) قرار می‌گیرند و ترتیب حفظ می‌شود.

**مرحله ۳: علامت‌گذاری رویدادها به عنوان منتشر شده**

```sql
UPDATE outbox
SET published = TRUE, published_at = NOW()
WHERE id IN (?, ?, ?, ...);
```

### طرح جدول Outbox

یک جدول outbox معمولی به شکل زیر است:

```sql
CREATE TABLE outbox (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    aggregate_type  VARCHAR(255) NOT NULL,
    aggregate_id    VARCHAR(255) NOT NULL,
    event_type      VARCHAR(255) NOT NULL,
    payload         TEXT NOT NULL,              -- JSON event payload
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    published       BOOLEAN NOT NULL DEFAULT FALSE,
    published_at    TIMESTAMP NULL,
    retry_count     INT NOT NULL DEFAULT 0,
    idempotency_key VARCHAR(255) UNIQUE NOT NULL
);

CREATE INDEX idx_outbox_unpublished ON outbox (published, created_at);
```

### ماشین حالت رویداد

هر رویداد در outbox از حالت‌های زیر عبور می‌کند:

```
  +-------------+       +-----------+       +-------------+
  |             |       |           |       |             |
  |   PENDING   | ----> | PUBLISHING| ----> |  PUBLISHED  |
  |             |       |           |       |             |
  +------+------+       +-----+-----+       +-------------+
         |                     |
         |                     | (failure)
         |                     v
         |              +------+------+
         |              |             |
         +------------> |   FAILED    | ---> (retry -> PUBLISHING)
                        |             |
                        +------+------+
                               |
                               | (max retries exceeded)
                               v
                        +------+------+
                        |             |
                        | DEAD LETTER |
                        |             |
                        +-------------+
```

| حالت        | توضیحات                                                        |
|-------------|----------------------------------------------------------------|
| PENDING     | رویداد در outbox نوشته شده، هنوز برداشته نشده                 |
| PUBLISHING  | رویداد توسط poller برداشته شده، در حال ارسال به واسطه         |
| PUBLISHED   | با موفقیت به واسطه پیام منتشر شده                              |
| FAILED      | انتشار شکست خورده، دوباره تلاش خواهد شد                       |
| DEAD LETTER | حداکثر تلاش‌های مجدد تمام شده، نیاز به مداخله دستی            |

### مدیریت خطاها

خطاها می‌توانند در نقاط مختلف رخ دهند:

1. **واسطه در دسترس نیست**: واسطه پیام به طور موقت از کار افتاده است. poller باید استثنا را بگیرد، `retry_count` را افزایش دهد و رویداد را به عنوان منتشر نشده برای چرخه بعدی polling باقی بگذارد.

2. **شکست جزئی دسته**: برخی رویدادها در یک دسته با موفقیت منتشر می‌شوند، برخی دیگر شکست می‌خورند. فقط رویدادهای موفق را به عنوان منتشر شده علامت‌گذاری کنید.

3. **انتشار تکراری**: اگر poller رویدادی را منتشر کند اما قبل از علامت‌گذاری آن به عنوان منتشر شده از کار بیفتد، poll بعدی همان رویداد را دوباره منتشر می‌کند. به همین دلیل **مصرف‌کنندگان باید ایدمپوتنت باشند**. فیلد `idempotency_key` در جدول outbox به مصرف‌کنندگان پایین‌دستی کمک می‌کند تکراری‌ها را شناسایی و رد کنند.

4. **مدیریت Dead Letter**: پس از تعداد قابل تنظیم تلاش‌های مجدد (مثلاً ۵ بار)، رویداد را به حالت Dead Letter منتقل کنید و تیم عملیات را برای بررسی دستی هشدار دهید.

```
Retry Strategy:
    Attempt 1: immediate
    Attempt 2: wait 1 second
    Attempt 3: wait 5 seconds
    Attempt 4: wait 30 seconds
    Attempt 5: wait 2 minutes
    After 5 attempts: mark as DEAD LETTER, alert ops team
```

### تضمین‌های ترتیب

رویدادها برای **همان aggregate** باید به ترتیب منتشر شوند. Polling Publisher این کار را با موارد زیر تضمین می‌کند:

1. پرس‌وجوی رویدادها مرتب شده بر اساس `created_at ASC`
2. استفاده از `aggregate_id` به عنوان کلید پیام (تضمین تحویل در همان پارتیشن در Kafka)
3. پردازش ترتیبی رویدادها به ازای هر aggregate در هر دسته

> **مهم**: اگر رویدادی برای aggregate X شکست بخورد، تمام رویدادهای بعدی برای aggregate X باید مسدود شوند تا رویداد شکست‌خورده با موفقیت منتشر شود. در غیر این صورت، مصرف‌کنندگان رویدادها را نامرتب می‌بینند.

### تنظیم فاصله Polling

فاصله polling، اهرم مصالحه بین **تأخیر** و **بار پایگاه داده** است:

| فاصله Polling  | تأخیر           | بار پایگاه داده | مورد استفاده                      |
|----------------|-----------------|-----------------|-----------------------------------|
| ۱۰ میلی‌ثانیه  | خیلی کم (~10ms) | خیلی زیاد       | نیازمندی‌های نزدیک به بلادرنگ     |
| ۱۰۰ میلی‌ثانیه | کم (~100ms)     | متوسط           | اکثر میکروسرویس‌ها (پیشنهادی)    |
| ۱ ثانیه        | متوسط (~1s)     | کم              | تحویل رویداد غیربحرانی           |
| ۵ ثانیه        | زیاد (~5s)      | خیلی کم         | دسته‌ای یا اولویت پایین          |

برای اکثر سیستم‌ها، فاصله polling **۱۰۰ میلی‌ثانیه** تعادل خوبی فراهم می‌کند.

### استراتژی پاکسازی

رویدادهای منتشر شده باید به صورت دوره‌ای از outbox حذف شوند تا از رشد بی‌رویه جدول جلوگیری شود:

```sql
-- Delete published events older than 7 days
DELETE FROM outbox
WHERE published = TRUE
  AND published_at < NOW() - INTERVAL 7 DAY
LIMIT 10000;
```

این پاکسازی را به عنوان یک وظیفه زمان‌بندی شده جداگانه (مثلاً هر ساعت یا روزی یک‌بار) اجرا کنید. از `LIMIT` استفاده کنید تا از عملیات حذف طولانی مدت که ممکن است جدول را قفل کند جلوگیری شود.

## ۵. مثال

یک سیستم تجارت الکترونیک را در نظر بگیرید که ثبت سفارش باید یک رویداد `OrderPlaced` را به Kafka منتشر کند تا سرویس‌های پرداخت، موجودی و اطلاع‌رسانی بتوانند واکنش نشان دهند.

### کد برنامه (نوشتن در Outbox)

```python
def place_order(order_data):
    with database.transaction():
        order = Order.create(order_data)

        outbox_event = OutboxEvent(
            aggregate_type="Order",
            aggregate_id=str(order.id),
            event_type="OrderPlaced",
            payload=json.dumps({
                "order_id": order.id,
                "customer_id": order.customer_id,
                "total": order.total,
                "items": order.items
            }),
            idempotency_key=f"order-placed-{order.id}"
        )
        outbox_event.save()

    return order
```

### Polling Publisher (وظیفه پس‌زمینه)

```python
class PollingPublisher:
    def __init__(self, db, kafka_producer, batch_size=100, poll_interval_ms=100):
        self.db = db
        self.kafka = kafka_producer
        self.batch_size = batch_size
        self.poll_interval = poll_interval_ms / 1000.0  # convert to seconds
        self.max_retries = 5

    def run(self):
        """Main polling loop."""
        while True:
            events = self.fetch_unpublished_events()

            if events:
                self.publish_batch(events)

            time.sleep(self.poll_interval)

    def fetch_unpublished_events(self):
        return self.db.execute("""
            SELECT id, aggregate_type, aggregate_id, event_type,
                   payload, created_at, idempotency_key
            FROM outbox
            WHERE published = FALSE AND retry_count < %s
            ORDER BY created_at ASC
            LIMIT %s
        """, [self.max_retries, self.batch_size])

    def publish_batch(self, events):
        published_ids = []
        failed_ids = []

        for event in events:
            try:
                topic = f"{event.aggregate_type.lower()}.events"
                self.kafka.produce(
                    topic=topic,
                    key=event.aggregate_id,
                    value=event.payload,
                    headers={"idempotency_key": event.idempotency_key}
                )
                published_ids.append(event.id)
            except Exception as e:
                logger.error(f"Failed to publish event {event.id}: {e}")
                failed_ids.append(event.id)

        # Flush to ensure all messages are delivered
        self.kafka.flush()

        if published_ids:
            self.db.execute("""
                UPDATE outbox
                SET published = TRUE, published_at = NOW()
                WHERE id IN (%s)
            """, published_ids)

        if failed_ids:
            self.db.execute("""
                UPDATE outbox
                SET retry_count = retry_count + 1
                WHERE id IN (%s)
            """, failed_ids)
```

### جدول زمانی اجرا

```
Time (ms)  |  Action
-----------+-------------------------------------------------------
  0        |  Poller wakes up
  5        |  SELECT 3 unpublished events from outbox
  10       |  Publish event #1 (OrderPlaced) to Kafka -> success
  15       |  Publish event #2 (OrderUpdated) to Kafka -> success
  20       |  Publish event #3 (OrderCancelled) to Kafka -> FAIL
  25       |  UPDATE events #1, #2 as published
  26       |  UPDATE event #3: retry_count = retry_count + 1
  30       |  Poller sleeps for 100ms
  ...      |  ...
  130      |  Poller wakes up
  135      |  SELECT unpublished events (includes event #3 again)
  140      |  Publish event #3 (OrderCancelled) to Kafka -> success
  145      |  UPDATE event #3 as published
  150      |  Poller sleeps for 100ms
```

## ۶. مزایا و معایب

### مزایا

| مزیت                            | توضیحات                                                                                               |
|---------------------------------|-------------------------------------------------------------------------------------------------------|
| پیاده‌سازی ساده                 | فقط نیاز به پرس‌وجوهای SQL و یک حلقه پس‌زمینه دارد. بدون زیرساخت پیچیده.                            |
| مستقل از پایگاه داده            | با هر پایگاه داده‌ای کار می‌کند: PostgreSQL، MySQL، MongoDB، SQL Server و غیره.                       |
| بدون نیاز به زیرساخت CDC        | بر خلاف [Transaction Log Tailing](./transaction-log-tailing.md)، نیازی به Debezium، Maxwell یا ابزار مشابه نیست. |
| درک آسان                        | کل مکانیزم با یک SELECT، یک فراخوانی publish و یک UPDATE قابل توضیح است.                             |
| اشکال‌زدایی آسان                | مستقیماً جدول outbox را پرس‌وجو کنید تا رویدادهای در انتظار، منتشر شده یا شکست‌خورده را ببینید.      |
| قابل حمل                        | بدون وابستگی به فروشنده خاص پایگاه داده یا ابزار CDC.                                                |
| تحویل حداقل یک‌بار مطمئن       | ترکیب با ایدمپوتنسی مصرف‌کننده، تضمین می‌کند هیچ رویدادی از دست نمی‌رود.                             |

### معایب

| عیب                             | توضیحات                                                                                               |
|---------------------------------|-------------------------------------------------------------------------------------------------------|
| سربار Polling                   | پرس‌وجوهای مکرر از پایگاه داده حتی زمانی که رویداد جدیدی وجود ندارد.                                |
| تأخیر بالاتر از CDC             | رویدادها با تأخیری برابر فاصله poll (معمولاً ۱۰۰ms تا ۵s) تحویل داده می‌شوند.                       |
| بار پایگاه داده                 | Polling مکرر بار خواندن به پایگاه داده اضافه می‌کند که در مقیاس بزرگ قابل توجه است.                  |
| چالش‌های مقیاس‌پذیری           | اجرای چند poller نیاز به هماهنگی (مثلاً قفل‌گذاری) برای جلوگیری از انتشار تکراری دارد.              |
| رشد جدول                        | بدون پاکسازی، جدول outbox بی‌رویه رشد می‌کند و عملکرد پرس‌وجو کاهش می‌یابد.                        |
| واقعاً بلادرنگ نیست             | فاصله poll تأخیر ذاتی‌ای ایجاد می‌کند که رویکردهای مبتنی بر CDC از آن اجتناب می‌کنند.              |

### مقایسه Polling Publisher با Transaction Log Tailing

| جنبه                    | Polling Publisher                          | Transaction Log Tailing (CDC)                   |
|-------------------------|--------------------------------------------|-------------------------------------------------|
| تأخیر                   | فاصله poll (100ms - ثانیه‌ها)              | نزدیک به بلادرنگ (میلی‌ثانیه)                  |
| زیرساخت                 | هیچ (فقط یک فرآیند پس‌زمینه)              | نیاز به ابزار CDC (Debezium، Maxwell و غیره)   |
| سازگاری با پایگاه داده  | هر پایگاه داده SQL/NoSQL                   | بستگی به پشتیبانی ابزار CDC دارد              |
| بار پایگاه داده         | پرس‌وجوهای خواندن در هر چرخه poll         | خواندن لاگ تراکنش (حداقل تأثیر بر جدول)       |
| پیچیدگی                 | بسیار کم                                   | متوسط تا زیاد                                  |
| مقیاس‌پذیری             | نیاز به هماهنگی برای چند poller            | کانکتور CDC مقیاس‌پذیری را مدیریت می‌کند      |
| اشکال‌زدایی             | آسان (پرس‌وجوی جدول outbox)               | سخت‌تر (بررسی لاگ‌ها/آفست‌های کانکتور CDC)    |
| سربار عملیاتی           | کم                                         | متوسط تا زیاد                                  |
| بهترین برای              | راه‌اندازی‌های ساده‌تر، مقیاس کوچک تا متوسط | نیازمندی‌های توان بالا و تأخیر کم             |

## ۷. الگوهای مرتبط

- **[Transactional Outbox](./transactional-outbox.md)** — الگویی که Polling Publisher آن را تکمیل می‌کند. outbox ذخیره‌سازی اتمیک رویداد را تضمین می‌کند؛ Polling Publisher آن رویدادها را به واسطه تحویل می‌دهد.

- **[Transaction Log Tailing](./transaction-log-tailing.md)** — مکانیزم تحویل جایگزین برای الگوی Transactional Outbox. به جای polling از CDC (Change Data Capture) برای خواندن رویدادهای جدید از لاگ تراکنش پایگاه داده استفاده می‌کند.

- **[معماری رویداد-محور](../event-driven/event-driven-architecture.md)** — سبک معماری گسترده‌تر که بر انتشار و مصرف مطمئن رویدادها در سراسر سرویس‌ها تکیه دارد.

- **[الگوی Saga](../distributed-transactions/saga.md)** — Sagaها به انتشار مطمئن رویداد برای هماهنگی تراکنش‌های توزیع‌شده وابسته هستند. Polling Publisher می‌تواند به عنوان مکانیزم تحویل برای رویدادهای saga عمل کند.

- **[CQRS](../data-patterns/cqrs.md)** — مدل‌های خواندن CQRS اغلب با مصرف رویدادهای دامنه به‌روزرسانی می‌شوند. Polling Publisher اطمینان می‌دهد آن رویدادها به واسطه پیام برای همگام‌سازی مدل خواندن می‌رسند.

- **[مقایسه RabbitMQ و Kafka](./rabbitmq-vs-kafka.md)** — درک واسطه پیام هدف در پیکربندی Polling Publisher (اندازه دسته، تأییدیه‌ها، پارتیشن‌بندی) کمک می‌کند.

- **[معماری Microservices](../architecture/microservices.md)** — Polling Publisher یک الگوی زیرساختی کلیدی در میکروسرویس‌ها برای ارتباط مطمئن بین سرویس‌ها است.

## ۸. استفاده در دنیای واقعی

الگوی Polling Publisher به طور گسترده در سیستم‌های تولیدی استفاده می‌شود، به ویژه در سازمان‌هایی که سادگی را بر پیچیدگی عملیاتی ابزارهای CDC ترجیح می‌دهند:

- **استارتاپ‌های مرحله اولیه**: تیم‌هایی با تخصص محدود زیرساختی اغلب با Polling Publisher شروع می‌کنند زیرا هیچ ابزار اضافی فراتر از پایگاه داده برنامه و واسطه پیام نیاز ندارد.

- **مهاجرت از Monolith به Microservices**: هنگام استخراج سرویس‌ها از یک monolith، Polling Publisher راهی کم‌ریسک برای شروع انتشار رویدادهای دامنه بدون معرفی زیرساخت CDC فراهم می‌کند.

- **برنامه‌های سازمانی با سیاست‌های سخت‌گیرانه پایگاه داده**: در سازمان‌هایی که دسترسی به لاگ‌های تراکنش پایگاه داده توسط DBAها محدود شده، Polling Publisher صرفاً از طریق پرس‌وجوهای استاندارد SQL کار می‌کند.

- **فریم‌ورک‌ها و کتابخانه‌ها**:
  - **Axon Framework** (جاوا) از Polling Publisher به عنوان یکی از استراتژی‌های انتشار رویداد پشتیبانی می‌کند.
  - **MassTransit** (.NET) پشتیبانی outbox با مکانیزم polling داخلی ارائه می‌دهد.
  - **Laravel** (PHP) توسعه‌دهندگان معمولاً انتشار outbox مبتنی بر polling را با استفاده از دستورات زمان‌بندی شده یا queue workers پیاده‌سازی می‌کنند.

- **مسیر پیشرفت رایج**: بسیاری از تیم‌ها با Polling Publisher برای سادگی شروع می‌کنند و بعداً به [Transaction Log Tailing](./transaction-log-tailing.md) (مثلاً Debezium) مهاجرت می‌کنند زمانی که به تأخیر کمتر یا توان بالاتر نیاز دارند.

### مقیاس‌دهی Polling Publisher

برای سیستم‌های با توان بالا، یک poller واحد ممکن است گلوگاه شود. استراتژی‌های رایج عبارتند از:

```
Strategy 1: Partitioned Polling (recommended)
+-------------------------------------------+
| Poller 1: WHERE aggregate_id % 3 = 0      |
| Poller 2: WHERE aggregate_id % 3 = 1      |
| Poller 3: WHERE aggregate_id % 3 = 2      |
+-------------------------------------------+
Each poller handles a subset of aggregates.
No coordination needed. Ordering preserved per aggregate.

Strategy 2: Leader Election
+-------------------------------------------+
| Only the leader poller runs at any time.   |
| If leader fails, another takes over.       |
| Uses distributed lock (Redis, ZooKeeper).  |
+-------------------------------------------+
Simple but limits throughput to a single poller.

Strategy 3: Database Row Locking
+-------------------------------------------+
| SELECT ... FOR UPDATE SKIP LOCKED          |
| Multiple pollers compete for rows.         |
| Database handles coordination.             |
+-------------------------------------------+
Good for moderate scale but adds DB lock contention.
```

## ۹. خلاصه

**Polling Publisher** الگویی ساده و عملی برای تحویل رویدادها از جدول outbox به واسطه پیام است. این الگو به عنوان مکانیزم تحویل برای الگوی [Transactional Outbox](./transactional-outbox.md) عمل می‌کند.

**نکات کلیدی:**

1. یک فرآیند پس‌زمینه به صورت دوره‌ای جدول outbox را برای رویدادهای منتشر نشده poll می‌کند
2. رویدادها به واسطه پیام منتشر می‌شوند و در پایگاه داده به عنوان منتشر شده علامت‌گذاری می‌شوند
3. این الگو **مستقل از پایگاه داده** است و **هیچ زیرساخت خاصی** نیاز ندارد
4. **تأخیر بالاتر** را در ازای **سادگی** نسبت به [Transaction Log Tailing](./transaction-log-tailing.md) معامله می‌کند
5. مصرف‌کنندگان باید **ایدمپوتنت** باشند زیرا تحویل حداقل یک‌بار ممکن است تکراری تولید کند
6. فاصله poll اهرم اصلی تنظیم است: فاصله کمتر یعنی تأخیر کمتر اما بار پایگاه داده بیشتر
7. رویدادهای منتشر شده باید به صورت دوره‌ای پاکسازی شوند تا از رشد بی‌رویه جدول جلوگیری شود
8. برای مقیاس‌دهی، از polling پارتیشن‌بندی شده یا استراتژی‌های قفل ردیف پایگاه داده استفاده کنید

Polling Publisher نقطه شروع عالی برای تیم‌هایی است که الگوی [Transactional Outbox](./transactional-outbox.md) را اتخاذ می‌کنند، و زمانی که تأخیر کمتر یا توان بالاتر لازم است می‌تواند با [Transaction Log Tailing](./transaction-log-tailing.md) جایگزین شود.

---

**همچنین ببینید:** [Transactional Outbox](./transactional-outbox.md) | [Transaction Log Tailing](./transaction-log-tailing.md) | [معماری رویداد-محور](../event-driven/event-driven-architecture.md) | [مقایسه RabbitMQ و Kafka](./rabbitmq-vs-kafka.md)
