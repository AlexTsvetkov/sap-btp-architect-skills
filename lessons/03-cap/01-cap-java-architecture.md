# Lesson 3.1 — CAP Java Architecture & Core Concepts

> **Summary:** The SAP Cloud Application Programming Model (CAP) for Java provides a framework built on Spring Boot that adds CDS-based domain modeling, a generic runtime for OData/REST services, an event-driven handler model, and built-in BTP service integration. This lesson covers the CAP Java architecture, CDS compilation pipeline, the event/handler model, CQN query API, and how CAP compares to pure Spring Boot development.

---

## 1. CAP Java Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    CAP Java Application                      │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  CDS Model Layer (.cds files)                        │    │
│  │  - Domain entities, types, associations              │    │
│  │  - Service definitions (projections, actions)        │    │
│  │  - Annotations (UI, auth, validation)                │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │ cds compile → CSN (JSON)           │
│  ┌──────────────────────▼──────────────────────────────┐    │
│  │  CAP Java Runtime                                    │    │
│  │  ┌──────────────┐ ┌──────────────┐ ┌─────────────┐  │    │
│  │  │ OData V4     │ │ REST         │ │ Messaging    │  │    │
│  │  │ Adapter      │ │ Adapter      │ │ Adapter      │  │    │
│  │  └──────┬───────┘ └──────┬───────┘ └──────┬──────┘  │    │
│  │         │                │                │          │    │
│  │  ┌──────▼────────────────▼────────────────▼──────┐   │    │
│  │  │            Generic Event Processor             │   │    │
│  │  │  (dispatches CQN events to handlers)           │   │    │
│  │  └──────────────────────┬─────────────────────────┘   │    │
│  │                         │                              │    │
│  │  ┌──────────────────────▼──────────────────────────┐  │    │
│  │  │         Custom Event Handlers (your code)        │  │    │
│  │  │  @Before / @On / @After                          │  │    │
│  │  └──────────────────────┬──────────────────────────┘  │    │
│  │                         │                              │    │
│  │  ┌──────────────────────▼──────────────────────────┐  │    │
│  │  │         Persistence Layer                        │  │    │
│  │  │  CQN → SQL (HANA / H2 / PostgreSQL)             │  │    │
│  │  └─────────────────────────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Spring Boot Foundation                                 │  │
│  │  DI, Auto-config, Actuator, Security, Web Server        │  │
│  └────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### CAP = Spring Boot + CDS + Generic Runtime

| What CAP Adds | What Spring Boot Provides |
|---|---|
| CDS modeling language | Dependency injection |
| Generic CRUD for OData/REST | Web server (Tomcat) |
| Event-driven handler model | Auto-configuration |
| CQN query abstraction | Actuator, profiles |
| BTP service integration | Security framework |
| Draft support | Transaction management |
| Fiori annotations | Testing infrastructure |

---

## 2. CDS Compilation Pipeline

### From .cds to Runtime

```
.cds files          cds compile          CSN (JSON)          CAP Runtime
                                                              
schema.cds    ─┐                    ┌─ db/csn.json      ─┐
service.cds   ─┤──→ cds compile ──→├─ srv/csn.json     ─┤──→ Java classes
annotations.cds─┘                    └─ EDMX metadata    ─┘    + generic handlers
```

### Domain Model (schema.cds)

```cds
namespace my.bookshop;

entity Books {
    key ID     : UUID;
    title      : String(200)  @mandatory;
    descr      : String(2000);
    author     : Association to Authors;
    genre      : Association to Genres;
    stock      : Integer      @assert.range: [0, 999];
    price      : Decimal(10,2);
    currency   : Currency;
    createdAt  : Timestamp    @cds.on.insert: $now;
    modifiedAt : Timestamp    @cds.on.insert: $now  @cds.on.update: $now;
}

entity Authors {
    key ID    : UUID;
    name      : String(100)  @mandatory;
    books     : Composition of many Books on books.author = $self;
    country   : Country;
}

entity Genres : sap.common.CodeList {
    key ID    : Integer;
    parent    : Association to Genres;
    children  : Composition of many Genres on children.parent = $self;
}
```

### Service Definition (service.cds)

```cds
using my.bookshop as db from '../db/schema';

service CatalogService @(path: '/catalog') {

    @readonly
    entity Books as projection on db.Books {
        *, author.name as authorName
    } excluding { createdAt, modifiedAt };

    @requires: 'Admin'
    entity Authors as projection on db.Authors;

    // Unbound action
    action submitOrder(book : Books:ID, quantity : Integer) returns { stock : Integer };

    // Bound function
    function getBooksByAuthor(authorId : Authors:ID) returns array of Books;
}
```

### What Gets Generated

From these CDS files, the CAP build produces:
1. **CSN** (Core Schema Notation) — JSON representation of the model
2. **EDMX** — OData V4 metadata document
3. **HDI artifacts** — SQL tables, views for HANA deployment
4. **Java interfaces** — Optional typed accessor interfaces

---

## 3. Event Handler Model

### Handler Registration

```java
@Component
@ServiceName(CatalogService_.CDS_NAME)
public class CatalogServiceHandler implements EventHandler {

    @Autowired
    private PersistenceService db;

    // BEFORE: validation, enrichment (runs before generic handler)
    @Before(event = CqnService.EVENT_CREATE, entity = Books_.CDS_NAME)
    public void validateBook(CdsCreateEventContext context) {
        context.getCqn().entries().forEach(book -> {
            String title = (String) book.get("title");
            if (title == null || title.isBlank()) {
                throw new ServiceException(ErrorStatuses.BAD_REQUEST, "Title is required");
            }
        });
    }

    // ON: replace generic handler (full control)
    @On(event = CqnService.EVENT_READ, entity = Books_.CDS_NAME)
    public void readBooks(CdsReadEventContext context) {
        // Custom read logic (e.g., external API call)
        CqnSelect query = context.getCqn();
        Result result = db.run(query);
        context.setResult(result);
    }

    // AFTER: post-processing (modify result before returning)
    @After(event = CqnService.EVENT_READ, entity = Books_.CDS_NAME)
    public void afterReadBooks(CdsReadEventContext context) {
        context.getResult().forEach(book -> {
            Integer stock = (Integer) book.get("stock");
            book.put("availability", stock > 0 ? "In Stock" : "Out of Stock");
        });
    }

    // Custom action handler
    @On(event = "submitOrder")
    public void onSubmitOrder(SubmitOrderContext context) {
        String bookId = context.getBook();
        Integer quantity = context.getQuantity();

        CqnSelect select = Select.from(Books_.class).byId(bookId);
        Books book = db.run(select).single(Books.class);

        if (book.getStock() < quantity) {
            throw new ServiceException(ErrorStatuses.CONFLICT, "Not enough stock");
        }

        book.setStock(book.getStock() - quantity);
        CqnUpdate update = Update.entity(Books_.class).data(book);
        db.run(update);

        context.setResult(Map.of("stock", book.getStock()));
        context.setCompleted();
    }
}
```

### Event Processing Order

```
HTTP Request (OData/REST)
    │
    ▼
┌─────────────┐
│  @Before     │ ← Validation, enrichment, authorization checks
│  handlers    │   (multiple handlers possible, ordered by @HandlerOrder)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  @On         │ ← Core logic (generic CRUD or your custom handler)
│  handler     │   Only ONE @On handler executes (first registered wins)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  @After      │ ← Post-processing (modify result, trigger side effects)
│  handlers    │   (multiple handlers possible)
└──────┬──────┘
       │
       ▼
HTTP Response
```

---

## 4. CQN Query API

CQN (CDS Query Notation) is CAP's type-safe query builder, analogous to JPA Criteria API or jOOQ:

### Select Queries

```java
// Simple select
CqnSelect query = Select.from(Books_.class);

// With filter
CqnSelect query = Select.from(Books_.class)
    .where(b -> b.stock().gt(0)
        .and(b.price().lt(50)));

// With expand (join)
CqnSelect query = Select.from(Books_.class)
    .columns(b -> b._all(), b -> b.author().expand())
    .where(b -> b.genre().name().eq("Fiction"))
    .orderBy(b -> b.title().asc())
    .limit(10, 0);

// By key
CqnSelect byId = Select.from(Books_.class)
    .byId("550e8400-e29b-41d4-a716-446655440000");
```

### Insert, Update, Delete

```java
// Insert
Map<String, Object> bookData = Map.of(
    "ID", UUID.randomUUID().toString(),
    "title", "Clean Code",
    "price", 29.99,
    "stock", 100
);
CqnInsert insert = Insert.into(Books_.class).entry(bookData);
Result result = db.run(insert);

// Update
CqnUpdate update = Update.entity(Books_.class)
    .data("stock", 50)
    .byId(bookId);
db.run(update);

// Upsert
CqnUpsert upsert = Upsert.into(Books_.class).entry(bookData);
db.run(upsert);

// Delete
CqnDelete delete = Delete.from(Books_.class)
    .where(b -> b.stock().eq(0));
db.run(delete);
```

### CQN vs JPA/Spring Data

| CQN (CAP) | JPA / Spring Data |
|---|---|
| `Select.from(Books_.class)` | `bookRepository.findAll()` |
| `.where(b -> b.stock().gt(0))` | `@Query("SELECT b FROM Book b WHERE b.stock > 0")` |
| `.columns(b -> b.title(), b -> b.price())` | Projection interfaces |
| `db.run(query).listOf(Books.class)` | Repository method return |
| CDS associations (auto-join) | `@ManyToOne`, `@OneToMany` |
| No entity manager / session | EntityManager, L1/L2 cache |

---

## 5. Project Structure

```
my-cap-project/
├── pom.xml                    # Parent POM
├── db/
│   ├── schema.cds             # Domain model
│   └── data/                  # CSV test data
│       └── my.bookshop-Books.csv
├── srv/
│   ├── pom.xml                # Service module POM
│   ├── src/main/
│   │   ├── resources/
│   │   │   ├── application.yaml
│   │   │   └── edmx/          # Generated OData metadata
│   │   └── java/
│   │       └── com/sap/demo/
│   │           ├── Application.java
│   │           └── handlers/
│   │               └── CatalogServiceHandler.java
│   └── cat-service.cds        # Service definition
├── app/                        # Fiori UI (optional)
└── mta.yaml                   # BTP deployment descriptor
```

### Key Maven Dependencies

```xml
<dependencies>
    <!-- CAP Java SDK -->
    <dependency>
        <groupId>com.sap.cds</groupId>
        <artifactId>cds-starter-spring-boot</artifactId>
    </dependency>
    <!-- OData V4 protocol adapter -->
    <dependency>
        <groupId>com.sap.cds</groupId>
        <artifactId>cds-adapter-odata-v4</artifactId>
    </dependency>
    <!-- HANA database support -->
    <dependency>
        <groupId>com.sap.cds</groupId>
        <artifactId>cds-feature-hana</artifactId>
    </dependency>
    <!-- XSUAA security -->
    <dependency>
        <groupId>com.sap.cds</groupId>
        <artifactId>cds-feature-xsuaa</artifactId>
    </dependency>
</dependencies>
```

---

## Top 5 Pitfalls

1. **Bypassing the CQN API to use raw JDBC.** CAP's generic runtime (authorization, draft, localization) only works through CQN. Raw SQL bypasses all of it.
2. **Putting business logic in `@After` instead of `@On`.** `@After` runs after the database operation. For validation or transformation before persistence, use `@Before`.
3. **Not understanding that `@On` replaces the generic handler.** If you register an `@On` handler for READ, the built-in CRUD handler won't run — you must handle the full query yourself or delegate to `db.run()`.
4. **Ignoring the CDS model as the single source of truth.** Don't define database tables manually. Let CDS generate HDI artifacts, EDMX metadata, and typed interfaces from one model.
5. **Using Spring Data repositories alongside CAP persistence.** CAP has its own persistence layer. Mixing Spring Data JPA with CAP causes confusion about which layer handles transactions, caching, and authorization.

---

## What to Learn Next

- **Lesson 3.2:** CAP Security — `@requires`, `@restrict`, XSUAA integration
- **Lesson 3.3:** CAP Remote Services — connecting to S/4HANA and external APIs
- **Lesson 3.4:** CAP Advanced Patterns — draft, localization, deployment topologies
- **Lesson 3.5:** CAP Fiori Elements — annotations for automatic UI generation