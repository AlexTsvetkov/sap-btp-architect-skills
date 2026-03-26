# Lesson 2.4 — Kyma Eventing & Extension Patterns

## Table of Contents

- [1. Side-by-Side Extension Model](#1-side-by-side-extension-model)
- [2. S/4HANA Event Integration](#2-s4hana-event-integration)
- [3. Extension Patterns](#3-extension-patterns)
- [4. Resilience Patterns](#4-resilience-patterns)
- [5. Event Processing Best Practices](#5-event-processing-best-practices)
- [Top 5 Pitfalls](#top-5-pitfalls)
- [What to Learn Next](#what-to-learn-next)

---

> **Summary:** SAP's recommended approach for extending S/4HANA and other SAP systems is "side-by-side" extensions on BTP — keeping the clean core clean while building custom logic externally. Kyma provides the eventing infrastructure and runtime for these extensions. This lesson covers the side-by-side extension model, event-driven architecture patterns, S/4HANA event integration, and production-ready extension design.

---

## 1. Side-by-Side Extension Model

### Clean Core Principle

```
┌──────────────────────────────────┐     ┌──────────────────────────────────┐
│        S/4HANA (Clean Core)       │     │     Kyma Runtime (Extensions)     │
│                                   │     │                                   │
│  ┌──────────────────────────────┐ │     │  ┌──────────────────────────────┐ │
│  │  Standard Business Logic     │ │     │  │  Custom Logic                │ │
│  │  (no modifications)          │ │     │  │  - Validations               │ │
│  └──────────────────────────────┘ │     │  │  - Enrichments               │ │
│                                   │     │  │  - Integrations              │ │
│  ┌──────────────────────────────┐ │     │  │  - Notifications             │ │
│  │  Released APIs (stable)      │─┼─────┼─→│  - Custom workflows          │ │
│  └──────────────────────────────┘ │     │  └──────────────────────────────┘ │
│                                   │     │                                   │
│  ┌──────────────────────────────┐ │     │  ┌──────────────────────────────┐ │
│  │  Business Events             │─┼─────┼─→│  Event Subscribers           │ │
│  │  (BP Changed, SO Created...) │ │     │  │  (Functions / Microservices) │ │
│  └──────────────────────────────┘ │     │  └──────────────────────────────┘ │
└──────────────────────────────────┘     └──────────────────────────────────┘
              │                                          │
              └──────────── Event Mesh ──────────────────┘
```

**Why side-by-side?**
- S/4HANA upgrades are not blocked by custom code
- Extensions can be developed, tested, and deployed independently
- Different lifecycle: extensions can iterate faster than core ERP
- Clear API boundaries via Released APIs and Business Events

---

## 2. S/4HANA Event Integration

### Event Flow: S/4HANA → Kyma

```
S/4HANA Cloud              SAP Event Mesh           Kyma Runtime
     │                          │                        │
     │ BP Changed event         │                        │
     │ (RAP Business Event)     │                        │
     │─────────────────────────→│                        │
     │                          │ CloudEvent format      │
     │                          │────────────────────────→│
     │                          │                        │ Subscription match
     │                          │                        │ → deliver to sink
     │                          │                        │
     │                          │                        │ ┌─────────────────┐
     │                          │                        │ │ Event Handler   │
     │                          │                        │ │ (Function/Pod)  │
     │                          │                        │ └────────┬────────┘
     │                          │                        │          │
     │                          │                        │   Call S/4 API
     │←─────────────────────────┼────────────────────────┼──────────┘
     │  GET /API_BUSINESS_PARTNER│                       │
     │  (Released API)           │                        │
```

### Common S/4HANA Business Events

| Event Type | Description | Use Case |
|---|---|---|
| `sap.s4.beh.businesspartner.changed.v1` | BP master data changed | Sync to CRM/MDM |
| `sap.s4.beh.businesspartner.created.v1` | New BP created | Trigger onboarding |
| `sap.s4.beh.salesorder.created.v1` | Sales order created | Fulfillment workflow |
| `sap.s4.beh.salesorder.changed.v1` | Sales order modified | Update downstream |
| `sap.s4.beh.purchaseorder.created.v1` | Purchase order created | Approval workflow |
| `sap.s4.beh.product.changed.v1` | Product master changed | Catalog sync |

### Event Payload (CloudEvents Format)

```json
{
  "specversion": "1.0",
  "type": "sap.s4.beh.businesspartner.changed.v1",
  "source": "/default/sap.s4.beh/XXXXXXXXXX",
  "id": "ABY+LHK...",
  "time": "2024-03-15T14:30:00Z",
  "datacontenttype": "application/json",
  "data": {
    "BusinessPartner": "1000042"
  }
}
```

> **Important:** S/4HANA events carry minimal data (usually just the key). You must call back to S/4HANA APIs to get the full entity data.

---

## 3. Extension Patterns

### Pattern 1: Event-Driven Data Sync

Sync business partner changes from S/4HANA to an external CRM:

```java
@Component
@ServiceName("EventProcessorService")
public class BPSyncHandler implements EventHandler {

    @Autowired
    private S4HanaClient s4Client;
    
    @Autowired
    private CrmClient crmClient;

    @On(event = "sap.s4.beh.businesspartner.changed.v1")
    public void onBPChanged(EventContext context) {
        String bpId = context.get("data").get("BusinessPartner").asText();
        
        // 1. Fetch full BP data from S/4HANA (Released API)
        BusinessPartner bp = s4Client.getBusinessPartner(bpId);
        
        // 2. Transform to CRM format
        CrmContact contact = CrmContact.builder()
            .externalId(bp.getBusinessPartner())
            .name(bp.getBusinessPartnerFullName())
            .email(bp.getEmailAddress())
            .build();
        
        // 3. Upsert in CRM
        crmClient.upsertContact(contact);
        
        log.info("Synced BP {} to CRM", bpId);
    }
}
```

### Pattern 2: Validation Extension (Pre-Check via API)

Validate business rules before S/4HANA processing using a custom API:

```
Fiori App → Custom Validation API (Kyma) → S/4HANA API
                     │
                     ├── Check credit limit (external service)
                     ├── Check sanctions list
                     └── Return allow/deny
```

```java
@On(event = CdsService.EVENT_CREATE, entity = "OrderValidation")
public void validateOrder(CdsCreateEventContext context) {
    CdsData order = context.getCqn().entries().get(0);
    
    String customerId = order.get("customerId").toString();
    BigDecimal amount = (BigDecimal) order.get("totalAmount");
    
    // External credit check
    CreditCheckResult credit = creditService.check(customerId, amount);
    if (!credit.isApproved()) {
        throw new ServiceException(ErrorStatuses.FORBIDDEN, 
            "Credit limit exceeded for customer " + customerId);
    }
    
    // Sanctions screening
    boolean sanctioned = sanctionsService.screen(customerId);
    if (sanctioned) {
        throw new ServiceException(ErrorStatuses.FORBIDDEN,
            "Customer is on sanctions list");
    }
    
    context.setResult(Map.of("approved", true, "creditScore", credit.getScore()));
}
```

### Pattern 3: Notification Extension

Send notifications when key business events occur:

```javascript
// Kyma Function: notify-on-large-order
module.exports = {
  main: async function (event, context) {
    const orderData = JSON.parse(event.data);
    const orderId = orderData.SalesOrder;
    
    // Fetch full order from S/4HANA
    const order = await fetchOrder(orderId);
    
    if (order.TotalNetAmount > 100000) {
      // Send Teams notification
      await sendTeamsNotification({
        title: `Large Order Created: ${orderId}`,
        text: `Customer: ${order.SoldToParty}\nAmount: ${order.TotalNetAmount} ${order.TransactionCurrency}`,
        urgency: "high"
      });
      
      // Send email to sales manager
      await sendEmail({
        to: order.SalesManagerEmail,
        subject: `Action Required: Large Order ${orderId}`,
        body: `A sales order exceeding 100K has been created...`
      });
    }
    
    return { statusCode: 200 };
  }
};
```

### Pattern 4: Saga / Orchestration

Coordinate multiple services with compensation on failure:

```
Order Created Event
      │
      ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ 1. Reserve       │───→│ 2. Process       │───→│ 3. Send          │
│    Inventory     │    │    Payment       │    │    Confirmation  │
│    (S/4HANA)     │    │    (Stripe)      │    │    (Email)       │
└────────┬────────┘    └────────┬────────┘    └─────────────────┘
         │ failure              │ failure
         ▼                      ▼
┌─────────────────┐    ┌─────────────────┐
│ Compensate:      │    │ Compensate:      │
│ Release inventory│    │ Release inventory│
│                  │    │ Refund payment   │
└─────────────────┘    └─────────────────┘
```

---

## 4. Resilience Patterns

### Retry with Exponential Backoff

```yaml
apiVersion: eventing.kyma-project.io/v1alpha2
kind: Subscription
metadata:
  name: bp-sync-sub
spec:
  source: ""
  types:
    - sap.s4.beh.businesspartner.changed.v1
  sink: http://bp-sync-service.my-app.svc.cluster.local
  config:
    maxInFlightMessages: "10"
```

In the handler, implement retry logic:

```java
@Retryable(
    value = {S4HanaException.class, HttpTimeoutException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 2.0)
)
public BusinessPartner fetchBPWithRetry(String bpId) {
    return s4Client.getBusinessPartner(bpId);
}
```

### Dead Letter Queue Pattern

```javascript
module.exports = {
  main: async function (event, context) {
    try {
      await processEvent(event);
    } catch (error) {
      // After max retries, send to dead letter topic
      if (getRetryCount(event) >= MAX_RETRIES) {
        await publishToDeadLetter(event, error);
        return { statusCode: 200 }; // ACK to prevent redelivery
      }
      // Return error to trigger redelivery
      return { statusCode: 500, body: { error: error.message } };
    }
  }
};
```

### Circuit Breaker

```java
@CircuitBreaker(name = "s4hana", fallbackMethod = "s4Fallback")
public BusinessPartner getBusinessPartner(String id) {
    return s4HanaApi.getByKey(id);
}

public BusinessPartner s4Fallback(String id, Exception ex) {
    log.warn("S/4HANA circuit open, using cached data for BP {}", id);
    return cache.get("bp:" + id);
}
```

---

## 5. Event Processing Best Practices

### Idempotency

Events may be delivered more than once. Always design idempotent handlers:

```java
@On(event = "sap.s4.beh.businesspartner.changed.v1")
public void onBPChanged(EventContext context) {
    String eventId = context.get("id").asText();
    
    // Check if already processed
    if (eventStore.exists(eventId)) {
        log.info("Event {} already processed, skipping", eventId);
        return;
    }
    
    // Process event
    processBusinessPartnerChange(context);
    
    // Mark as processed
    eventStore.save(eventId);
}
```

### Event Ordering

Events from the same source may arrive out of order. Use timestamps or version numbers:

```java
public void syncBusinessPartner(BusinessPartner incoming) {
    BusinessPartner existing = repository.findById(incoming.getId());
    
    if (existing != null && existing.getLastChanged().isAfter(incoming.getLastChanged())) {
        log.warn("Stale event for BP {}, skipping", incoming.getId());
        return;
    }
    
    repository.save(incoming);
}
```

---

## Top 5 Pitfalls

1. **Treating events as commands.** Events report what happened ("BP changed"), not what to do. Don't assume the event payload has all the data — call back to APIs for details.
2. **Not implementing idempotency.** Events can be delivered multiple times. Without idempotent handlers, you'll get duplicate processing.
3. **Tight coupling to S/4HANA availability.** If your event handler synchronously calls S/4HANA and it's down, you lose events. Use retry + DLQ patterns.
4. **Ignoring event ordering.** Two rapid changes to the same BP may arrive in reverse order. Use timestamps to detect stale events.
5. **Building a monolithic extension.** Split extensions by bounded context (one service for BP sync, another for order validation). Each should have its own subscriptions and lifecycle.

---

## What to Learn Next

- **Lesson 1.3:** Event Mesh — deep dive into messaging infrastructure
- **Lesson 2.1:** Kyma Architecture — eventing module internals
- **Lesson 2.3:** Serverless Functions — lightweight event handlers
- **Lesson 3.3:** CAP Remote Services — calling S/4HANA APIs from CAP