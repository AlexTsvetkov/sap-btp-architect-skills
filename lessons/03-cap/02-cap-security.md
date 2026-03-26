# Lesson 3.2 — CAP Security, Authorization & Authentication

> **Summary:** CAP Java integrates with SAP XSUAA and IAS for authentication and provides a declarative authorization model via CDS annotations (`@requires`, `@restrict`). This lesson covers the authentication flow, CDS-based authorization, instance-based authorization, programmatic security, multi-tenant security concerns, and testing with mock users.

---

## 1. Authentication Architecture

### Authentication Flow

```
Browser/Client          Approuter           XSUAA              CAP Java
     │                     │                  │                    │
     │  GET /app           │                  │                    │
     │────────────────────→│                  │                    │
     │                     │ redirect to login│                    │
     │←────────────────────│                  │                    │
     │                     │                  │                    │
     │  Login credentials  │                  │                    │
     │─────────────────────┼─────────────────→│                    │
     │                     │                  │ Authenticate       │
     │                     │                  │ Issue JWT           │
     │  JWT token          │                  │                    │
     │←────────────────────┼──────────────────│                    │
     │                     │                  │                    │
     │  GET /odata/v4/catalog/Books           │                    │
     │  Authorization: Bearer <JWT>           │                    │
     │────────────────────────────────────────┼───────────────────→│
     │                     │                  │                    │
     │                     │                  │    Validate JWT    │
     │                     │                  │    Extract roles   │
     │                     │                  │    Check @restrict │
     │                     │                  │                    │
     │  200 OK + data      │                  │                    │
     │←───────────────────────────────────────┼────────────────────│
```

### JWT Claims Used by CAP

```json
{
  "sub": "user123",
  "email": "john@example.com",
  "scope": [
    "my-app!t12345.Viewer",
    "my-app!t12345.Admin"
  ],
  "xs.user.attributes": {
    "Country": ["DE", "US"],
    "CostCenter": ["CC100"]
  },
  "zid": "tenant-zone-id",
  "grant_type": "authorization_code"
}
```

CAP extracts from JWT:
- **Roles** — from `scope` claim (mapped via role templates)
- **User attributes** — from `xs.user.attributes` (for instance-based auth)
- **Tenant** — from `zid` claim (for multi-tenancy)
- **User ID** — from `sub` or `user_name` claim

### Configuration

```yaml
# application.yaml
cds:
  security:
    authentication:
      mode: jwt          # jwt (production) or dummy (development)
    mock:
      users:
        alice:
          password: alice
          roles: [Admin, Viewer]
          attributes:
            Country: [DE]
        bob:
          password: bob
          roles: [Viewer]
          attributes:
            Country: [US]
```

---

## 2. CDS-Based Authorization

### @requires — Role-Based Access to Entire Services/Entities

```cds
// Require authentication for the entire service
service CatalogService @(requires: 'authenticated-user') {

    // Public read, but require Viewer role
    @requires: 'Viewer'
    entity Books as projection on db.Books;

    // Only Admin can manage authors
    @requires: 'Admin'
    entity Authors as projection on db.Authors;
}

// Admin-only service
service AdminService @(requires: 'Admin') {
    entity Config as projection on db.AppConfig;
}
```

### @restrict — Fine-Grained Operation-Level Authorization

```cds
service CatalogService {

    // Different roles for different operations
    @restrict: [
        { grant: 'READ',   to: 'Viewer' },
        { grant: 'WRITE',  to: 'Editor' },
        { grant: '*',      to: 'Admin'  }
    ]
    entity Books as projection on db.Books;

    // Instance-based: users can only see their own orders
    @restrict: [
        { grant: 'READ', to: 'Customer', where: 'createdBy = $user' },
        { grant: '*',    to: 'Admin' }
    ]
    entity Orders as projection on db.Orders;

    // Attribute-based: users see data from their country only
    @restrict: [
        { grant: 'READ', to: 'Viewer', where: 'country = $user.Country' },
        { grant: '*',    to: 'Admin' }
    ]
    entity Suppliers as projection on db.Suppliers;
}
```

### Pseudo-Variables in @restrict

| Variable | Meaning | Source |
|---|---|---|
| `$user` | Current user ID | JWT `sub` claim |
| `$user.<attr>` | User attribute value | JWT `xs.user.attributes` |
| `$now` | Current timestamp | Server clock |
| `$unrestricted` | Bypass all restrictions | Used in privileged service calls |

### xs-security.json Mapping

```json
{
  "xsappname": "my-app",
  "tenant-mode": "dedicated",
  "scopes": [
    { "name": "$XSAPPNAME.Viewer", "description": "Read access" },
    { "name": "$XSAPPNAME.Editor", "description": "Write access" },
    { "name": "$XSAPPNAME.Admin",  "description": "Full access" }
  ],
  "attributes": [
    { "name": "Country", "description": "Country filter", "valueType": "string" },
    { "name": "CostCenter", "description": "Cost center", "valueType": "string" }
  ],
  "role-templates": [
    {
      "name": "Viewer",
      "scope-references": ["$XSAPPNAME.Viewer"],
      "attribute-references": ["Country"]
    },
    {
      "name": "Editor",
      "scope-references": ["$XSAPPNAME.Editor", "$XSAPPNAME.Viewer"],
      "attribute-references": ["Country"]
    },
    {
      "name": "Admin",
      "scope-references": ["$XSAPPNAME.Admin", "$XSAPPNAME.Editor", "$XSAPPNAME.Viewer"]
    }
  ]
}
```

---

## 3. Programmatic Authorization

### Accessing User Info in Handlers

```java
@Component
@ServiceName(CatalogService_.CDS_NAME)
public class CatalogServiceHandler implements EventHandler {

    @Before(event = CqnService.EVENT_CREATE, entity = Orders_.CDS_NAME)
    public void enrichOrder(CdsCreateEventContext context) {
        UserInfo user = context.getUserInfo();

        // Get user ID
        String userId = user.getName();

        // Check roles
        boolean isAdmin = user.hasRole("Admin");

        // Get user attributes
        List<String> countries = user.getAttributeValues("Country");

        // Set audit fields
        context.getCqn().entries().forEach(order -> {
            order.put("createdBy", userId);
            order.put("createdAt", Instant.now());
        });
    }

    @Before(event = CqnService.EVENT_UPDATE, entity = Orders_.CDS_NAME)
    public void checkOwnership(CdsUpdateEventContext context) {
        UserInfo user = context.getUserInfo();
        if (user.hasRole("Admin")) return; // Admin can update anything

        // Check if user owns the order
        String orderId = context.getCqn().entries().get(0).get("ID").toString();
        CqnSelect check = Select.from(Orders_.class)
            .where(o -> o.ID().eq(orderId).and(o.createdBy().eq(user.getName())));

        if (db.run(check).rowCount() == 0) {
            throw new ServiceException(ErrorStatuses.FORBIDDEN,
                "You can only update your own orders");
        }
    }
}
```

### Privileged (Technical) Access

For service-to-service calls or background jobs that need to bypass authorization:

```java
@Autowired
private CqnService catalogService;

public void backgroundJob() {
    // Run with system/privileged user — bypasses @restrict
    RequestContext privileged = RequestContext.CDS_SYSTEM_USER;

    catalogService.run(privileged, ctx -> {
        CqnSelect allOrders = Select.from(Orders_.class);
        return db.run(allOrders);
    });
}
```

---

## 4. Multi-Tenant Security

### Tenant Isolation

In multi-tenant applications, CAP automatically:
1. Extracts tenant ID from JWT `zid` claim
2. Routes database queries to the correct HDI container
3. Prevents cross-tenant data access

```java
@Before(event = "*")
public void logTenantAccess(EventContext context) {
    String tenant = context.getUserInfo().getTenant();
    log.info("Request from tenant: {}", tenant);
    // CAP automatically routes to correct HDI container
    // No manual tenant filtering needed!
}
```

### Cross-Tenant API Calls

If a provider-side job needs to access tenant data:

```java
public void processTenant(String tenantId) {
    RequestContext tenantContext = RequestContext.of(b -> {
        b.tenant(tenantId);
        b.user(SystemUser.of(tenantId));
    });

    service.run(tenantContext, ctx -> {
        // This runs in the context of the specified tenant
        return db.run(Select.from(Orders_.class));
    });
}
```

---

## 5. Testing with Mock Users

### Integration Test Setup

```java
@SpringBootTest
@AutoConfigureMockMvc
class CatalogServiceAuthTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @WithMockUser(username = "alice", roles = {"Viewer"})
    void viewerCanReadBooks() throws Exception {
        mockMvc.perform(get("/odata/v4/catalog/Books"))
            .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(username = "bob", roles = {"Viewer"})
    void viewerCannotCreateBooks() throws Exception {
        mockMvc.perform(post("/odata/v4/catalog/Books")
                .contentType("application/json")
                .content("{\"title\":\"New Book\",\"price\":29.99}"))
            .andExpect(status().isForbidden());
    }

    @Test
    void unauthenticatedCannotAccess() throws Exception {
        mockMvc.perform(get("/odata/v4/catalog/Books"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(username = "admin", roles = {"Admin"})
    void adminCanCreateBooks() throws Exception {
        mockMvc.perform(post("/odata/v4/catalog/Books")
                .contentType("application/json")
                .content("{\"title\":\"New Book\",\"price\":29.99}"))
            .andExpect(status().isCreated());
    }
}
```

### Mock Users in application.yaml (Dev Profile)

```yaml
spring:
  profiles: default  # local development only
cds:
  security:
    authentication:
      mode: dummy     # accepts any credentials
    mock:
      users:
        admin:
          password: admin
          roles: [Admin, Editor, Viewer]
          attributes:
            Country: [DE, US]
        viewer:
          password: viewer
          roles: [Viewer]
          attributes:
            Country: [DE]
```

---

## Top 5 Pitfalls

1. **Using `authentication.mode: dummy` in production.** This disables JWT validation entirely. Always use `jwt` mode in production and ensure XSUAA binding exists.
2. **Confusing scopes and roles.** XSUAA scopes are technical (`my-app!t12345.Viewer`). Role templates map scopes to business roles (`Viewer`). CDS `@requires` uses role template names.
3. **Forgetting instance-based authorization for user-specific data.** `@requires: 'Viewer'` grants access to ALL entities of that type. Use `@restrict` with `where` clauses for row-level filtering.
4. **Not testing authorization in integration tests.** Unit tests often skip security. Use `@WithMockUser` or CAP's mock user configuration to test authorization rules.
5. **Hardcoding tenant checks in handlers.** CAP handles tenant isolation automatically via the persistence layer. Manual `WHERE tenant_id = ?` filters are unnecessary and error-prone.

---

## What to Learn Next

- **Lesson 1.4:** Multi-Tenancy — XSUAA identity zones and tenant-mode
- **Lesson 3.1:** CAP Architecture — handler model and event processing
- **Lesson 3.3:** CAP Remote Services — securing service-to-service communication
- **Lesson 1.1:** BTP Architecture — XSUAA and trust configuration