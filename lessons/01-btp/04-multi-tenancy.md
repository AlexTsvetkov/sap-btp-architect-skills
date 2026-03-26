# Lesson 1.4 — BTP Multi-Tenancy

## Table of Contents

- [1. BTP Multi-Tenancy Model](#1-btp-multi-tenancy-model)
- [2. Tenant Isolation Patterns](#2-tenant-isolation-patterns)
- [3. XSUAA in Multi-Tenant Context](#3-xsuaa-in-multi-tenant-context)
- [4. CAP Multi-Tenancy Support (@sap/cds-mtxs)](#4-cap-multi-tenancy-support-sapcds-mtxs)
- [5. Operational Concerns](#5-operational-concerns)
- [Top 5 Pitfalls](#top-5-pitfalls)
- [What to Learn Next](#what-to-learn-next)

---

> **Summary:** Multi-tenancy on SAP BTP enables building SaaS applications where a single deployment serves multiple customers (tenants) with data isolation, independent configuration, and per-tenant extensibility. This lesson covers the BTP multi-tenancy lifecycle, tenant isolation patterns, XSUAA identity zones, CAP's MTX framework, and operational concerns — compared with Spring Boot / Hibernate multi-tenancy patterns.

---

## 1. BTP Multi-Tenancy Model

### Core Concept

In BTP's multi-tenancy model, your application is deployed once in a **provider subaccount**, and customer organizations **subscribe** from their own subaccounts:

```
┌─────────────────────────────────┐
│     Provider Subaccount          │
│     (your SaaS app deployed)     │
│                                  │
│  ┌────────────────────────────┐  │
│  │  App Instance (shared)     │  │
│  │  - Single CF app / K8s pod │  │
│  │  - Shared code, shared URL │  │
│  └────────────────────────────┘  │
└──────────┬───────────────────────┘
           │ subscriptions
     ┌─────┼──────────────┐
     │     │              │
     ▼     ▼              ▼
┌────────┐ ┌────────┐ ┌────────┐
│Tenant A│ │Tenant B│ │Tenant C│
│(sub-   │ │(sub-   │ │(sub-   │
│account)│ │account)│ │account)│
│        │ │        │ │        │
│ Own HDI│ │ Own HDI│ │ Own HDI│
│ Own IDP│ │ Own IDP│ │ Own IDP│
└────────┘ └────────┘ └────────┘
```

### SaaS Provisioning Service (saas-registry)

The `saas-registry` service manages the tenant subscription lifecycle:

```json
{
  "xsappname": "my-saas-app",
  "appUrls": {
    "onSubscription": "https://my-saas-app-srv.cfapps.eu10.hana.ondemand.com/mt/v1.0/subscriptions/tenants/{tenantId}",
    "getDependencies": "https://my-saas-app-srv.cfapps.eu10.hana.ondemand.com/mt/v1.0/subscriptions/dependencies"
  }
}
```

### Subscription Lifecycle

```
Tenant Admin                SaaS Registry           Your App (callback)        HANA Cloud
     │                           │                        │                       │
     │ Subscribe to app          │                        │                       │
     │──────────────────────────→│                        │                       │
     │                           │ PUT /subscriptions/    │                       │
     │                           │     tenants/{tenantId} │                       │
     │                           │───────────────────────→│                       │
     │                           │                        │                       │
     │                           │                        │ Create HDI container  │
     │                           │                        │──────────────────────→│
     │                           │                        │      OK              │
     │                           │                        │←─────────────────────│
     │                           │                        │                       │
     │                           │                        │ Deploy schema         │
     │                           │                        │──────────────────────→│
     │                           │                        │      OK              │
     │                           │                        │←─────────────────────│
     │                           │                        │                       │
     │                           │  200 OK + app URL      │                       │
     │                           │←──────────────────────│                       │
     │  App available at         │                        │                       │
     │  tenant-a.my-app.com      │                        │                       │
     │←─────────────────────────│                        │                       │
```

---

## 2. Tenant Isolation Patterns

### Schema-Per-Tenant (HDI Containers)

This is the **default and recommended** pattern on BTP:

```
HANA Cloud Instance
├── HDI Container (Tenant A) → Schema: TENANT_A_HDI
│   ├── Table: BOOKS
│   └── Table: AUTHORS
├── HDI Container (Tenant B) → Schema: TENANT_B_HDI
│   ├── Table: BOOKS
│   └── Table: AUTHORS
└── HDI Container (Tenant C) → Schema: TENANT_C_HDI
    ├── Table: BOOKS
    └── Table: AUTHORS
```

**Pros:**
- Complete data isolation at the database level
- Tenants cannot accidentally access each other's data
- Independent schema evolution per tenant (for extensibility)
- GDPR compliance: delete tenant = drop HDI container

**Cons:**
- Resource overhead: each container has metadata tables
- Schema upgrades must propagate to all containers
- Cross-tenant queries impossible (by design)
- HANA Cloud connection limits may become a bottleneck

### Shared-Schema with Discriminator Column

All tenants share the same tables, distinguished by a `tenant_id` column:

```sql
CREATE TABLE books (
    id UUID PRIMARY KEY,
    tenant_id VARCHAR(36) NOT NULL,  -- discriminator
    title VARCHAR(200),
    price DECIMAL(10,2)
);

-- Every query MUST include tenant_id filter
SELECT * FROM books WHERE tenant_id = 'tenant-a-guid' AND price > 10;
```

**Pros:**
- Lower resource overhead
- Simpler deployment (single schema)
- Cross-tenant analytics possible (with proper authorization)

**Cons:**
- Risk of data leakage if tenant filter is missing
- No database-level isolation
- Complex GDPR data deletion
- CAP does **not** support this pattern natively

### Comparison with Spring Boot / Hibernate

| Aspect | Spring Boot + Hibernate | CAP on BTP |
|---|---|---|
| Schema-per-tenant | `MultiTenantConnectionProvider` + `CurrentTenantIdentifierResolver` | Built-in via `@sap/cds-mtxs` |
| Shared schema | `@TenantId` column + Hibernate filters | Not natively supported |
| Tenant resolution | Custom filter / interceptor | Automatic from JWT `zid` claim |
| Schema migration | Flyway per-tenant loop | HDI deployer per-container |
| Tenant provisioning | Custom code | `saas-registry` callbacks |

---

## 3. XSUAA in Multi-Tenant Context

### Tenant Resolution via Subdomain

The **approuter** resolves the tenant from the URL subdomain:

```
https://tenant-a.my-saas-app.cfapps.eu10.hana.ondemand.com
         ^^^^^^^^
         subdomain → maps to tenant identity zone
```

The approuter:
1. Extracts subdomain from the request URL
2. Resolves it to an XSUAA identity zone
3. Redirects the user to the tenant-specific XSUAA for authentication
4. Receives a JWT with the tenant's `zid` (zone ID) claim

### JWT Tenant Claims

```json
{
  "zid": "tenant-a-zone-guid",
  "ext_attr": {
    "subaccountid": "tenant-a-subaccount-guid",
    "zdn": "tenant-a"
  },
  "iss": "https://tenant-a.authentication.eu10.hana.ondemand.com/oauth/token",
  "scope": ["my-saas-app!t12345.Viewer"],
  "client_id": "sb-my-saas-app!t12345"
}
```

The `zid` claim is used by CAP to route database queries to the correct HDI container.

### xs-security.json for Multi-Tenancy

```json
{
  "xsappname": "my-saas-app",
  "tenant-mode": "shared",
  "scopes": [
    { "name": "$XSAPPNAME.Viewer", "description": "View data" },
    { "name": "$XSAPPNAME.Admin", "description": "Administer" },
    { "name": "$XSAPPNAME.mtcallback", "description": "MT callback", "grant-as-authority-to-same-serviceplan": ["application"] }
  ],
  "role-templates": [
    { "name": "Viewer", "scope-references": ["$XSAPPNAME.Viewer"] },
    { "name": "Admin", "scope-references": ["$XSAPPNAME.Admin", "$XSAPPNAME.Viewer"] }
  ]
}
```

Note `"tenant-mode": "shared"` — this enables multi-tenant operation. The `mtcallback` scope allows the saas-registry to call your onSubscription endpoint.

---

## 4. CAP Multi-Tenancy Support (@sap/cds-mtxs)

### Architecture

CAP provides the `@sap/cds-mtxs` module (MTX Sidecar) that handles:
- Tenant provisioning (HDI container creation)
- Schema deployment per tenant
- Schema upgrade propagation
- Tenant extensibility (custom CDS extensions per tenant)

```
┌─────────────────────────────────────────────┐
│              CAP Application                 │
│                                             │
│  ┌─────────────────┐  ┌─────────────────┐  │
│  │  Java Backend    │  │  MTX Sidecar    │  │
│  │  (CAP Java)      │  │  (Node.js)      │  │
│  │                  │  │                  │  │
│  │  - CDS services  │  │  - /mt/v1.0/... │  │
│  │  - Event handlers│  │  - HDI deployer  │  │
│  │  - OData/REST    │  │  - Extension mgr │  │
│  │                  │  │  - Schema upgrade│  │
│  └────────┬─────────┘  └────────┬─────────┘  │
│           │                      │            │
│           └──────────┬───────────┘            │
│                      │                        │
│               Service Bindings                │
│  (XSUAA, saas-registry, service-manager)      │
└──────────────────────┬────────────────────────┘
                       │
              ┌────────┴────────┐
              │  HANA Cloud      │
              │  ┌──────┐       │
              │  │HDI-A │       │
              │  │HDI-B │       │
              │  │HDI-C │       │
              │  └──────┘       │
              └─────────────────┘
```

### Sidecar Configuration

```json
// package.json (MTX sidecar)
{
  "cds": {
    "requires": {
      "multitenancy": true,
      "extensibility": true,
      "toggles": true
    },
    "mtx": {
      "element-prefix": ["Z_", "MY_"],
      "namespace-blocklist": ["com.sap.", "sap."]
    }
  }
}
```

### Schema Upgrade Propagation

When you deploy a new version of your CDS model, the MTX sidecar must upgrade all tenant schemas:

```bash
# Trigger upgrade for all tenants
cds-mtx upgrade --all

# Or via REST API
POST /mtx/v1/model/upgrade
{
  "tenants": ["*"]  // all tenants
}
```

The sidecar iterates through all subscribed tenants, deploys the new schema to each HDI container, and handles migration. This is atomic per tenant — if one tenant's upgrade fails, others are not affected.

---

## 5. Operational Concerns

### Tenant-Aware Logging

```java
import org.slf4j.MDC;

@Before(event = "*")
public void addTenantToMDC(EventContext ctx) {
    String tenantId = ctx.getUserInfo().getTenant();
    MDC.put("tenant_id", tenantId);
}
```

Logback pattern:

```xml
<pattern>%d{ISO8601} [%thread] %-5level [tenant=%X{tenant_id}] %logger{36} - %msg%n</pattern>
```

### Noisy-Neighbor Detection

Monitor per-tenant resource consumption:

```sql
-- Per-tenant HDI container memory usage
SELECT SCHEMA_NAME,
       ROUND(SUM(MEMORY_SIZE_IN_TOTAL) / 1024 / 1024, 2) AS total_mb,
       SUM(RECORD_COUNT) AS total_records
FROM M_CS_TABLES
GROUP BY SCHEMA_NAME
ORDER BY total_mb DESC;
```

### GDPR: Tenant Data Deletion

With schema-per-tenant, GDPR "right to erasure" is straightforward:

```bash
# Unsubscribe tenant → triggers onSubscription DELETE callback
# MTX sidecar drops the HDI container
DELETE /mt/v1.0/subscriptions/tenants/{tenantId}
```

This physically removes all tenant data, making compliance verification simple.

### Connection Pool Considerations

Each tenant's HDI container requires database connections. With 100 tenants and a pool size of 5 per tenant, you need 500 connections:

```yaml
# application.yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 5        # per tenant
      minimum-idle: 1             # keep connections warm
      idle-timeout: 300000        # 5 minutes
      connection-timeout: 30000   # 30 seconds
```

> **Pitfall:** HANA Cloud has connection limits per instance size. Monitor `M_CONNECTIONS` and size your HANA instance accordingly.

---

## Top 5 Pitfalls

1. **Forgetting `tenant-mode: shared` in xs-security.json.** Without this, XSUAA creates a dedicated (single-tenant) service instance.
2. **Not testing schema upgrades with multiple tenants.** A CDS model change that works for new tenants may fail on existing tenants with data.
3. **Ignoring connection pool sizing.** More tenants = more HDI containers = more connections needed. Plan HANA instance sizing early.
4. **Missing the MTX sidecar in deployment.** The Java backend alone cannot handle tenant provisioning — the Node.js sidecar is required.
5. **Not implementing tenant-aware logging from day one.** Without tenant context in logs, debugging production issues across tenants is nearly impossible.

---

## What to Learn Next

- **Lesson 1.1:** BTP Architecture — subaccount hierarchy and entitlements
- **Lesson 1.2:** HANA Cloud — HDI container internals
- **Lesson 3.2:** CAP Security — authorization in multi-tenant context
- **Lesson 3.4:** CAP Advanced Patterns — deployment topologies for MTX
- SAP BTP Multi-Tenancy documentation