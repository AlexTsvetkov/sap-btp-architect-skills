# Lesson 3.3 — CAP Remote Services & Mashups

## Table of Contents

- [1. Remote Service Architecture](#1-remote-service-architecture)
- [2. Importing External Service Definitions](#2-importing-external-service-definitions)
- [3. Destination Configuration](#3-destination-configuration)
- [4. Consuming Remote Services in Handlers](#4-consuming-remote-services-in-handlers)
- [5. Error Handling & Resilience](#5-error-handling-resilience)
- [Top 5 Pitfalls](#top-5-pitfalls)
- [What to Learn Next](#what-to-learn-next)

---

> **Summary:** CAP Java supports integrating external services — S/4HANA OData APIs, REST APIs, and other CAP services — as "remote services" defined in CDS. This enables mashup scenarios where your application combines local data with remote data, projecting external entities alongside your own. This lesson covers importing external APIs, the Destination Service, remote service consumption, mashup patterns, and error handling.

---

## 1. Remote Service Architecture

```
┌────────────────────────────────────────────────────────────┐
│                  CAP Java Application                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  CDS Model                                           │   │
│  │  ┌──────────────┐  ┌──────────────────────────────┐  │   │
│  │  │ Local Entities│  │ External Service (imported)   │  │   │
│  │  │ (Books,      │  │ (API_BUSINESS_PARTNER)       │  │   │
│  │  │  Orders)     │  │                              │  │   │
│  │  └──────┬───────┘  └──────────────┬───────────────┘  │   │
│  │         │                         │                   │   │
│  │  ┌──────▼─────────────────────────▼───────────────┐   │   │
│  │  │  Mashup Service                                 │   │   │
│  │  │  (combines local + remote in one projection)    │   │   │
│  │  └──────────────────────┬──────────────────────────┘   │   │
│  └─────────────────────────┼──────────────────────────────┘   │
│                            │                                   │
│  ┌─────────────────────────▼──────────────────────────────┐   │
│  │  CAP Runtime                                            │   │
│  │  ┌──────────────────┐  ┌────────────────────────────┐  │   │
│  │  │ Local DB (HANA)  │  │ Remote HTTP Client          │  │   │
│  │  │ via CQN → SQL    │  │ via Destination Service     │  │   │
│  │  └──────────────────┘  └─────────────┬──────────────┘  │   │
│  └──────────────────────────────────────┼─────────────────┘   │
└─────────────────────────────────────────┼─────────────────────┘
                                          │
                          ┌───────────────▼───────────────┐
                          │  BTP Destination Service       │
                          │  ┌─────────────────────────┐   │
                          │  │ S4HANA_DEST              │   │
                          │  │  URL: https://s4.sap.com │   │
                          │  │  Auth: OAuth2SAMLBearer  │   │
                          │  └─────────────────────────┘   │
                          └───────────────┬───────────────┘
                                          │
                          ┌───────────────▼───────────────┐
                          │  S/4HANA Cloud                 │
                          │  /sap/opu/odata/sap/           │
                          │  API_BUSINESS_PARTNER          │
                          └───────────────────────────────┘
```

---

## 2. Importing External Service Definitions

### Step 1: Download the EDMX

From the SAP Business Accelerator Hub or your S/4HANA system:

```bash
# Download API_BUSINESS_PARTNER EDMX
curl -o srv/external/API_BUSINESS_PARTNER.edmx \
  "https://api.sap.com/api/API_BUSINESS_PARTNER/resource/edmx"
```

### Step 2: Import in CDS

```cds
// srv/external/API_BUSINESS_PARTNER.cds (auto-generated from EDMX)
// Run: cds import srv/external/API_BUSINESS_PARTNER.edmx

@cds.external
service API_BUSINESS_PARTNER {
    @cds.persistence.skip
    entity A_BusinessPartner {
        key BusinessPartner         : String(10);
        BusinessPartnerFullName     : String(81);
        FirstName                   : String(40);
        LastName                    : String(40);
        BusinessPartnerCategory     : String(1);
        to_BusinessPartnerAddress   : Composition of many A_BusinessPartnerAddress;
    }

    @cds.persistence.skip
    entity A_BusinessPartnerAddress {
        key BusinessPartner : String(10);
        key AddressID       : String(10);
        Country             : String(3);
        CityName            : String(40);
        StreetName          : String(60);
        PostalCode          : String(10);
    }
}
```

### Step 3: Reference in Your Service

```cds
using { API_BUSINESS_PARTNER as s4bp } from './external/API_BUSINESS_PARTNER';
using { my.bookshop as db } from '../db/schema';

service MashupService {
    // Local entity
    entity Orders as projection on db.Orders;

    // Remote entity (projected from imported service)
    @readonly
    entity BusinessPartners as projection on s4bp.A_BusinessPartner {
        key BusinessPartner,
        BusinessPartnerFullName,
        FirstName,
        LastName
    };
}
```

---

## 3. Destination Configuration

### BTP Destination Setup

```
Destination Name: S4HANA_BP
URL:              https://my-s4hana.s4hana.ondemand.com
Authentication:   OAuth2SAMLBearerAssertion
Audience:         https://my-s4hana.s4hana.ondemand.com
Token URL:        https://my-s4hana.s4hana.ondemand.com/sap/bc/sec/oauth2/token
Client ID:        <communication arrangement client>
Client Secret:    <communication arrangement secret>
```

### CAP Configuration for Remote Services

```yaml
# application.yaml
cds:
  remote:
    services:
      API_BUSINESS_PARTNER:
        destination:
          name: S4HANA_BP
          type: odata-v2      # S/4HANA uses OData V2
          suffix: /sap/opu/odata/sap/API_BUSINESS_PARTNER
```

Or via `package.json` / `.cdsrc.json`:

```json
{
  "cds": {
    "requires": {
      "API_BUSINESS_PARTNER": {
        "kind": "odata-v2",
        "model": "srv/external/API_BUSINESS_PARTNER",
        "[production]": {
          "credentials": {
            "destination": "S4HANA_BP",
            "path": "/sap/opu/odata/sap/API_BUSINESS_PARTNER"
          }
        }
      }
    }
  }
}
```

---

## 4. Consuming Remote Services in Handlers

### Basic Remote Service Call

```java
@Component
@ServiceName(MashupService_.CDS_NAME)
public class MashupServiceHandler implements EventHandler {

    @Autowired
    @Qualifier(API_BUSINESS_PARTNER_.CDS_NAME)
    private CqnService bpService;  // Remote service (injected by CAP)

    @On(event = CqnService.EVENT_READ, entity = "MashupService.BusinessPartners")
    public void readBusinessPartners(CdsReadEventContext context) {
        // CAP translates CQN to OData query and sends to S/4HANA
        CqnSelect query = context.getCqn();
        Result result = bpService.run(query);
        context.setResult(result);
    }
}
```

### Mashup: Combine Local + Remote Data

```java
@After(event = CqnService.EVENT_READ, entity = Orders_.CDS_NAME)
public void enrichOrdersWithBPData(CdsReadEventContext context) {
    List<Map<String, Object>> orders = context.getResult().listOf(HashMap.class);

    // Collect unique BP IDs
    Set<String> bpIds = orders.stream()
        .map(o -> (String) o.get("customerId"))
        .filter(Objects::nonNull)
        .collect(Collectors.toSet());

    if (bpIds.isEmpty()) return;

    // Batch fetch from S/4HANA
    CqnSelect bpQuery = Select.from(A_BusinessPartner_.class)
        .columns(bp -> bp.BusinessPartner(), bp -> bp.BusinessPartnerFullName())
        .where(bp -> bp.BusinessPartner().in(bpIds));

    Map<String, String> bpNames = bpService.run(bpQuery)
        .streamOf(A_BusinessPartner_.class)
        .collect(Collectors.toMap(
            A_BusinessPartner::getBusinessPartner,
            A_BusinessPartner::getBusinessPartnerFullName
        ));

    // Enrich local orders with remote BP names
    orders.forEach(order -> {
        String bpId = (String) order.get("customerId");
        order.put("customerName", bpNames.getOrDefault(bpId, "Unknown"));
    });
}
```

### Custom Action: Create in Remote System

```java
@On(event = "createBusinessPartner")
public void createBP(CreateBusinessPartnerContext context) {
    Map<String, Object> bpData = new HashMap<>();
    bpData.put("BusinessPartnerCategory", "1"); // Person
    bpData.put("FirstName", context.getFirstName());
    bpData.put("LastName", context.getLastName());

    CqnInsert insert = Insert.into(A_BusinessPartner_.class).entry(bpData);

    try {
        Result result = bpService.run(insert);
        context.setResult(result.single(A_BusinessPartner_.class));
    } catch (ServiceException e) {
        throw new ServiceException(ErrorStatuses.BAD_GATEWAY,
            "Failed to create BP in S/4HANA: " + e.getMessage(), e);
    }
}
```

---

## 5. Error Handling & Resilience

### Timeout Configuration

```yaml
cds:
  remote:
    services:
      API_BUSINESS_PARTNER:
        http:
          timeout: 30000       # 30 seconds
          connectionTimeout: 5000  # 5 seconds
```

### Error Handling Pattern

```java
@On(event = CqnService.EVENT_READ, entity = "MashupService.BusinessPartners")
public void readBPWithFallback(CdsReadEventContext context) {
    try {
        Result result = bpService.run(context.getCqn());
        context.setResult(result);
    } catch (ServiceException e) {
        if (isTransientError(e)) {
            // Retry once
            try {
                Thread.sleep(1000);
                Result result = bpService.run(context.getCqn());
                context.setResult(result);
            } catch (Exception retryEx) {
                log.error("Retry failed for BP read", retryEx);
                throw new ServiceException(ErrorStatuses.BAD_GATEWAY,
                    "S/4HANA temporarily unavailable");
            }
        } else {
            throw new ServiceException(ErrorStatuses.BAD_GATEWAY,
                "Failed to read from S/4HANA: " + e.getMessage());
        }
    }
}

private boolean isTransientError(ServiceException e) {
    int status = e.getErrorStatus().getCode();
    return status == 503 || status == 429 || status == 408;
}
```

### Caching Remote Data

```java
@Autowired
private CacheManager cacheManager;

@Cacheable(value = "businessPartners", key = "#bpId", unless = "#result == null")
public A_BusinessPartner getCachedBP(String bpId) {
    CqnSelect query = Select.from(A_BusinessPartner_.class).byId(bpId);
    return bpService.run(query).single(A_BusinessPartner_.class);
}
```

```yaml
spring:
  cache:
    caffeine:
      spec: maximumSize=500,expireAfterWrite=300s  # 5 min TTL
```

---

## Top 5 Pitfalls

1. **Not using the Destination Service.** Hardcoding S/4HANA URLs and credentials in your app is a security risk and breaks environment portability. Always use BTP Destinations.
2. **Fetching too much data from remote systems.** CQN queries on remote services translate to OData `$filter`, `$select`, `$top`. If you fetch all BPs without filters, you'll time out. Always project and filter.
3. **Ignoring OData version mismatch.** S/4HANA uses OData V2. CAP's remote service client handles translation, but you must configure `type: odata-v2` correctly.
4. **No error handling for remote calls.** Remote systems go down. Without try/catch, a single S/4HANA timeout will return 500 to your users instead of a graceful degradation.
5. **N+1 query problem with mashups.** Fetching BP data for each order individually creates N remote calls. Always batch-fetch remote data using `IN` filters.

---

## What to Learn Next

- **Lesson 3.1:** CAP Architecture — handler model and CQN queries
- **Lesson 3.4:** CAP Advanced Patterns — deployment topologies, caching strategies
- **Lesson 2.4:** Kyma Eventing — event-driven integration alternative to synchronous API calls
- **Lesson 1.3:** Event Mesh — asynchronous data synchronization patterns