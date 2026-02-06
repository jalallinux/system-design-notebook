# Domain-Specific Protocol

## 1. Introduction

In distributed systems, communication between services is typically handled by general-purpose protocols such as HTTP/REST and gRPC for synchronous calls, or message brokers like RabbitMQ and Kafka for asynchronous messaging. However, certain communication scenarios have unique requirements that these general-purpose protocols cannot serve efficiently.

**Domain-specific protocols** are specialized communication protocols designed and optimized for particular use cases. They encode domain knowledge directly into the protocol layer, providing superior performance, semantics, and developer experience for their target scenarios. Examples include SMTP for email, MQTT for IoT devices, WebSocket for real-time bidirectional communication, and FIX for financial trading.

This document explores when and why you should reach beyond general-purpose protocols, the major domain-specific protocols available, and how to choose the right one for your system.

## 2. Context and Problem

### Context

You are designing a distributed system where services need to communicate. You have already considered general-purpose synchronous protocols like [RPI](./rpi.md) (REST, gRPC) and asynchronous protocols like [Messaging](./messaging.md) (RabbitMQ, Kafka). However, some communication needs in your system have specialized requirements.

### Problem

General-purpose protocols like REST/HTTP or message queuing may be **inefficient or insufficient** for certain communication patterns:

- **REST/HTTP** is request-response only. It cannot push data from server to client without polling, making it unsuitable for real-time dashboards or live notifications.
- **gRPC** supports streaming but still operates over HTTP/2, which may be too heavy for constrained IoT devices with limited bandwidth and battery.
- **Traditional message brokers** add latency and infrastructure overhead that may be unacceptable for peer-to-peer media streaming or sub-millisecond financial trading.

```
Problem Illustration:

    REST/HTTP (Request-Response Only)
    +---------+        +---------+
    | Client  | -----> | Server  |    Client must poll repeatedly
    |         | <----- |         |    for updates (inefficient)
    +---------+        +---------+

    IoT Sensors (Thousands of constrained devices)
    +-------+  +-------+  +-------+
    |Sensor1|  |Sensor2|  |Sensor3|  ... x10,000
    +-------+  +-------+  +-------+
        |          |          |
        v          v          v
    HTTP overhead per message = UNACCEPTABLE
    (headers, TLS handshake, connection setup)
```

## 3. Forces

Several forces push toward adopting domain-specific protocols:

| Force | Description |
|-------|-------------|
| **Real-time requirements** | Some use cases demand server-initiated push or bidirectional streaming with minimal latency |
| **Resource constraints** | IoT devices, mobile clients, or embedded systems have limited CPU, memory, and bandwidth |
| **Connection patterns** | Peer-to-peer communication, pub-sub fan-out, or persistent connections do not map well to request-response |
| **Data volume** | High-frequency data streams (sensor telemetry, market data) overwhelm HTTP's per-request overhead |
| **Protocol semantics** | Domains like email, financial trading, or media streaming have specific semantics that benefit from protocol-level support |
| **Reliability guarantees** | Some domains need specific QoS levels (e.g., at-most-once for telemetry vs. exactly-once for transactions) |
| **Standardization** | Industry-wide protocols ensure interoperability between vendors (e.g., SMTP for email, FIX for trading) |

## 4. Solution

Use **domain-specific protocols** tailored to the communication needs of your system. Rather than forcing all communication through a single general-purpose protocol, select specialized protocols for scenarios where they provide clear advantages.

### Protocol Comparison Matrix

```
+------------------+-------------+-----------+------------+-----------+---------------+
| Protocol         | Direction   | Transport | Latency    | Overhead  | Best For      |
+------------------+-------------+-----------+------------+-----------+---------------+
| HTTP/REST        | Req-Resp    | TCP       | Medium     | High      | CRUD APIs     |
| gRPC             | Bi-dir      | HTTP/2    | Low-Med    | Medium    | Service-to-   |
|                  | streaming   |           |            |           | service RPC   |
| WebSocket        | Bi-dir      | TCP       | Low        | Low       | Real-time     |
|                  | full-duplex |           |            |           | apps          |
| SSE              | Server-push | HTTP/1.1  | Low        | Low       | Live feeds    |
| MQTT             | Pub-Sub     | TCP       | Low        | Very Low  | IoT devices   |
| AMQP             | Pub-Sub /   | TCP       | Low-Med    | Medium    | Enterprise    |
|                  | Queuing     |           |            |           | messaging     |
| WebRTC           | Peer-to-    | UDP/SRTP  | Very Low   | Medium    | Media         |
|                  | Peer        |           |            |           | streaming     |
| SMTP             | Store-and-  | TCP       | High       | Medium    | Email         |
|                  | Forward     |           |            |           | delivery      |
| FIX              | Req-Resp /  | TCP       | Very Low   | Low       | Financial     |
|                  | Streaming   |           |            |           | trading       |
| GraphQL Subs     | Server-push | WebSocket | Low        | Medium    | Real-time     |
|                  |             |           |            |           | queries       |
+------------------+-------------+-----------+------------+-----------+---------------+
```

### 4.1 WebSocket

WebSocket provides **full-duplex bidirectional communication** over a single persistent TCP connection. After an initial HTTP handshake, the connection is upgraded and both client and server can send messages independently at any time.

```
WebSocket vs REST Comparison:

REST (Polling for updates):
Client  ----GET /updates----->  Server     (no data)
Client  ----GET /updates----->  Server     (no data)
Client  ----GET /updates----->  Server     (data!)
Client  ----GET /updates----->  Server     (no data)
        4 requests, 1 useful response

WebSocket (Persistent connection):
Client  ====UPGRADE==========>  Server
Client  <==== message ========  Server     (data pushed instantly)
Client  ===== message =======>  Server     (client can send too)
Client  <==== message ========  Server     (data pushed instantly)
        1 connection, 0 wasted requests
```

**When to use:** Chat applications, live dashboards, collaborative editing, gaming, real-time notifications.

### 4.2 Server-Sent Events (SSE)

SSE provides a **unidirectional server-to-client** push channel over standard HTTP. Unlike WebSocket, it uses a regular HTTP connection and is natively supported by browsers via the `EventSource` API.

**When to use:** Live feeds, stock tickers, progress updates, notification streams where only server-to-client push is needed.

**SSE vs WebSocket:** Choose SSE when you only need server push (simpler setup, automatic reconnection, works through HTTP proxies). Choose WebSocket when you need bidirectional communication.

### 4.3 MQTT (Message Queuing Telemetry Transport)

MQTT is an extremely **lightweight publish-subscribe** protocol designed for constrained devices and unreliable networks. It uses minimal bandwidth and supports three Quality of Service (QoS) levels.

```
MQTT Architecture:

  +----------+         +-------------+         +----------+
  | Sensor A | --pub-> |             | --sub-> | Dashboard|
  +----------+   topic |   MQTT      |  topic  +----------+
  +----------+  /temp  |   Broker    | /temp   +----------+
  | Sensor B | --pub-> |             | --sub-> | Alert    |
  +----------+         +-------------+         | Service  |
                            |                  +----------+
                       QoS Levels:
                       0 = At most once  (fire and forget)
                       1 = At least once (acknowledged)
                       2 = Exactly once  (4-part handshake)
```

**Packet size comparison:**
- HTTP request minimum: ~300-700 bytes (headers alone)
- MQTT PUBLISH minimum: **2 bytes** (fixed header only)

**When to use:** IoT sensor networks, home automation, vehicle telemetry, mobile apps with unreliable connectivity.

### 4.4 AMQP (Advanced Message Queuing Protocol)

AMQP is a **wire-level protocol** for enterprise message-oriented middleware. Unlike MQTT's simplicity, AMQP provides rich features including exchanges, bindings, routing, transactions, and security. RabbitMQ is the most popular AMQP implementation.

**When to use:** Enterprise integration, complex message routing, systems requiring message-level transactions and acknowledgments.

### 4.5 GraphQL Subscriptions

GraphQL Subscriptions extend the GraphQL query language with **real-time capabilities**. Clients subscribe to specific data changes using the same query syntax they use for regular reads, and receive updates over a WebSocket connection.

**When to use:** Applications already using GraphQL that need real-time updates for specific query results.

### 4.6 WebRTC (Web Real-Time Communication)

WebRTC enables **peer-to-peer** audio, video, and data communication directly between browsers or devices, bypassing the server for media transfer.

```
WebRTC Connection Establishment:

  +--------+                              +--------+
  | Peer A |                              | Peer B |
  +--------+                              +--------+
      |     1. Offer (SDP) via Signaling      |
      | -------- Server ------------------>   |
      |     2. Answer (SDP) via Signaling     |
      | <------- Server -------------------   |
      |     3. ICE Candidates exchanged       |
      | <--------- STUN/TURN ------------>    |
      |                                       |
      |     4. Direct P2P Media Stream        |
      | <================================>    |
      |     (UDP/SRTP - no server needed)     |
```

**When to use:** Video conferencing, voice calls, peer-to-peer file sharing, screen sharing, low-latency gaming.

### 4.7 SMTP (Simple Mail Transfer Protocol)

SMTP is the internet standard for **email transmission**. It uses a store-and-forward model where mail servers relay messages hop by hop until they reach the destination mail server.

**When to use:** Email delivery (there is no practical alternative for internet email).

### 4.8 FIX (Financial Information eXchange)

FIX is the de facto messaging standard for the **financial trading** industry. It provides pre-defined message types for orders, executions, market data, and trade allocations with strict sequencing and guaranteed delivery.

**When to use:** Electronic trading platforms, order management systems, market data distribution in financial services.

### Protocol Selection Criteria

When choosing a protocol, evaluate these dimensions:

```
Decision Matrix:

                        Low Latency
                            |
                  WebRTC    |    FIX
                  WebSocket |    MQTT
                            |
   Bidirectional -----------+----------- Unidirectional
                            |
                  AMQP      |    SSE
                  gRPC      |    SMTP
                            |
                       High Latency


   Step 1: What is the communication direction?
           Bidirectional  --> WebSocket, WebRTC, gRPC
           Server-push    --> SSE, MQTT, GraphQL Subs
           Request-Reply  --> REST, gRPC

   Step 2: What are the client constraints?
           Constrained IoT --> MQTT
           Browser         --> WebSocket, SSE, WebRTC
           Server-to-Server --> gRPC, AMQP

   Step 3: What latency is acceptable?
           Sub-millisecond --> FIX, custom TCP
           Low (1-50ms)    --> WebSocket, MQTT, WebRTC
           Medium (50-500ms) --> gRPC, AMQP
           High (seconds+) --> REST, SMTP

   Step 4: What reliability guarantee is needed?
           At-most-once    --> MQTT QoS 0, UDP
           At-least-once   --> MQTT QoS 1, AMQP
           Exactly-once    --> MQTT QoS 2, FIX
```

## 5. Example

### IoT Smart Factory: Multi-Protocol Architecture

Consider a smart factory system that monitors equipment, provides a real-time dashboard for operators, and sends alerts.

```
Architecture Overview:

  +----------+  +----------+  +----------+
  | Temp     |  | Pressure |  | Vibration|    (Thousands of sensors)
  | Sensor   |  | Sensor   |  | Sensor   |
  +----+-----+  +----+-----+  +----+-----+
       |              |              |
       |   MQTT (QoS 1, lightweight)|
       v              v              v
  +-----------------------------------+
  |           MQTT Broker             |
  |         (e.g., Mosquitto)         |
  +--------+----------+--------------+
           |          |
           v          v
  +--------+--+  +----+---------+         +-----------------+
  | Data       |  | Alert       |         | Operator        |
  | Ingestion  |  | Engine      |         | Dashboard       |
  | Service    |  | (threshold  |         | (Browser)       |
  |            |  |  monitoring)|         |                 |
  +--------+---+  +-----+------+         +-------+---------+
           |             |                        |
           |  REST API   |  SMTP (email alerts)   | WebSocket
           |  (store to  |  + SMS Gateway         | (real-time
           |   TimescaleDB)                       |  updates)
           v             v                        v
  +--------+---+  +------+-------+  +-------------+------+
  | Time-Series|  | Notification |  | WebSocket Server   |
  | Database   |  | Service      |  | (pushes live data  |
  |            |  |              |  |  to dashboard)      |
  +------------+  +--------------+  +--------------------+
```

**Protocol choices explained:**

| Layer | Protocol | Why |
|-------|----------|-----|
| Sensors to Broker | MQTT QoS 1 | Sensors are constrained (low power, limited bandwidth). MQTT's 2-byte overhead is ideal. QoS 1 ensures at-least-once delivery |
| Broker to Services | MQTT subscribe | Services subscribe to relevant topics for real-time processing |
| Data Ingestion to DB | REST/HTTP | Standard CRUD storage; no real-time requirement for writes |
| Alerts | SMTP + SMS | Email is the standard for alert delivery; SMS for critical alerts |
| Dashboard | WebSocket | Operators need real-time updates without page refresh. Bidirectional allows filter changes |

### Message Flow Example

```
Sensor publishes temperature reading:

1. Sensor -> MQTT Broker
   Topic: factory/floor-2/machine-42/temperature
   Payload: {"value": 87.3, "unit": "C", "ts": 1706900000}
   QoS: 1 (at-least-once)
   Packet size: ~80 bytes

2. MQTT Broker -> Alert Engine (subscriber)
   Rule: IF temperature > 85 THEN alert
   Action: Trigger SMTP email to maintenance team

3. MQTT Broker -> Data Ingestion Service (subscriber)
   Action: POST to TimescaleDB via REST

4. Data Ingestion -> WebSocket Server -> Dashboard
   Push: {"machine": "42", "temp": 87.3, "status": "WARNING"}
   Dashboard updates in <100ms from sensor reading
```

## 6. Benefits and Drawbacks

### Benefits

| Benefit | Description |
|---------|-------------|
| **Optimized performance** | Each protocol is tuned for its target scenario (e.g., MQTT's 2-byte header vs. HTTP's ~700-byte headers) |
| **Better semantics** | Protocol-level support for domain concepts (QoS levels, topic hierarchies, peer discovery) reduces application-level complexity |
| **Resource efficiency** | Constrained devices can participate in the system without the overhead of general-purpose protocols |
| **Industry standardization** | Domain protocols (SMTP, FIX, AMQP) ensure interoperability across vendors and organizations |
| **Appropriate reliability** | Choose the exact delivery guarantee needed (fire-and-forget for telemetry, exactly-once for trades) |
| **Leverages domain expertise** | Decades of domain-specific optimization (e.g., FIX protocol's 30+ years in financial markets) |

### Drawbacks

| Drawback | Description |
|----------|-------------|
| **Smaller ecosystem** | Fewer libraries, tools, and community resources compared to HTTP/REST |
| **Specialized knowledge** | Team members need protocol-specific expertise; harder to hire for niche protocols |
| **Tooling gaps** | Debugging, monitoring, and observability tools may be less mature than HTTP equivalents |
| **Interoperability challenges** | Connecting systems that use different domain protocols requires translation layers or gateways |
| **Operational complexity** | Running and managing protocol-specific brokers/servers (MQTT broker, WebSocket server, TURN server) adds infrastructure burden |
| **Vendor lock-in risk** | Some protocol implementations are tied to specific vendors or platforms |
| **Gateway/proxy issues** | Corporate firewalls and proxies may block non-HTTP protocols; WebSocket and MQTT over WebSocket help mitigate this |

### When to Use Domain-Specific Protocols

```
Use domain-specific protocols when:

  [x] General-purpose protocols create measurable bottlenecks
  [x] The domain has an established standard protocol (email=SMTP, trading=FIX)
  [x] Client constraints make HTTP impractical (IoT, embedded)
  [x] Real-time bidirectional communication is a core requirement
  [x] Peer-to-peer communication is needed (WebRTC)
  [x] You need fine-grained QoS control per message

Stick with general-purpose protocols when:

  [x] Standard CRUD operations are the primary pattern
  [x] Team lacks expertise in the specialized protocol
  [x] The system is simple enough that HTTP polling is acceptable
  [x] Interoperability with many external systems is critical
  [x] Debugging simplicity and tool availability are priorities
```

## 7. Related Patterns

| Pattern | Relationship |
|---------|-------------|
| [RPI (Remote Procedure Invocation)](./rpi.md) | General-purpose synchronous protocol pattern; domain-specific protocols address cases where RPI is insufficient |
| [Messaging](./messaging.md) | General-purpose asynchronous protocol pattern; MQTT and AMQP are domain-specific refinements of messaging |
| API Gateway | Can act as a protocol translation layer, converting between client-facing protocols (WebSocket) and internal protocols (gRPC, AMQP) |
| Backend for Frontend (BFF) | Different frontends may need different protocols (WebSocket for web, MQTT for mobile IoT companion app) |
| Event-Driven Architecture | Domain-specific protocols like MQTT and AMQP are often the transport layer for event-driven systems |
| Circuit Breaker | Resilience patterns apply to domain-specific protocol connections too (e.g., reconnection logic for WebSocket) |

## 8. Real-World Usage

| Company/System | Protocol | Use Case |
|----------------|----------|----------|
| **WhatsApp** | Custom XMPP-based | Messaging at scale with minimal bandwidth usage; their modified protocol handles billions of messages daily |
| **Slack** | WebSocket | Real-time message delivery to connected clients; falls back to polling when WebSocket is unavailable |
| **Bloomberg Terminal** | FIX + proprietary | Financial market data distribution with sub-millisecond latency requirements |
| **AWS IoT Core** | MQTT | Connects billions of IoT devices; supports MQTT 3.1.1 and 5.0 with device shadows for offline state |
| **Zoom** | WebRTC + custom | Video conferencing using WebRTC for peer connections and custom protocols for server-mediated large meetings |
| **Robinhood** | FIX | Order execution and market data feeds connecting to stock exchanges |
| **Tesla** | MQTT | Vehicle telemetry data collection from millions of cars to central data platform |
| **Discord** | WebSocket + WebRTC | WebSocket for chat presence and events; WebRTC for voice channels |
| **Google Gmail** | SMTP + IMAP | Standard email protocols for sending (SMTP) and receiving (IMAP) across the internet |
| **Home Assistant** | MQTT | Smart home automation hub using MQTT to communicate with thousands of IoT device types |

## 9. Summary

Domain-specific protocols fill the gaps that general-purpose protocols (REST, gRPC, traditional messaging) leave open. They are not replacements for general-purpose protocols but rather **complements** used in specific parts of the system where specialized behavior is required.

**Key takeaways:**

1. **Start with general-purpose protocols** ([RPI](./rpi.md) for synchronous, [Messaging](./messaging.md) for asynchronous) and only adopt domain-specific protocols when there is a clear, measurable need.
2. **Match the protocol to the domain**: MQTT for IoT, WebSocket for real-time web apps, FIX for trading, SMTP for email, WebRTC for peer-to-peer media.
3. **Consider the trade-offs**: better performance and semantics come at the cost of specialized tooling, knowledge requirements, and operational complexity.
4. **Multi-protocol architectures are normal**: real-world systems commonly use 3-5 different protocols, each serving a specific communication need.
5. **Protocol translation gateways** (API Gateway, protocol bridges) are essential when mixing multiple protocols in a single system.

```
Summary Decision Tree:

   Need to communicate?
         |
         v
   Is REST/HTTP sufficient?
   |                    |
  YES                  NO
   |                    |
   Use REST        Is it server-push only?
                   |                   |
                  YES                 NO
                   |                   |
                  SSE           Is it P2P media?
                               |              |
                              YES            NO
                               |              |
                            WebRTC      Are clients constrained?
                                        |                |
                                       YES              NO
                                        |                |
                                      MQTT          WebSocket
                                                   (or gRPC streaming)
```

---

*Domain-specific protocols are the "right tool for the right job" principle applied to inter-service and client-server communication. Knowing when to step beyond REST and gRPC is a mark of mature distributed system design.*
