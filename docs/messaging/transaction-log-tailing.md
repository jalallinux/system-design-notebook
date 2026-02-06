# Transaction Log Tailing

## 1. Introduction

**Transaction Log Tailing**, also known as **Change Data Capture (CDC)**, is a technique that reads the database's internal transaction log to detect committed changes and publish them as events to a message broker. Rather than modifying application code to emit events, CDC piggybacks on the mechanism every relational database already uses for durability and replication: the **write-ahead log** (WAL in PostgreSQL) or the **binary log** (binlog in MySQL).

This pattern is most commonly used in conjunction with the **Transactional Outbox** pattern. When a service writes business data and an outbox row within the same local transaction, a CDC connector detects the new outbox row by reading the transaction log and forwards it to a message broker such as [Apache Kafka](./rabbitmq-vs-kafka.md).

```
+---------------------+       +-------------------+       +-----------------+
|    Application      |       | Transaction Log   |       | Message Broker  |
|                     |       | Tailer (CDC)      |       | (e.g. Kafka)    |
| +---------+------+  |       |                   |       |                 |
| | Business | Out- |  | WAL  |  Reads committed  | Pub   |  Downstream     |
| | Tables   | box  |--+----->|  rows from log    |------>|  Consumers      |
| |          | Table|  |      |                   |       |                 |
| +---------+------+  |       +-------------------+       +-----------------+
+---------------------+
```

Transaction Log Tailing provides a **non-invasive**, **low-latency**, and **reliable** way to capture every committed change without polling the database or modifying application queries.

---

## 2. Context and Problem

In a [microservices architecture](../architecture/microservices.md), services frequently need to update their own database **and** notify other services that something changed. A common approach is the Transactional Outbox pattern, where the service writes an event record to an `outbox` table as part of the same database transaction that modifies business data. This guarantees atomicity between the state change and the event.

However, a critical question remains: **how do you get the events out of the outbox table and into the message broker?**

Two primary strategies exist:

1. **Polling Publisher** -- a background process periodically queries the outbox table for unpublished rows, publishes them, and marks them as sent.
2. **Transaction Log Tailing (CDC)** -- a connector reads the database's transaction log to detect newly committed outbox rows and publishes them to the broker.

### Problems with polling

| Problem | Description |
|---------|-------------|
| **Latency** | Events are delayed by the polling interval (e.g. 1--5 seconds) |
| **Database load** | Frequent `SELECT` queries hit the database even when there are no new events |
| **Scalability** | Multiple pollers require coordination to avoid duplicate publishing |
| **Missed events** | Incorrect polling logic or race conditions can skip rows |
| **Cleanup overhead** | Polled rows must be deleted or flagged, adding write load |

Transaction Log Tailing addresses all of these problems by reading the database's own internal record of committed transactions.

---

## 3. Forces

The following forces drive the adoption of Transaction Log Tailing over alternative approaches:

- **Guaranteed delivery** -- Every committed transaction appears in the log; nothing is skipped.
- **Low latency** -- Changes are captured within milliseconds of being committed, far faster than polling intervals.
- **Minimal database overhead** -- Reading the log is a lightweight operation that does not execute SQL queries against business tables.
- **No application code changes** -- The CDC connector works at the database level; the application does not need to emit events explicitly.
- **Exactly-once semantics (with care)** -- Combined with idempotent consumers or deduplication, the pattern can achieve effectively exactly-once processing.
- **Ordering guarantees** -- The transaction log preserves the exact commit order, ensuring events are published in the order they were written.
- **Operational complexity** -- CDC infrastructure (connectors, Kafka Connect, schema registry) must be deployed, monitored, and maintained.
- **Database coupling** -- The implementation is tied to a specific database engine's log format.

---

## 4. Solution

### How Database Transaction Logs Work

Every major relational database maintains a sequential log of all committed changes for durability and replication.

#### PostgreSQL: Write-Ahead Log (WAL)

PostgreSQL writes every change to the WAL **before** applying it to the data files. This ensures that even if the server crashes, committed transactions can be recovered by replaying the WAL. PostgreSQL exposes its WAL through **logical replication slots**, which CDC tools use to stream changes.

```
  Client Transaction
        |
        v
  +------------------+
  | Write to WAL     |  <-- Durable, sequential log
  +------------------+
        |
        v
  +------------------+
  | Apply to Tables  |  <-- Actual data pages updated
  +------------------+
        |
        v
  +------------------+
  | Logical Decoding |  <-- CDC reads changes from here
  +------------------+
```

#### MySQL: Binary Log (binlog)

MySQL's binlog records all data-modifying statements (in statement-based mode) or actual row changes (in row-based mode). CDC tools connect as a MySQL replication replica and read the binlog stream in real time.

```
  Client Transaction
        |
        v
  +------------------+
  | Write to binlog  |  <-- Row-based changes logged
  +------------------+
        |
        v
  +------------------+
  | Apply to InnoDB  |  <-- Storage engine updated
  +------------------+
        |
        v
  +------------------+
  | CDC Connector    |  <-- Reads binlog as a replica
  +------------------+
```

### CDC Connector Architecture

The CDC connector sits between the database and the message broker:

```
+------------------+     +---------------------+     +------------------+
|   Source DB      |     |   CDC Connector     |     |  Message Broker  |
|                  |     |   (e.g. Debezium)   |     |  (e.g. Kafka)   |
|  +-----------+   |     |                     |     |                  |
|  | outbox    |   | log |  1. Connect as      |     |  +-----------+  |
|  | table     |---+---->|     replica         |     |  | outbox    |  |
|  +-----------+   |     |  2. Read log stream |---->|  | topic     |  |
|  | orders    |   |     |  3. Filter tables   |     |  +-----------+  |
|  | customers |   |     |  4. Transform       |     |  +-----------+  |
|  +-----------+   |     |  5. Publish event   |     |  | orders    |  |
|                  |     |                     |     |  | topic     |  |
+------------------+     +---------------------+     +------------------+
```

### Debezium: The Industry Standard

[Debezium](https://debezium.io/) is the most widely adopted open-source CDC platform. It runs as a set of **Kafka Connect** source connectors, one per supported database.

#### Debezium Architecture

```
+------------+     +-------------------+     +------------------+     +----------------+
| PostgreSQL |     | Kafka Connect     |     | Apache Kafka     |     | Consumers      |
|            | WAL | +-----------+     |     |                  |     |                |
| outbox     |---->| | Debezium  |     | pub | +------------+  |     | Order Service  |
| table      |     | | Postgres  |-----+---->| | outbox     |--+---->| Notification   |
|            |     | | Connector |     |     | | events     |  |     | Analytics      |
+------------+     | +-----------+     |     | +------------+  |     +----------------+
                   +-------------------+     +------------------+

+------------+     +-------------------+            |
| MySQL      |     | Kafka Connect     |            |
|            | bin | +-----------+     |            |
| outbox     |---->| | Debezium  |     |     +------v---------+
| table      |     | | MySQL     |-----+---->| outbox events  |
|            |     | | Connector |     |     | topic          |
+------------+     | +-----------+     |     +----------------+
                   +-------------------+
```

**How Debezium works:**

1. **Connects to the database** as a replication client (logical replication in PostgreSQL, binlog reader in MySQL).
2. **Takes an initial snapshot** of the existing data (optional, configurable).
3. **Streams changes** in real time as they are committed to the transaction log.
4. **Transforms changes** into structured events (JSON or Avro) containing before/after row states.
5. **Publishes events** to Kafka topics (one topic per table by default).
6. **Tracks offsets** in Kafka Connect so it can resume from where it left off after a restart.

#### Debezium Event Structure

A typical Debezium change event contains:

```json
{
  "schema": { ... },
  "payload": {
    "before": null,
    "after": {
      "id": "evt-001",
      "aggregate_type": "Order",
      "aggregate_id": "ord-123",
      "event_type": "OrderCreated",
      "payload": "{\"orderId\":\"ord-123\",\"amount\":99.99}",
      "created_at": 1706000000000
    },
    "source": {
      "version": "2.5.0",
      "connector": "mysql",
      "name": "inventory",
      "ts_ms": 1706000000000,
      "db": "ecommerce",
      "table": "outbox"
    },
    "op": "c",
    "ts_ms": 1706000000123
  }
}
```

| Field | Description |
|-------|-------------|
| `before` | Row state before the change (`null` for inserts) |
| `after` | Row state after the change |
| `source` | Metadata about the database, table, connector version |
| `op` | Operation type: `c` = create, `u` = update, `d` = delete, `r` = read (snapshot) |
| `ts_ms` | Timestamp when Debezium processed the event |

### Other CDC Tools

| Tool | Database Support | Destination | Notes |
|------|-----------------|-------------|-------|
| **Debezium** | PostgreSQL, MySQL, MongoDB, SQL Server, Oracle, Cassandra | Kafka (via Kafka Connect) | Most popular, open-source, Red Hat backed |
| **Maxwell's Daemon** | MySQL only | Kafka, Kinesis, RabbitMQ, Redis | Lightweight, MySQL-specific |
| **AWS DMS** | Most relational DBs | Kinesis, S3, Redshift, Kafka | Managed service, AWS ecosystem |
| **LinkedIn Databus** | Oracle, MySQL | Custom consumers | Pioneer of CDC at scale, inspired Debezium |
| **Striim** | Wide range | Kafka, cloud storage, databases | Commercial, real-time integration |
| **Airbyte** | Wide range | Various destinations | Open-source ELT with CDC modes |
| **Google Datastream** | MySQL, PostgreSQL, Oracle | BigQuery, Cloud Storage | Managed, GCP ecosystem |

### Outbox Event Router (Debezium)

Debezium provides a built-in **Outbox Event Router** Single Message Transform (SMT) that simplifies the Transactional Outbox pattern. Instead of publishing raw database change events, the router extracts the outbox table columns and routes them to appropriate Kafka topics based on the `aggregate_type` column.

```
Outbox Table Row:
+----------+----------------+--------------+---------------+------------------+
| id       | aggregate_type | aggregate_id | event_type    | payload          |
+----------+----------------+--------------+---------------+------------------+
| evt-001  | Order          | ord-123      | OrderCreated  | {"amount":99.99} |
+----------+----------------+--------------+---------------+------------------+

                        |
                        v  (Outbox Event Router SMT)

Kafka Topic: "outbox.event.Order"
Key: "ord-123"
Value: {"amount":99.99}
Headers: { "id": "evt-001", "eventType": "OrderCreated" }
```

---

## 5. Example

### End-to-End: MySQL binlog with Debezium and Kafka

Consider an e-commerce system where the Order Service creates orders and needs to notify downstream services.

#### Step 1: Application writes to business table and outbox atomically

```sql
BEGIN;

INSERT INTO orders (id, customer_id, total, status)
VALUES ('ord-123', 'cust-456', 99.99, 'CREATED');

INSERT INTO outbox (id, aggregate_type, aggregate_id, event_type, payload, created_at)
VALUES (
    'evt-001',
    'Order',
    'ord-123',
    'OrderCreated',
    '{"orderId":"ord-123","customerId":"cust-456","total":99.99}',
    NOW()
);

COMMIT;
```

Both rows are written in the same transaction. If the transaction fails, neither row is written.

#### Step 2: MySQL writes to binlog

MySQL records the committed changes in the binlog:

```
# at position 12345
#260206 10:30:00 server id 1  end_log_pos 12500
### INSERT INTO ecommerce.orders
### SET
###   @1='ord-123'
###   @2='cust-456'
###   @3=99.99
###   @4='CREATED'

### INSERT INTO ecommerce.outbox
### SET
###   @1='evt-001'
###   @2='Order'
###   @3='ord-123'
###   @4='OrderCreated'
###   @5='{"orderId":"ord-123","customerId":"cust-456","total":99.99}'
###   @6='2026-02-06 10:30:00'
```

#### Step 3: Debezium reads the binlog and publishes to Kafka

The Debezium MySQL connector, configured to monitor the `outbox` table, detects the new row and publishes an event to the Kafka topic `outbox.event.Order`.

**Debezium Connector Configuration (JSON):**

```json
{
  "name": "ecommerce-outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql-primary",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "********",
    "database.server.id": "184054",
    "topic.prefix": "ecommerce",
    "database.include.list": "ecommerce",
    "table.include.list": "ecommerce.outbox",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.id": "id",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.table.field.event.type": "event_type",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.route.by.field": "aggregate_type",
    "transforms.outbox.route.topic.replacement": "outbox.event.${routedByValue}"
  }
}
```

#### Step 4: Downstream services consume events

```
+------------------+     +-----------+     +------------------+     +-------------------+
| Order Service    |     | MySQL     |     | Debezium +       |     | Kafka Topic:      |
|                  |     | binlog    |     | Kafka Connect    |     | outbox.event.Order|
| BEGIN;           |     |           |     |                  |     |                   |
|  INSERT orders   |---->| log entry |---->| read binlog      |---->| {"orderId":...}   |
|  INSERT outbox   |     |           |     | transform + pub  |     |                   |
| COMMIT;          |     |           |     |                  |     +--------+----------+
+------------------+     +-----------+     +------------------+              |
                                                                 +-----------+-----------+
                                                                 |           |           |
                                                           +-----v--+  +----v---+  +----v------+
                                                           |Notific-|  |Payment |  |Analytics  |
                                                           |ation   |  |Service |  |Service    |
                                                           |Service |  |        |  |           |
                                                           +--------+  +--------+  +-----------+
```

#### Step 5: Outbox table cleanup

After Debezium has read and published the events, the outbox table rows can be periodically deleted to prevent unbounded growth:

```sql
-- Cron job or scheduled task
DELETE FROM outbox WHERE created_at < NOW() - INTERVAL 7 DAY;
```

Since Debezium reads the log (not the table itself), deleting old rows does not affect event delivery.

---

## 6. Benefits and Drawbacks

### Benefits

| Benefit | Description |
|---------|-------------|
| **Low latency** | Events are captured within milliseconds of transaction commit, compared to seconds with polling |
| **No polling overhead** | Eliminates repeated `SELECT` queries against the outbox table |
| **Captures all changes** | The transaction log is the source of truth; no committed change is missed |
| **No application code changes** | CDC works at the database level; the application only writes to tables |
| **Preserves ordering** | Events are published in the exact order they were committed |
| **Supports multiple tables** | A single connector can capture changes from multiple tables simultaneously |
| **Proven at scale** | Used by LinkedIn, Uber, Airbnb, and most Kafka-based architectures |
| **Decoupled from application** | The CDC connector runs as a separate process, independent of the application lifecycle |

### Drawbacks

| Drawback | Description |
|----------|-------------|
| **Database-specific** | Each database engine requires a different connector (PostgreSQL, MySQL, MongoDB, etc.) |
| **Operational complexity** | Requires deploying and managing Kafka Connect, connectors, and potentially a schema registry |
| **Requires DB configuration** | The database must be configured for logical replication (PostgreSQL) or row-based binlog (MySQL) |
| **Log retention** | If the CDC connector falls behind, the database may purge log entries before they are consumed |
| **Schema evolution** | Changes to table schemas must be handled carefully to avoid breaking downstream consumers |
| **Debugging difficulty** | Tracing an issue from the application through the log, connector, and broker adds complexity |
| **Infrastructure cost** | Kafka Connect cluster, connectors, and monitoring add to infrastructure requirements |
| **Snapshot overhead** | Initial snapshots of large tables can be slow and resource-intensive |

### Transaction Log Tailing vs Polling Publisher

| Aspect | Transaction Log Tailing (CDC) | Polling Publisher |
|--------|-------------------------------|-------------------|
| **Latency** | Milliseconds (near real-time) | Seconds (polling interval) |
| **Database load** | Minimal (reads log, not tables) | Moderate (repeated SELECT queries) |
| **Reliability** | Very high (log is source of truth) | Good (but race conditions possible) |
| **Implementation** | Infrastructure-heavy (Kafka Connect, Debezium) | Simple (application code + scheduler) |
| **Ordering** | Guaranteed (log order) | Approximate (query order) |
| **Scalability** | Excellent (single reader, horizontal consumers) | Requires coordination for multiple pollers |
| **DB dependency** | Tight (DB-specific log format) | Loose (standard SQL) |
| **Operational cost** | Higher (CDC infrastructure) | Lower (just application code) |
| **Best for** | High-throughput, low-latency systems | Simple systems, small teams, prototypes |

---

## 7. Related Patterns

- **Transactional Outbox** -- Transaction Log Tailing is the recommended mechanism for reading events from the outbox table. The outbox pattern guarantees atomicity between the business write and the event; CDC handles reliable delivery to the broker.

- **Polling Publisher** -- The simpler alternative to Transaction Log Tailing. Instead of reading the transaction log, a background process polls the outbox table at regular intervals. Easier to implement but introduces latency, database load, and potential race conditions.

- **[Event-Driven Architecture](../event-driven/event-driven-architecture.md)** -- Transaction Log Tailing enables event-driven communication between services by reliably publishing domain events to a message broker.

- **[CQRS](../data-patterns/cqrs.md)** -- CDC can be used to keep read models in sync with the write model. Changes committed to the write database are captured via the transaction log and used to update read-side projections.

- **[Saga Pattern](../distributed-transactions/saga.md)** -- Sagas coordinate distributed transactions through events. Transaction Log Tailing ensures these events are reliably published from each service's database.

- **Event Sourcing** -- While Event Sourcing stores events as the primary source of truth, CDC captures state changes from traditional databases. Both produce event streams, but their sources differ.

- **[RabbitMQ vs Kafka](./rabbitmq-vs-kafka.md)** -- Kafka is the most common target for CDC pipelines due to its log-based architecture and replay capabilities. RabbitMQ can also be a target (e.g., with Maxwell's Daemon) but lacks native replay.

- **[CAP Theorem](../fundamentals/cap-theorem.md)** -- CDC-based systems are eventually consistent. Understanding CAP trade-offs is essential when designing systems where the read side lags behind the write side.

---

## 8. Real-World Usage

### LinkedIn

LinkedIn pioneered CDC at scale with **Databus**, an open-source change data capture system. Databus was built to replicate changes from Oracle and MySQL databases to downstream search indexes, caches, and read replicas. It served as the inspiration for many modern CDC tools including Debezium.

### Uber

Uber uses CDC extensively for its data platform. Changes from operational MySQL databases are captured via the binlog and streamed to Apache Kafka, feeding real-time analytics, search indexes (Elasticsearch), and data warehouses (Apache Hive/Spark).

### Airbnb

Airbnb adopted Debezium-based CDC to synchronize data between their primary databases and downstream services. This enabled real-time search index updates when listing details changed, replacing batch ETL jobs that introduced hours of delay.

### Shopify

Shopify uses CDC to power their real-time data pipeline. Changes to merchant and order data are captured from MySQL and streamed to Kafka, enabling real-time dashboards, fraud detection, and inventory synchronization.

### Zalando

Zalando built their event-driven architecture around CDC. They use a custom solution called **Nakadi** (event broker) combined with PostgreSQL logical replication to publish domain events, enabling loose coupling between hundreds of microservices.

### Common CDC Architecture in Production

```
+-------------------+     +-------------------+     +-------------------+
|  Service A        |     |  Service B        |     |  Service C        |
|  (PostgreSQL)     |     |  (MySQL)          |     |  (MongoDB)        |
+--------+----------+     +--------+----------+     +--------+----------+
         |                         |                         |
    WAL  |                  binlog |                 oplog   |
         v                         v                         v
+--------+----------+     +--------+----------+     +--------+----------+
| Debezium          |     | Debezium          |     | Debezium          |
| PG Connector      |     | MySQL Connector   |     | MongoDB Connector |
+--------+----------+     +--------+----------+     +--------+----------+
         |                         |                         |
         +------------+------------+------------+------------+
                      |                         |
                      v                         v
              +-------+--------+        +-------+--------+
              | Apache Kafka   |        | Schema Registry|
              | (Event Hub)    |        | (Avro/JSON)    |
              +-------+--------+        +----------------+
                      |
         +------------+------------+------------+
         |            |            |            |
         v            v            v            v
   +-----------+ +---------+ +---------+ +-----------+
   | Search    | | Analyt- | | Notif-  | | Data      |
   | Index     | | ics     | | ication | | Warehouse |
   | (Elastic) | | (Flink) | | Service | | (Redshift)|
   +-----------+ +---------+ +---------+ +-----------+
```

### Configuration and Operational Considerations

| Consideration | Recommendation |
|---------------|----------------|
| **Log retention** | Set database log retention high enough for the CDC connector to recover from downtime (e.g., 3--7 days for PostgreSQL WAL, binlog expiration in MySQL) |
| **Connector monitoring** | Monitor connector lag, throughput, and error rates via JMX or Prometheus metrics |
| **Schema changes** | Use a schema registry (Confluent Schema Registry) to handle schema evolution gracefully |
| **Snapshotting** | For initial setup, configure snapshot mode (`initial`, `schema_only`, `never`) based on data volume |
| **Idempotent consumers** | Design consumers to handle duplicate events, as at-least-once delivery is the default guarantee |
| **Outbox cleanup** | Schedule periodic deletion of old outbox rows to prevent table bloat |
| **High availability** | Run Kafka Connect in distributed mode with multiple workers for fault tolerance |
| **Security** | Use SSL/TLS for database connections and Kafka communication; restrict CDC user privileges to read-only on the log |

---

## 9. Summary

Transaction Log Tailing (CDC) is the **production-grade approach** for reliably publishing database changes as events. It eliminates the latency and overhead of polling, guarantees that no committed change is missed, and preserves the exact ordering of transactions.

**Key takeaways:**

1. **CDC reads the database's own transaction log** (WAL, binlog, oplog) to detect changes -- it does not poll application tables.
2. **Debezium** is the de facto standard CDC platform, running as Kafka Connect source connectors.
3. **Combined with the Transactional Outbox pattern**, CDC provides a robust mechanism for eventually consistent event-driven communication between [microservices](../architecture/microservices.md).
4. **Trade-off**: operational complexity of CDC infrastructure vs. simplicity of polling. For small teams or prototypes, polling may be sufficient. For production systems at scale, CDC is the recommended approach.
5. **Real-world adoption** by LinkedIn, Uber, Airbnb, Shopify, and most organizations using Kafka-based architectures validates the pattern's reliability.

```
Choose your approach:

+---------------------------+     +---------------------------+
|   Simple / Small Scale    |     |   Production / At Scale   |
|                           |     |                           |
|   Polling Publisher       |     |   Transaction Log Tailing |
|                           |     |                           |
|   - Cron / scheduler      |     |   - Debezium / CDC        |
|   - SELECT from outbox    |     |   - Read WAL / binlog     |
|   - Mark as published     |     |   - Publish to Kafka      |
|   - Simple to implement   |     |   - Low latency           |
|   - Higher latency        |     |   - No DB load            |
|   - DB load from polling  |     |   - Operationally complex |
+---------------------------+     +---------------------------+
```

---

**See also:**
- [Event-Driven Architecture](../event-driven/event-driven-architecture.md)
- [CQRS](../data-patterns/cqrs.md)
- [Saga Pattern](../distributed-transactions/saga.md)
- [RabbitMQ vs Kafka](./rabbitmq-vs-kafka.md)
- [CAP Theorem](../fundamentals/cap-theorem.md)
- [Microservices Architecture](../architecture/microservices.md)
