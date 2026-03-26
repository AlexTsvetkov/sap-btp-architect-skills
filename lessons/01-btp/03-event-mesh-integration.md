# Lesson 1.3 — BTP Event Mesh & Integration

> **Summary:** Event-driven architecture on SAP BTP centers on SAP Event Mesh — a fully managed messaging service that implements the CloudEvents specification and integrates natively with S/4HANA business events. This lesson covers Event Mesh internals, messaging patterns, comparison with Kafka/RabbitMQ, SAP Integration Suite positioning, and a hands-on implementation of an event-driven CAP Java microservice.

---

## 1. SAP Event Mesh Architecture

### Overview

SAP Event Mesh is a multi-protocol message broker built on top of SAP's messaging infrastructure. It supports AMQP 1.0, MQTT 3.1.1, and HTTP (REST) protocols.

```
┌──────────────────────────────────────────────────────────┐
│                    SAP Event Mesh                         │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │              Message Broker Cluster                │    │
│  │                                                    │    │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐        │    │
│  │  │ Queue A │   │ Queue B │   │ Queue C │  ...    │    │
│  │  └────┬────┘   └────┬────┘   └────┬────┘        │    │
│  │       │              │              │             │    │
│  │  Topic Subscriptions (pattern matching)           │    │
│  │  sap/s4/beh/businesspartner/changed/v1           │    │
│  │  sap/s4/beh/salesorder/created/v1                │    │
│  │  my/app/custom/event/v1                          │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  Protocols: AMQP 1.0 | MQTT 3.1.1 | HTTP/REST           │
│  Format: CloudEvents 1.0                                 │
└──────────────────────────────────────────────────────────┘
```

### Queues vs Topic Subscriptions

Event Mesh uses a **topic-based publish/subscribe** model with **queue-based consumption**:

- **Topics** are hierarchical strings (e.g., `sap/s4/beh/businesspartner/changed/v1`). Publishers send messages to topics.
- **Queues** are durable message stores. Consumers read from queues.
- **Topic subscriptions** bind a queue to one or more topic patterns. A queue receives all messages matching its subscribed topics.

```
Publisher → Topic: sap/s4/beh/salesorder/created/v1
                    │
                    ├──→ Queue "order-processor" (subscribed: sap/s4/beh/salesorder/*)
                    │         → Consumer A (microservice)
                    │
                    └──→ Queue "audit-logger" (subscribed: sap/s4/beh/+/+/v1)
                              → Consumer B (audit service)
```

Topic patterns support:
- `*` — matches one segment (e.g., `sap/s4/beh/*/created/v1`)
- `>` — matches one or more segments at the end (e.g., `sap/s4/beh/>`)

### QoS Levels

| QoS | Meaning | Use Case |
|---|---|---|
| 0 (At most once) | Fire and forget, no ack | Telemetry, metrics |
| 1 (At least once) | Delivered at least once, may duplicate | Business events (default for S/4HANA events) |
| 2 (Exactly once) | Not supported by Event Mesh | — |

> **Critical:** Event Mesh provides **at-least-once** delivery. Your consumers **must be idempotent**. Design handlers to safely process the same event multiple times.

### Webhook Registration

For services that cannot maintain persistent connections (e.g., serverless), Event Mesh supports webhook delivery:

```json
{
  "name": "my-webhook-sub",
  "address": "queue:my-events",
  "webhookUrl": "https://my-app.cfapps.eu10.hana.ondemand.com/api/events",
  "qos": 1,
  "handshakeSecret": "my-secret-123"
}
```

Event Mesh performs a handshake (challenge-response) to verify webhook ownership before activating the subscription.

### Comparison with Kafka / RabbitMQ

| Feature | Event Mesh | Apache Kafka | RabbitMQ |
|---|---|---|---|
| Model | Topic → Queue (pub/sub) | Topic → Partition → Consumer Group | Exchange → Queue (pub/sub or direct) |
| Ordering | Per-queue FIFO | Per-partition ordering | Per-queue FIFO |
| Replay | No (consumed = gone) | Yes (offset-based replay) | No (ack = gone) |
| Retention | TTL-based | Time/size-based retention | TTL-based |
| Throughput | Medium (~1000 msg/s per queue) | Very high (~100K+ msg/s) | High (~10K+ msg/s) |
| Protocol | AMQP, MQTT, HTTP | Kafka protocol (TCP) | AMQP 0.9.1 |
| SAP integration | Native S/4HANA events | Requires connector | Requires connector |
| Management | Fully managed on BTP | Self-managed or Confluent | Self-managed or CloudAMQP |
| Dead letter | Manual (DLQ pattern) | Via connectors | Built-in DLX |

**Key takeaway for Kafka users:** Event Mesh does **not** support message replay. If you need event sourcing or replay capability, consider Kafka on Kyma or a hybrid architecture.

---

## 2. CloudEvents Specification

SAP standardized on the [CloudEvents](https://cloudevents.io/) specification for all business events. CloudEvents defines a common envelope format:

### Envelope Structure

```json
{
  "specversion": "1.0",
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "source": "/default/sap.s4.beh/XXXXXXXXXX",
  "type": "sap.s4.beh.businesspartner.changed.v1",
  "datacontenttype": "application/json",
  "time": "2024-11-15T10:30:00Z",
  "data": {
    "BusinessPartner": "1000042",
    "BusinessPartnerName": "ACME Corp",
    "ChangedFields": ["BusinessPartnerName", "SearchTerm1"]
  }
}
```

### S/4HANA Event Types

S/4HANA emits events following a naming convention:

```
sap.s4.beh.<business-object>.<operation>.v<version>

Examples:
sap.s4.beh.businesspartner.changed.v1
sap.s4.beh.businesspartner.created.v1
sap.s4.beh.salesorder.created.v1
sap.s4.beh.salesorder.changed.v1
sap.s4.beh.purchaseorder.released.v1
sap.s4.beh.product.created.v1
```

The `data` payload is intentionally minimal — it contains the key fields and changed fields, not the full business object. Your consumer is expected to **call back** to S/4HANA's OData API to fetch the complete data.

### Topic-to-Queue Mapping for S/4HANA Events

```
S/4HANA → Event Mesh Topic: sap/s4/beh/businesspartner/changed/v1
                                │
Queue subscription pattern: sap/s4/beh/businesspartner/*/v1
                                │
                                ▼
Queue: "bp-processor" → Your microservice receives the event
```

---

## 3. Enterprise Messaging Patterns

### Fan-Out

One event consumed by multiple independent services:

```
                        ┌──→ Queue "enrichment" → Enrichment Service
                        │
Topic: order.created ───┼──→ Queue "inventory"  → Inventory Service
                        │
                        └──→ Queue "notification" → Notification Service
```

Each queue has its own subscription to the same topic. Each consumer processes independently.

### Competing Consumers

Multiple instances of the same service consume from one queue for horizontal scaling:

```
Queue: "order-processor"
    ├──→ Instance 1 (processes message A)
    ├──→ Instance 2 (processes message B)
    └──→ Instance 3 (processes message C)
```

Event Mesh distributes messages round-robin across connected consumers on the same queue. Only one consumer receives each message.

### Dead-Letter Queue Pattern

Event Mesh doesn't have a built-in DLQ mechanism. Implement it in your application:

```java
@On(event = "sap.s4.beh.businesspartner.changed.v1")
public void handleBusinessPartnerChanged(EventContext ctx) {
    int retryCount = getRetryCount(ctx);
    
    try {
        processEvent(ctx.getData());
    } catch (TransientException e) {
        if (retryCount < MAX_RETRIES) {
            // Re-publish with incremented retry count and delay
            republishWithDelay(ctx, retryCount + 1, calculateBackoff(retryCount));
        } else {
            // Send to dead-letter queue for manual investigation
            publishToDeadLetterQueue(ctx);
            log.error("Event moved to DLQ after {} retries: {}", MAX_RETRIES, ctx.getId());
        }
    } catch (PermanentException e) {
        // Immediately DLQ — no retry will fix this
        publishToDeadLetterQueue(ctx);
        log.error("Permanent failure, moved to DLQ: {}", e.getMessage());
    }
}
```

### Retry with Exponential Backoff

```java
private Duration calculateBackoff(int retryCount) {
    // Base: 1s, then 2s, 4s, 8s, 16s (max 30s)
    long delayMs = Math.min(
        (long) Math.pow(2, retryCount) * 1000,
        30_000
    );
    // Add jitter to prevent thundering herd
    delayMs += ThreadLocalRandom.current().nextLong(0, delayMs / 4);
    return Duration.ofMillis(delayMs);
}
```

---

## 4. Integration Suite vs Event Mesh

### When to Use Each

```
┌─────────────────────────────┐    ┌─────────────────────────────┐
│   SAP Integration Suite      │    │   SAP Event Mesh             │
│                              │    │                              │
│  - Request/Response (sync)   │    │  - Fire-and-forget (async)   │
│  - Data transformation       │    │  - Pub/sub pattern           │
│  - Protocol mediation        │    │  - Event-driven extensions   │
│  - Complex routing (iFlows)  │    │  - Real-time notifications   │
│  - Batch data replication    │    │  - Loose coupling            │
│  - API management            │    │  - CloudEvents standard      │
│                              │    │                              │
│  Good for:                   │    │  Good for:                   │
│  - ETL / data sync           │    │  - Side-by-side extensions   │
│  - Legacy system adapters    │    │  - Microservice choreography │
│  - Complex transformations   │    │  - Decoupled architecture    │
└─────────────────────────────┘    └─────────────────────────────┘
```

### Hybrid Architecture

In practice, you often combine both:

```
S/4HANA
   │
   ├──→ Event Mesh (business events: BP changed, SO created)
   │         │
   │         └──→ Kyma microservice (real-time processing)
   │
   └──→ Integration Suite (batch replication: nightly sync of master data)
            │
            └──→ HANA Cloud (data warehouse / reporting)
```

**Rule of thumb:**
- Use **Event Mesh** when the consumer decides *when* and *how* to react to business events
- Use **Integration Suite** when you need data transformation, protocol mediation, or orchestrated integration flows

---

## 5. Implementing an Event-Driven CAP Java Microservice

### Project Setup

```
my-event-app/
├── srv/
│   ├── src/main/java/com/myapp/
│   │   └── handlers/
│   │       └── BusinessPartnerEventHandler.java
│   └── src/main/resources/
│       └── application.yaml
├── db/
│   └── schema.cds
├── package.json
├── pom.xml
└── mta.yaml
```

### CDS Model

```cds
// db/schema.cds
namespace my.events;

entity BusinessPartnerMirror {
    key bpId         : String(10);
    name             : String(200);
    city             : String(100);
    country          : String(3);
    lastSyncedAt     : Timestamp;
    sourceEvent      : String(100);
}

// srv/event-service.cds
using my.events from '../db/schema';

service EventService {
    entity BusinessPartners as projection on events.BusinessPartnerMirror;
}

// Declare event subscription
@protocol: 'none'  // no OData exposure for the event handler service
service EventHandlerService {
    event businessPartnerChanged : {
        BusinessPartner : String(10);
    };
}
```

### CAP Messaging Configuration

```yaml
# srv/src/main/resources/application.yaml
cds:
  messaging:
    services:
      - name: "messaging"
        kind: "enterprise-messaging"
        format: "cloudevents"
        subscriptions:
          - event: "sap.s4.beh.businesspartner.changed.v1"
            queue: "bp-changed-queue"
          - event: "sap.s4.beh.businesspartner.created.v1"
            queue: "bp-created-queue"
```

### Event Handler (Java)

```java
package com.myapp.handlers;

import com.sap.cds.services.EventContext;
import com.sap.cds.services.handler.EventHandler;
import com.sap.cds.services.handler.annotations.On;
import com.sap.cds.services.handler.annotations.ServiceName;
import com.sap.cds.services.messaging.TopicMessageEventContext;
import com.sap.cds.services.persistence.PersistenceService;
import com.sap.cds.ql.Upsert;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import java.time.Instant;
import java.util.Map;

@Component
@ServiceName("messaging")
public class BusinessPartnerEventHandler implements EventHandler {

    private static final Logger log = LoggerFactory.getLogger(BusinessPartnerEventHandler.class);

    private final PersistenceService db;
    private final S4HanaClient s4Client;  // SAP Cloud SDK client

    public BusinessPartnerEventHandler(PersistenceService db, S4HanaClient s4Client) {
        this.db = db;
        this.s4Client = s4Client;
    }

    @On(event = "sap.s4.beh.businesspartner.changed.v1")
    public void onBusinessPartnerChanged(TopicMessageEventContext ctx) {
        Map<String, Object> eventData = ctx.getData();
        String bpId = (String) eventData.get("BusinessPartner");
        
        log.info("Received BP changed event for: {}", bpId);

        // 1. Call back to S/4HANA to get full BP data
        BusinessPartner fullBp = s4Client.getBusinessPartner(bpId);

        // 2. Upsert into local mirror table (idempotent operation)
        Map<String, Object> mirrorData = Map.of(
            "bpId", bpId,
            "name", fullBp.getBusinessPartnerName(),
            "city", fullBp.getCity(),
            "country", fullBp.getCountry(),
            "lastSyncedAt", Instant.now().toString(),
            "sourceEvent", "sap.s4.beh.businesspartner.changed.v1"
        );

        db.run(Upsert.into("my.events.BusinessPartnerMirror").entry(mirrorData));
        
        log.info("BP {} mirrored successfully", bpId);

        // 3. Optionally publish a derived event
        publishDerivedEvent(bpId, fullBp);
    }

    @On(event = "sap.s4.beh.businesspartner.created.v1")
    public void onBusinessPartnerCreated(TopicMessageEventContext ctx) {
        // Similar logic, reuse the upsert pattern
        onBusinessPartnerChanged(ctx);
    }

    private void publishDerivedEvent(String bpId, BusinessPartner bp) {
        // Publish to a custom topic for downstream consumers
        Map<String, Object> derivedEvent = Map.of(
            "bpId", bpId,
            "enrichedName", bp.getBusinessPartnerName(),
            "region", mapCountryToRegion(bp.getCountry()),
            "timestamp", Instant.now().toString()
        );
        
        // CAP messaging API for publishing
        // messaging.emit("my.app.bp.enriched.v1", derivedEvent);
    }
}
```

### MTA Deployment Descriptor

```yaml
# mta.yaml
_schema-version: "3.1"
ID: my-event-app
version: 1.0.0

modules:
  - name: my-event-app-srv
    type: java
    path: srv
    parameters:
      memory: 1024M
      disk-quota: 512M
      buildpack: sap_java_buildpack
    requires:
      - name: my-event-app-db
      - name: my-event-app-messaging
      - name: my-event-app-xsuaa
      - name: my-event-app-destination

  - name: my-event-app-db-deployer
    type: hdb
    path: db
    requires:
      - name: my-event-app-db

resources:
  - name: my-event-app-db
    type: com.sap.xs.hdi-container
    parameters:
      service: hana
      service-plan: hdi-shared

  - name: my-event-app-messaging
    type: org.cloudfoundry.managed-service
    parameters:
      service: enterprise-messaging
      service-plan: default
      path: ./event-mesh.json  # Event Mesh descriptor

  - name: my-event-app-xsuaa
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json

  - name: my-event-app-destination
    type: org.cloudfoundry.managed-service
    parameters:
      service: destination
      service-plan: lite
```

### Event Mesh Service Descriptor

```json
// event-mesh.json
{
  "emname": "my-event-app",
  "version": "1.0.0",
  "namespace": "my/app/events",
  "options": {
    "management": true,
    "messagingrest": true
  },
  "rules": {
    "queueRules": {
      "publishFilter": ["${namespace}/**"],
      "subscribeFilter": ["${namespace}/**", "sap/s4/beh/**"]
    },
    "topicRules": {
      "publishFilter": ["${namespace}/**"],
      "subscribeFilter": ["${namespace}/**", "sap/s4/beh/**"]
    }
  }
}
```

### Sequence Diagram

```
S/4HANA          Event Mesh         CAP Java Service        HANA Cloud
   │                  │                    │                     │
   │ BP Changed       │                    │                     │
   │─────────────────→│                    │                     │
   │  (CloudEvent)    │                    │                     │
   │                  │ Queue: bp-changed  │                     │
   │                  │───────────────────→│                     │
   │                  │                    │                     │
   │                  │                    │ GET /BusinessPartner│
   │←─────────────────┼────────────────────│ ('1000042')        │
   │ {full BP data}   │                    │                     │
   │─────────────────→┼───────────────────→│                     │
   │                  │                    │                     │
   │                  │                    │ UPSERT mirror table │
   │                  │                    │────────────────────→│
   │                  │                    │        OK           │
   │                  │                    │←────────────────────│
   │                  │                    │                     │
   │                  │  Publish derived   │                     │
   │                  │←───────────────────│                     │
   │                  │ my/app/bp/enriched │                     │
   │                  │                    │                     │
```

---

## Top 5 Pitfalls

1. **Not designing for idempotency.** Event Mesh delivers at-least-once. Use UPSERT instead of INSERT, and check for duplicate event IDs.
2. **Expecting full payloads in S/4HANA events.** S/4HANA events are notifications with keys only. You must call back to get the full object.
3. **Ignoring topic subscription filters.** Overly broad filters (e.g., `sap/s4/beh/>`) cause your queue to receive events you don't need, wasting resources.
4. **Missing the Event Mesh service descriptor.** Without proper `rules.queueRules.subscribeFilter`, your app can't subscribe to S/4HANA event topics.
5. **Confusing Event Mesh with Kafka.** No replay, no consumer group offsets. If you need replay, architect a separate event store.

---

## What to Learn Next

- **Lesson 2.4:** Kyma Eventing & Extension Patterns — event-driven extensions on Kubernetes
- **Lesson 3.3:** CAP Remote Services — calling back to S/4HANA OData from CAP
- **Lesson 4.1:** CI/CD — deploying event-driven apps with MTA
- SAP Event Mesh REST API documentation
- CloudEvents specification: https://cloudevents.io/