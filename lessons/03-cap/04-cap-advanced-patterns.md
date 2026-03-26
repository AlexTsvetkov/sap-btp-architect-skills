# Lesson 3.4 — CAP Advanced Patterns & Production Readiness

> **Summary:** Production CAP Java applications require mastering draft handling for Fiori UIs, localization/i18n, temporal data, deployment topologies (MTA vs Helm), and performance optimization. This lesson covers these advanced patterns with practical examples and configuration.

---

## 1. Draft Handling

### What is Draft?

Draft enables Fiori's "edit session" pattern: a user opens an entity for editing, makes changes over multiple requests, then saves or discards. Changes are stored in a draft table until explicitly activated.

```
User clicks "Edit"     User modifies fields     User clicks "Save"
       │                      │                        │
       ▼                      ▼                        ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐
│ DRAFT_NEW    │    │ Draft table  │    │ Copy draft → active  │
│ creates draft│───→│ updated      │───→│ Delete draft          │
│ copy         │    │ (multiple    │    │ Return active entity  │
│              │    │  requests)   │    │                       │
└──────────────┘    └──────────────┘    └──────────────────────┘
```

### CDS Annotation

```cds
service AdminService {
    @odata.draft.enabled
    entity Books as projection on db.Books;
}
```

This auto-generates:
- `DRAFT_Books` table for storing in-progress edits
- Draft-specific OData actions: `draftEdit`, `draftActivate`, `draftPrepare`

### Draft Handler in Java

```java
@Component
@ServiceName(AdminService_.CDS_NAME)
public class AdminServiceHandler implements EventHandler {

    // Runs when user clicks "Save" (draft → active)
    @Before(event = DraftService.EVENT_DRAFT_SAVE, entity = Books_.CDS_NAME)
    public void validateOnSave(DraftSaveEventContext context) {
        CqnSelect draftQuery = Select.from(Books_.class)
            .where(b -> b.IsActiveEntity().eq(false)  // draft version
                .and(b.ID().eq(context.getCqn().entries().get(0).get("ID"))));

        Books draft = db.run(draftQuery).single(Books.class);

        if (draft.getTitle() == null || draft.getTitle().isBlank()) {
            throw new ServiceException(ErrorStatuses.BAD_REQUEST,
                "Title is required before saving");
        }
        if (draft.getPrice() == null || draft.getPrice().compareTo(BigDecimal.ZERO) <= 0) {
            throw new ServiceException(ErrorStatuses.BAD_REQUEST,
                "Price must be positive");
        }
    }

    // Runs during draft preparation (field validation while editing)
    @Before(event = DraftService.EVENT_DRAFT_PREPARE, entity = Books_.CDS_NAME)
    public void prepareWarnings(DraftPrepareEventContext context) {
        // Side-effect free validation — add warnings but don't reject
        Messages messages = context.getMessages();
        // Add warnings that show in Fiori UI but don't block save
    }
}
```

### Draft Timeout

```yaml
cds:
  drafts:
    cancellation-timeout: 15d  # Auto-delete drafts after 15 days
```

---

## 2. Localization (i18n)

### CDS Text Bundles

```cds
// db/schema.cds
entity Books {
    key ID : UUID;
    title  : localized String(200);  // enables translations
    descr  : localized String(2000);
}
```

The `localized` keyword generates a `Books.texts` table:

```sql
-- Auto-generated
CREATE TABLE my_bookshop_Books_texts (
    locale VARCHAR(14),
    ID UUID,
    title VARCHAR(200),
    descr VARCHAR(2000),
    PRIMARY KEY (locale, ID)
);
```

### i18n Properties Files

```properties
# srv/src/main/resources/i18n/messages.properties (default/English)
book_not_found=Book with ID {0} not found
order_submitted=Order {0} submitted successfully
insufficient_stock=Not enough stock for book {0}. Available: {1}

# srv/src/main/resources/i18n/messages_de.properties (German)
book_not_found=Buch mit ID {0} nicht gefunden
order_submitted=Bestellung {0} erfolgreich eingereicht
insufficient_stock=Nicht genug Bestand für Buch {0}. Verfügbar: {1}
```

### Using Localized Messages in Handlers

```java
@On(event = "submitOrder")
public void onSubmitOrder(SubmitOrderContext context) {
    Books book = db.run(Select.from(Books_.class).byId(context.getBook()))
        .first(Books.class)
        .orElseThrow(() -> new ServiceException(ErrorStatuses.NOT_FOUND,
            MessageKeys.BOOK_NOT_FOUND, context.getBook()));

    if (book.getStock() < context.getQuantity()) {
        throw new ServiceException(ErrorStatuses.CONFLICT,
            MessageKeys.INSUFFICIENT_STOCK, book.getTitle(), book.getStock());
    }
    // ... process order
}
```

---

## 3. Temporal Data & Audit Logging

### Managed Aspects (Auto-filled Fields)

```cds
using { managed, cuid } from '@sap/cds/common';

entity Orders : cuid, managed {
    // cuid provides: key ID : UUID;
    // managed provides: createdAt, createdBy, modifiedAt, modifiedBy
    orderNo    : String(20);
    totalAmount: Decimal(15,2);
    status     : String(20);
}
```

CAP automatically fills these fields:
- `createdAt` / `modifiedAt` — server timestamp
- `createdBy` / `modifiedBy` — from JWT user ID

### Audit Logging with @changelog

```cds
@changelog: [title, price, stock]
entity Books : managed {
    key ID : UUID;
    title  : String(200);
    price  : Decimal(10,2);
    stock  : Integer;
}
```

CAP generates change log entries for annotated fields, recording old value, new value, user, and timestamp.

### Soft Delete Pattern

```cds
entity Orders : managed {
    key ID      : UUID;
    // ... fields ...
    deletedAt   : Timestamp;
    deletedBy   : String(255);
}

// Service shows only non-deleted
service OrderService {
    entity Orders as projection on db.Orders
        where deletedAt is null;
}
```

```java
@On(event = CqnService.EVENT_DELETE, entity = Orders_.CDS_NAME)
public void softDelete(CdsDeleteEventContext context) {
    // Instead of physical delete, set deleted fields
    String orderId = extractId(context);
    CqnUpdate update = Update.entity(Orders_.class)
        .data("deletedAt", Instant.now())
        .data("deletedBy", context.getUserInfo().getName())
        .byId(orderId);
    db.run(update);
    context.setResult(List.of(Map.of("ID", orderId)));
    context.setCompleted(); // prevent generic delete handler
}
```

---

## 4. Deployment Topologies

### MTA (Cloud Foundry)

```yaml
# mta.yaml
_schema-version: "3.1"
ID: my-cap-app
version: 1.0.0

modules:
  - name: my-cap-srv
    type: java
    path: srv
    parameters:
      memory: 1024M
      buildpack: sap_java_buildpack_jakarta
    properties:
      SPRING_PROFILES_ACTIVE: cloud
      JBP_CONFIG_COMPONENTS: '{jres: ["com.sap.xs.java.buildpack.jre.SAPMachineJRE"]}'
    requires:
      - name: my-xsuaa
      - name: my-hana
      - name: my-destination

  - name: my-cap-db-deployer
    type: hdb
    path: gen/db
    requires:
      - name: my-hana

resources:
  - name: my-xsuaa
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json

  - name: my-hana
    type: com.sap.xs.hdi-container
    parameters:
      service: hana
      service-plan: hdi-shared

  - name: my-destination
    type: org.cloudfoundry.managed-service
    parameters:
      service: destination
      service-plan: lite
```

### Helm (Kyma)

```
my-cap-helm/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment-srv.yaml
│   ├── deployment-sidecar.yaml    # MTX sidecar (if multi-tenant)
│   ├── service.yaml
│   ├── apirule.yaml
│   ├── service-instance-xsuaa.yaml
│   ├── service-instance-hana.yaml
│   ├── service-binding-xsuaa.yaml
│   ├── service-binding-hana.yaml
│   └── job-db-deploy.yaml          # HDI deployer as K8s Job
```

### CF vs Kyma Deployment Comparison

| Aspect | Cloud Foundry (MTA) | Kyma (Helm) |
|---|---|---|
| Packaging | MTA archive (.mtar) | Docker image + Helm chart |
| DB deployment | `hdb` module (auto) | K8s Job with HDI deployer image |
| Service binding | MTA `requires` | BTP Operator CRDs |
| Scaling | CF `instances` parameter | K8s HPA |
| Routing | CF routes + approuter | APIRule + Istio |
| Sidecar (MTX) | Separate CF app | Separate K8s Deployment |

---

## 5. Performance Optimization

### Query Optimization

```java
// BAD: fetches all columns, all rows
CqnSelect bad = Select.from(Books_.class);

// GOOD: project only needed columns, filter, limit
CqnSelect good = Select.from(Books_.class)
    .columns(b -> b.ID(), b -> b.title(), b -> b.price())
    .where(b -> b.stock().gt(0))
    .orderBy(b -> b.title().asc())
    .limit(50);
```

### Batch Operations

```java
// BAD: N individual inserts
for (Map<String, Object> book : books) {
    db.run(Insert.into(Books_.class).entry(book));
}

// GOOD: single batch insert
db.run(Insert.into(Books_.class).entries(books));
```

### Connection Pool Tuning

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

### Caching with Spring Cache

```java
@Cacheable(value = "genres", key = "'all'")
public List<Genres> getAllGenres() {
    return db.run(Select.from(Genres_.class)).listOf(Genres.class);
}

@CacheEvict(value = "genres", allEntries = true)
public void onGenreChanged() {
    // Called when genres are modified
}
```

---

## Top 5 Pitfalls

1. **Not enabling draft for Fiori edit scenarios.** Without `@odata.draft.enabled`, Fiori Elements uses "direct edit" mode which doesn't support multi-step editing or concurrent user detection.
2. **Ignoring the `localized` keyword.** Adding translations retroactively requires database migration. Plan for i18n from the start using `localized` on user-facing String fields.
3. **Deploying without a startup probe.** CAP Java apps with CDS compilation at startup can take 30-60s. Without a startup probe, Kubernetes kills the pod before it's ready.
4. **Not batching database operations.** Individual inserts/updates in a loop are orders of magnitude slower than batch operations. Always use `.entries(list)`.
5. **MTA descriptor out of sync with actual services.** Forgetting to add a new service to `mta.yaml` causes deployment failures. Keep the MTA descriptor as the definitive deployment manifest.

---

## What to Learn Next

- **Lesson 3.5:** CAP Fiori Elements — UI annotations for automatic UI generation
- **Lesson 1.4:** Multi-Tenancy — MTX sidecar deployment patterns
- **Lesson 4.1:** CI/CD — automating MTA builds and Helm deployments
- **Lesson 4.2:** Observability — monitoring CAP applications in production