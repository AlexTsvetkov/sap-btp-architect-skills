# Lesson 4.3 — SAP BTP for Spring Boot Developers — Migration Guide

## Table of Contents

- [1. Web Layer: Spring MVC → CAP Protocol Adapters](#1-web-layer-spring-mvc--cap-protocol-adapters)
- [2. Persistence: JPA + Hibernate → CDS + CQN](#2-persistence-jpa--hibernate--cds--cqn)
- [3. Security: Spring Security → XSUAA + @restrict](#3-security-spring-security--xsuaa--restrict)
- [4. Configuration, Messaging & Profiles](#4-configuration-messaging--profiles)
- [5. Testing & Observability](#5-testing--observability)
- [Top 5 Pitfalls](#top-5-pitfalls)
- [What to Learn Next](#what-to-learn-next)

---

> **Summary:** This lesson is a practical mapping guide for experienced Spring Boot developers transitioning to SAP CAP Java on BTP. For each core Spring concept — Web MVC, Data JPA, Security, Cloud Config, Actuator, Cloud Stream, profiles, testing, and migrations — we show the equivalent CAP/BTP mechanism, compare the two side by side with code examples, and give an honest assessment of what's better, what's worse, and what's just different. The goal is not to sell CAP but to build an accurate mental model so Spring developers can be productive immediately.

---

## 1. Web Layer: Spring MVC → CAP Protocol Adapters

### Spring Boot Approach

```java
// Spring Boot — explicit REST controller
@RestController
@RequestMapping("/api/books")
public class BookController {

    @Autowired
    private BookRepository bookRepository;

    @GetMapping
    public List<Book> getAll() {
        return bookRepository.findAll();
    }

    @GetMapping("/{id}")
    public Book getById(@PathVariable UUID id) {
        return bookRepository.findById(id)
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Book create(@Valid @RequestBody Book book) {
        return bookRepository.save(book);
    }

    @PutMapping("/{id}")
    public Book update(@PathVariable UUID id, @Valid @RequestBody Book book) {
        book.setId(id);
        return bookRepository.save(book);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable UUID id) {
        bookRepository.deleteById(id);
    }
}
```

### CAP Equivalent

```cds
// CDS model — service definition IS the API
service CatalogService @(path: '/catalog') {
    entity Books as projection on db.Books;
    action submitOrder(book : Books:ID, quantity : Integer) returns { stock : Integer };
}
```

```java
// CAP Java — only handle what you need to customize
@Component
@ServiceName(CatalogService_.CDS_NAME)
public class CatalogServiceHandler implements EventHandler {

    // No controller needed for CRUD — CAP generates OData/REST endpoints automatically
    // You only write handlers for custom logic

    @Before(event = CqnService.EVENT_CREATE, entity = Books_.CDS_NAME)
    public void validateBook(CdsCreateEventContext context) {
        // Validation logic
    }

    @On(event = "submitOrder")
    public void onSubmitOrder(SubmitOrderContext context) {
        // Custom action logic
    }
}
```

### Comparison

| Aspect | Spring MVC | CAP Protocol Adapters |
|---|---|---|
| Endpoint definition | Explicit `@GetMapping`, `@PostMapping` | Implicit from CDS model |
| CRUD boilerplate | You write it (or use Spring Data REST) | Zero — generated from CDS |
| Protocol | REST (JSON) | OData V4 (default) + REST |
| Query capabilities | Custom `@Query` or Specification | Built-in `$filter`, `$expand`, `$orderby`, `$select` |
| Custom endpoints | `@RequestMapping` | CDS actions/functions + `@On` handlers |
| URL structure | You control | Follows OData conventions (`/Entity(key)`) |
| Content negotiation | Spring's `ContentNegotiation` | OData format (`$format=json`) |
| **What's better** | Full control over URL design, response shape | Zero boilerplate for data services, built-in pagination/filtering |
| **What's worse** | Lots of boilerplate for standard CRUD | Less control over URL and response shape |

> **Key insight:** In CAP, the CDS model is your controller, your DTO, and your API contract all in one. You don't write controllers — you write event handlers for exceptions to the default behavior.

---

## 2. Persistence: JPA + Hibernate → CDS + CQN

### Spring Boot Approach

```java
// Entity
@Entity
@Table(name = "books")
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false, length = 200)
    private String title;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private Author author;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    private Integer stock;
}

// Repository
public interface BookRepository extends JpaRepository<Book, UUID> {
    List<Book> findByAuthorName(String authorName);

    @Query("SELECT b FROM Book b WHERE b.stock > 0 AND b.price < :maxPrice")
    List<Book> findAvailableUnder(@Param("maxPrice") BigDecimal maxPrice);
}

// Usage
List<Book> books = bookRepository.findAvailableUnder(new BigDecimal("50.00"));
```

### CAP Equivalent

```cds
// CDS model (replaces @Entity + @Table + @Column)
entity Books {
    key ID    : UUID;
    title     : String(200) @mandatory;
    author    : Association to Authors;
    price     : Decimal(10,2);
    stock     : Integer;
}
```

```java
// CQN queries (replaces Repository + @Query)
@Autowired
private PersistenceService db;

// Simple select
Result books = db.run(Select.from(Books_.class));

// Filtered query (equivalent to custom @Query)
CqnSelect query = Select.from(Books_.class)
    .where(b -> b.stock().gt(0)
        .and(b.price().lt(new BigDecimal("50.00"))));
List<Books> available = db.run(query).listOf(Books.class);

// Join via association (equivalent to findByAuthorName)
CqnSelect byAuthor = Select.from(Books_.class)
    .where(b -> b.author().name().eq("Robert Martin"));
List<Books> authorBooks = db.run(byAuthor).listOf(Books.class);

// Expand (equivalent to EAGER fetch)
CqnSelect withAuthor = Select.from(Books_.class)
    .columns(b -> b._all(), b -> b.author().expand());
```

### Comparison

| Aspect | JPA + Hibernate | CDS + CQN |
|---|---|---|
| Model definition | Java classes with annotations | CDS schema files |
| Query language | JPQL / Criteria API / Spring Data methods | CQN (fluent Java API) |
| N+1 problem | Requires `@EntityGraph` or `JOIN FETCH` | Use `.expand()` — explicit, no lazy loading traps |
| Session / cache | L1 cache (session), L2 cache (optional) | No session, no cache — stateless by design |
| Lazy loading | Default for `@ManyToOne` | No lazy loading — you get what you ask for |
| Schema generation | `hibernate.ddl-auto` or Flyway/Liquibase | HDI deployer (HANA) or auto-DDL (H2/PostgreSQL) |
| Transactions | `@Transactional` | CAP's `ChangeSetContext` (one per request) |
| Database dialects | Hibernate dialect abstraction | CDS compiles to target DB SQL |
| **What's better** | Rich ORM features, detached entities, L2 cache | No lazy loading surprises, stateless, simpler mental model |
| **What's worse** | N+1 traps, session management complexity | No entity manager, no detached entities, less flexible for complex queries |

### Transaction Management

```java
// Spring Boot
@Transactional
public void transferStock(UUID fromId, UUID toId, int quantity) {
    Book from = bookRepository.findById(fromId).orElseThrow();
    Book to = bookRepository.findById(toId).orElseThrow();
    from.setStock(from.getStock() - quantity);
    to.setStock(to.getStock() + quantity);
    // Hibernate auto-flushes dirty entities at commit
}

// CAP Java — transaction is per-request by default
@On(event = "transferStock")
public void onTransferStock(TransferStockContext context) {
    CqnSelect selectFrom = Select.from(Books_.class).byId(context.getFromId());
    Books from = db.run(selectFrom).single(Books.class);

    CqnSelect selectTo = Select.from(Books_.class).byId(context.getToId());
    Books to = db.run(selectTo).single(Books.class);

    db.run(Update.entity(Books_.class)
        .data("stock", from.getStock() - context.getQuantity())
        .byId(context.getFromId()));

    db.run(Update.entity(Books_.class)
        .data("stock", to.getStock() + context.getQuantity())
        .byId(context.getToId()));
    // All updates commit together at the end of the request/changeset
}
```

### Schema Migration: Flyway/Liquibase → HDI Deployer

| Aspect | Flyway / Liquibase | HDI Deployer |
|---|---|---|
| Migration files | SQL scripts (V1__create.sql) | CDS model is the source of truth |
| Approach | Imperative (you write ALTER TABLE) | Declarative (CDS compiler diffs) |
| Rollback | Explicit rollback scripts | No built-in rollback — forward only |
| Version tracking | Schema version table | HDI container metadata |
| Data migration | SQL migration scripts | Not handled — separate task |
| **What's better** | Explicit control, rollback support | No migration scripts to maintain |
| **What's worse** | Manual migration writing | No rollback, risky for breaking changes |

---

## 3. Security: Spring Security → XSUAA + @restrict

### Spring Boot Approach

```java
// SecurityConfig.java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/books/**").hasAnyRole("VIEWER", "ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
        return http.build();
    }
}

// Controller-level
@PreAuthorize("hasAuthority('SCOPE_Admin')")
@DeleteMapping("/{id}")
public void delete(@PathVariable UUID id) {
    bookRepository.deleteById(id);
}
```

### CAP Equivalent

```cds
// CDS-level authorization (replaces SecurityConfig + @PreAuthorize)
service CatalogService @(path: '/catalog') {

    @readonly
    entity Books as projection on db.Books;

    @restrict: [
        { grant: 'READ',   to: 'Viewer' },
        { grant: 'WRITE',  to: 'Admin' },
        { grant: 'DELETE', to: 'Admin' }
    ]
    entity Authors as projection on db.Authors;

    // Instance-based authorization (row-level security)
    @restrict: [{
        grant: '*',
        to: 'RegionalManager',
        where: 'region = $user.region'
    }]
    entity Orders as projection on db.Orders;
}
```

```json
// xs-security.json (replaces OAuth2 client registration)
{
  "xsappname": "my-cap-app",
  "tenant-mode": "dedicated",
  "scopes": [
    { "name": "$XSAPPNAME.Viewer" },
    { "name": "$XSAPPNAME.Admin" }
  ],
  "role-templates": [
    { "name": "Viewer", "scope-references": ["$XSAPPNAME.Viewer"] },
    { "name": "Admin",  "scope-references": ["$XSAPPNAME.Admin", "$XSAPPNAME.Viewer"] }
  ]
}
```

### Comparison

| Aspect | Spring Security | CAP @restrict + XSUAA |
|---|---|---|
| Auth definition | Java config + annotations | CDS annotations + xs-security.json |
| Granularity | URL-pattern or method-level | Entity + event-level (READ, WRITE, DELETE) |
| Row-level security | Custom `@PostFilter` or SQL | Built-in `where` clause in `@restrict` |
| User attributes | From JWT claims via `@AuthenticationPrincipal` | `$user.<attribute>` in CDS where-clauses |
| OAuth2 flows | Spring Security OAuth2 Resource Server | XSUAA (same OAuth2, BTP-specific) |
| **What's better** | More flexible, middleware-like approach | Declarative, model-level, instance-based auth built-in |
| **What's worse** | Row-level security is manual | Less flexible for complex auth logic |

### Custom Authorization Logic

```java
// Spring Boot — custom voter or @PreAuthorize SpEL
@PreAuthorize("@authService.canAccessOrder(#id, authentication)")
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable UUID id) { ... }

// CAP Java — @Before handler for custom auth
@Before(event = CqnService.EVENT_READ, entity = Orders_.CDS_NAME)
public void checkOrderAccess(CdsReadEventContext context) {
    UserInfo user = context.getUserInfo();
    if (!user.hasRole("Admin") && !user.hasRole("RegionalManager")) {
        // Check if user has custom attribute
        String userRegion = user.getAttributeValues("region").stream()
            .findFirst().orElse(null);
        if (userRegion == null) {
            throw new ServiceException(ErrorStatuses.FORBIDDEN, "No region assigned");
        }
        // CAP's @restrict where-clause handles the rest
    }
}
```

---

## 4. Configuration, Messaging & Profiles

### Spring Cloud Config → BTP Destination Service

| Spring Cloud | BTP Equivalent |
|---|---|
| Config Server (Git-backed) | BTP Destination Service + environment variables |
| `@Value("${api.url}")` | `DestinationAccessor.getDestination("API")` |
| `bootstrap.yml` + profiles | `mta.yaml` parameters + service bindings |
| Consul / Vault | SAP Credential Store |
| `@RefreshScope` | Destination cache TTL (auto-refreshes) |

```java
// Spring Boot — config-driven URL
@Value("${external.api.url}")
private String apiUrl;

@Autowired
private RestTemplate restTemplate;

public Data fetchData() {
    return restTemplate.getForObject(apiUrl + "/data", Data.class);
}

// CAP / BTP — destination-driven URL
public Data fetchData() {
    HttpDestination dest = DestinationAccessor
        .getDestination("EXTERNAL_API")
        .asHttp();
    // URL, auth, proxy type — all configured in BTP Cockpit, not in code
    HttpClient client = HttpClientAccessor.getHttpClient(dest);
    HttpGet request = new HttpGet("/data");
    return client.execute(request, responseHandler);
}
```

### Spring Cloud Stream → CAP Messaging + Event Mesh

```java
// Spring Cloud Stream (Kafka)
@Bean
public Consumer<OrderEvent> processOrder() {
    return event -> {
        log.info("Received order: {}", event.getOrderId());
        orderService.process(event);
    };
}

// application.yml
spring:
  cloud:
    stream:
      bindings:
        processOrder-in-0:
          destination: orders
          group: order-processors
      kafka:
        binder:
          brokers: localhost:9092
```

```java
// CAP Messaging (Event Mesh)
@Component
@ServiceName("messaging")
public class OrderEventHandler implements EventHandler {

    @On(event = "OrderCreated", service = "messaging")
    public void onOrderCreated(EventContext context) {
        Map<String, Object> data = context.get("data");
        log.info("Received order: {}", data.get("orderId"));
        processOrder(data);
    }
}
```

```yaml
# application.yaml (CAP messaging config)
cds:
  messaging:
    services:
      messaging:
        kind: event-mesh
        subscribePrefix: sap/s4/beh
        publishPrefix: my/app
```

| Aspect | Spring Cloud Stream | CAP Messaging |
|---|---|---|
| Broker | Kafka, RabbitMQ, etc. | SAP Event Mesh (or Kafka via config) |
| Binding | Functional bean binding | `@On` event handlers |
| Message format | Custom POJOs | CloudEvents specification |
| Consumer groups | `group` property | Event Mesh queue subscriptions |
| Error handling | DLQ, retry via binder | Event Mesh dead-letter, manual retry |
| **What's better** | Broker-agnostic, rich binding options | CloudEvents standard, S/4HANA integration built-in |
| **What's worse** | No built-in S/4HANA event types | Tied to Event Mesh (or limited Kafka support) |

### Spring Profiles → MTA Descriptors + Helm Values

```
┌──────────────────────────────────────────────────────────────┐
│  Spring Boot                     │  CAP on BTP               │
├──────────────────────────────────┼───────────────────────────┤
│  application-dev.yml             │  mta.yaml (dev space)     │
│  application-staging.yml         │  values-staging.yaml      │
│  application-prod.yml            │  values-prod.yaml         │
│                                  │                           │
│  SPRING_PROFILES_ACTIVE=prod     │  CF: different spaces     │
│                                  │  Kyma: different          │
│                                  │    namespaces + values    │
│                                  │                           │
│  @Profile("dev")                 │  CDS profiles:            │
│  @Bean                           │  [development]            │
│  DataSource devDataSource()      │  [production]             │
│                                  │  in .cdsrc.json           │
└──────────────────────────────────┴───────────────────────────┘
```

```json
// .cdsrc.json — CDS profiles
{
  "[development]": {
    "requires": {
      "db": { "kind": "sql" }
    }
  },
  "[production]": {
    "requires": {
      "db": { "kind": "hana" }
    }
  }
}
```

---

## 5. Testing & Observability

### Testing: Spring Boot Test → CAP Testing

```java
// Spring Boot integration test
@SpringBootTest
@AutoConfigureMockMvc
class BookControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean BookRepository bookRepository;

    @Test
    void shouldReturnBooks() throws Exception {
        when(bookRepository.findAll()).thenReturn(List.of(testBook()));

        mockMvc.perform(get("/api/books"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$[0].title").value("Clean Code"));
    }
}

// CAP Java integration test — same Spring Boot test infra!
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class CatalogServiceTest {

    @Autowired MockMvc mockMvc;

    @Test
    void shouldReturnBooks() throws Exception {
        // CAP uses in-memory H2 with CSV test data automatically
        mockMvc.perform(get("/odata/v4/CatalogService/Books")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.value[0].title").exists());
    }

    @Test
    void shouldExpandAuthor() throws Exception {
        mockMvc.perform(get("/odata/v4/CatalogService/Books?$expand=author")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.value[0].author.name").exists());
    }
}
```

| Aspect | Spring Boot Testing | CAP Testing |
|---|---|---|
| Framework | JUnit 5 + Spring Boot Test | JUnit 5 + Spring Boot Test (same!) |
| In-memory DB | H2 (via `@DataJpaTest`) | H2 (via CDS auto-config) |
| Test data | SQL scripts or `@Sql` | CSV files in `db/data/` folder |
| Mocking services | `@MockBean` | `@MockBean` or CDS mock services |
| API testing | `MockMvc` | `MockMvc` (same!) |
| Testcontainers | PostgreSQL, MySQL | HANA Cloud or PostgreSQL |
| **What's better** | Same — CAP runs on Spring Boot | Built-in CSV test data loading |
| **What's worse** | — | OData URL syntax more complex for assertions |

### Observability Mapping

| Spring Boot | CAP / BTP Equivalent |
|---|---|
| `spring-boot-starter-actuator` | Same (CAP includes it) |
| Micrometer + Prometheus | Same (works identically) |
| Sleuth / Zipkin | OpenTelemetry + SAP Cloud Logging |
| Spring Boot Admin | SAP Cloud ALM |
| Logback + console | `cf-java-logging-support` (structured JSON) |
| `@Timed`, `@Counted` | Same Micrometer annotations |
| `HealthIndicator` | Same Spring interface |

The key difference: SAP's `cf-java-logging-support` library adds BTP-specific fields (tenant ID, correlation ID, component name) to log entries and is required for proper integration with SAP Cloud Logging Service.

---

## Migration Decision Matrix

```
┌──────────────────────────────────────────────────────────────────┐
│                  Should You Migrate to CAP?                       │
│                                                                   │
│  Building CRUD-heavy data services?                               │
│  ├── Yes → CAP saves massive boilerplate                          │
│  └── No                                                           │
│      ├── Complex algorithmic backend? → Stay with Spring Boot     │
│      └── Mixed workload?                                          │
│          └── CAP for data services + Spring beans for logic       │
│                                                                   │
│  Need OData V4 with Fiori Elements?                               │
│  ├── Yes → CAP is the only practical choice                       │
│  └── No → Spring Boot REST is fine                                │
│                                                                   │
│  Multi-tenant SaaS on BTP?                                        │
│  ├── Yes → CAP's MTX sidecar saves months of work                 │
│  └── No → Either works                                            │
│                                                                   │
│  Team knows only Spring Boot?                                     │
│  ├── Good news: CAP runs ON Spring Boot                           │
│  └── Learning curve: CDS language + CQN API + OData conventions   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Top 5 Pitfalls

1. **Trying to use Spring Data JPA alongside CAP.** CAP has its own persistence layer. Mixing both causes transaction conflicts, dual schema management, and authorization bypass (CAP's `@restrict` doesn't apply to JPA queries).
2. **Writing REST controllers instead of CDS actions.** You can add `@RestController` beans to a CAP app, but they bypass CAP's authorization, audit logging, and protocol-agnostic dispatch. Use CDS actions/functions instead.
3. **Expecting Hibernate-style lazy loading.** CQN is explicit — you get exactly what you query. Use `.expand()` for associations. There is no session, no proxy objects, no `LazyInitializationException` (which is actually a good thing).
4. **Hardcoding URLs instead of using Destinations.** On BTP, all external URLs should go through the Destination Service. This enables central credential management, Cloud Connector routing, and principal propagation.
5. **Ignoring CDS as a modeling language.** Developers who skip CDS and try to define everything in Java miss CAP's core value: one model driving database schema, API contract, authorization rules, and UI annotations simultaneously.

---

## What to Learn Next

- **Lesson 3.1:** CAP Java Architecture — deep dive into CDS, CQN, and the handler model
- **Lesson 3.2:** CAP Security — `@restrict` annotations in detail
- **Lesson 1.1:** BTP Architecture — understanding XSUAA, destinations, and connectivity
- **Lesson 4.1:** CI/CD for BTP — how MTA builds replace Maven-only pipelines