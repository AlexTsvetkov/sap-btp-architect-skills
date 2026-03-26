# Lesson 1.2 — SAP HANA Cloud Deep Dive

## Table of Contents

- [1. HANA Cloud vs On-Premise HANA](#1-hana-cloud-vs-on-premise-hana)
- [2. HANA Multi-Model Engine](#2-hana-multi-model-engine)
- [3. HDI (HANA Deployment Infrastructure)](#3-hdi-hana-deployment-infrastructure)
- [4. Connectivity from CF / Kyma Applications](#4-connectivity-from-cf-kyma-applications)
- [5. Performance Tuning](#5-performance-tuning)
- [Comparison with PostgreSQL / MySQL](#comparison-with-postgresql-mysql)
- [Top 5 Pitfalls](#top-5-pitfalls)
- [What to Learn Next](#what-to-learn-next)

---

> **Summary:** SAP HANA Cloud is a columnar, in-memory relational database offered as a managed service on BTP. For Java architects coming from PostgreSQL or MySQL, the paradigm shift is significant: HANA's column store, multi-model engines (document, graph, spatial), and HDI deployment model all require new mental models. This lesson covers HANA Cloud internals, the HDI container architecture, connectivity patterns from CF/Kyma, and performance tuning strategies — all compared against familiar RDBMS concepts.

---

## 1. HANA Cloud vs On-Premise HANA

### Architecture Differences

```
┌─────────────────────────────────────────────────────┐
│                 SAP HANA Cloud                       │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────┐ │
│  │ Column   │  │ Row      │  │ Native Storage    │ │
│  │ Store    │  │ Store    │  │ Extension (NSE)   │ │
│  │ (hot)    │  │ (OLTP)   │  │ (warm/cold data)  │ │
│  │ in-memory│  │ in-memory│  │ disk-based pages  │ │
│  └──────────┘  └──────────┘  └───────────────────┘ │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │         Persistence Layer (disk)              │   │
│  │  - Data volumes, log volumes                  │   │
│  │  - Page-loadable columns (NSE)                │   │
│  │  - Automatic tiering                          │   │
│  └──────────────────────────────────────────────┘   │
│                                                     │
│  Managed by SAP: backups, patching, HA, scaling     │
└─────────────────────────────────────────────────────┘
```

| Feature | On-Premise HANA | HANA Cloud |
|---|---|---|
| Memory model | All data in memory (column store) | Memory + disk (NSE for warm data) |
| Scaling | Scale-up only (add RAM) | Elastic compute scaling, storage auto-grows |
| Administration | Full DBA access, OS-level | No OS access, limited admin via SQL/Cockpit |
| Multi-tenancy | MDC (Multitenant Database Containers) | Managed by SAP, one instance per service |
| Backup/Recovery | Manual or scripted | Automatic, point-in-time recovery via Cockpit |
| Network | Direct TCP access | TLS-only, access via service binding credentials |
| Cost model | License + hardware | Consumption-based (CU — Compute Units) |

### Native Storage Extension (NSE)

NSE is HANA Cloud's game-changer for cost optimization. It allows you to mark columns or entire tables as **page-loadable** instead of **column-loadable**:

```sql
-- Column-loadable (default): entire column loaded into memory
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    amount DECIMAL(15,2),
    created_at TIMESTAMP
);

-- Page-loadable: only accessed pages loaded into memory buffer cache
ALTER TABLE orders ALTER (amount DECIMAL(15,2) PAGE LOADABLE);

-- Or make entire table page-loadable
ALTER TABLE orders PAGE LOADABLE;
```

**When to use NSE:**
- Historical data that is rarely queried
- Large text/LOB columns
- Audit tables
- Tables where the hot working set is < 20% of total data

**Analogy for PostgreSQL devs:** NSE is conceptually similar to PostgreSQL's TOAST mechanism combined with buffer pool management — data lives on disk and is paged into memory on demand. The key difference is that HANA's column store still provides columnar compression benefits even for page-loadable data.

---

## 2. HANA Multi-Model Engine

HANA is not just a relational database. It includes five distinct processing engines:

### Column Store (Primary)

The column store is HANA's default and strongest engine. Data is stored column-by-column with dictionary encoding and various compression schemes:

```
Traditional Row Store (PostgreSQL/MySQL):
Row 1: [id:1, name:"Alice", city:"Berlin", age:30]
Row 2: [id:2, name:"Bob",   city:"Berlin", age:25]
Row 3: [id:3, name:"Carol", city:"Munich", age:35]

HANA Column Store:
id column:   [1, 2, 3]
name column: [dict: 0=Alice, 1=Bob, 2=Carol] → [0, 1, 2]
city column: [dict: 0=Berlin, 1=Munich] → [0, 0, 1]  ← high compression
age column:  [30, 25, 35]
```

Benefits:
- **Compression:** Dictionary encoding, run-length encoding, cluster encoding. Typical 5-10x compression ratio.
- **Vectorized processing:** Column scans process data in CPU-cache-friendly chunks.
- **Aggregation speed:** `SELECT city, AVG(age) FROM users GROUP BY city` only reads `city` and `age` columns, skipping all others.

```sql
-- Check compression ratio
SELECT TABLE_NAME, 
       ROUND(MEMORY_SIZE_IN_TOTAL / 1024 / 1024, 2) AS memory_mb,
       ROUND(DISK_SIZE / 1024 / 1024, 2) AS disk_mb,
       RECORD_COUNT
FROM M_CS_TABLES
WHERE SCHEMA_NAME = 'MY_SCHEMA'
ORDER BY MEMORY_SIZE_IN_TOTAL DESC;
```

### Row Store

Used for:
- System tables (internal catalog)
- Tables with frequent single-row updates (point lookups by PK)
- Temporary tables in stored procedures

```sql
-- Explicitly create a row-store table (rarely needed)
CREATE ROW TABLE session_cache (
    session_id VARCHAR(128) PRIMARY KEY,
    user_id BIGINT,
    data NCLOB,
    expires_at TIMESTAMP
);
```

> **Rule of thumb:** Use column store for everything unless you have a specific reason not to. HANA's column store handles OLTP workloads well due to its delta merge architecture.

### Document Store (JSON Collections)

HANA includes a native JSON document store — useful for schema-less data:

```sql
-- Create a JSON collection
CREATE COLLECTION products;

-- Insert documents
INSERT INTO products VALUES ('{"name": "Laptop", "specs": {"ram": 16, "cpu": "i7"}, "tags": ["electronics", "computing"]}');
INSERT INTO products VALUES ('{"name": "Phone", "specs": {"ram": 8, "cpu": "A15"}, "tags": ["electronics", "mobile"]}');

-- Query with JSON path expressions
SELECT * FROM products 
WHERE "specs"."ram" > 10;

-- Join collections with relational tables
SELECT p.*, o.order_date
FROM products p
JOIN orders o ON p."_id" = o.product_id;
```

**Comparison with MongoDB:** HANA's document store supports basic CRUD and querying but lacks MongoDB's aggregation pipeline, change streams, and sharding. Use it for auxiliary semi-structured data, not as a primary document database.

### Graph Engine

For traversal-heavy queries (org charts, supply chains, social networks):

```sql
-- Create a graph workspace
CREATE GRAPH WORKSPACE my_graph
    EDGE TABLE edges
        SOURCE COLUMN source_id
        TARGET COLUMN target_id
        KEY COLUMN edge_id
    VERTEX TABLE vertices
        KEY COLUMN vertex_id;

-- Shortest path query
SELECT * FROM SHORTEST_PATH(
    my_graph,
    vertex_start => 'A',
    vertex_end => 'Z',
    direction => 'OUTGOING'
);

-- Pattern matching (like Cypher in Neo4j)
MATCH (a)-[e]->(b)-[f]->(c)
WHERE a.name = 'SAP' AND c.type = 'Subsidiary'
RETURN a.name, b.name, c.name;
```

### Spatial Engine

For geographic data (store locations, delivery zones, geofencing):

```sql
CREATE TABLE stores (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    location ST_POINT(4326)  -- SRID 4326 = WGS84
);

INSERT INTO stores VALUES (1, 'Berlin Store', ST_GeomFromText('POINT(13.405 52.52)', 4326));
INSERT INTO stores VALUES (2, 'Munich Store', ST_GeomFromText('POINT(11.582 48.135)', 4326));

-- Find stores within 100km of a point
SELECT name, location.ST_Distance(ST_GeomFromText('POINT(13.0 52.5)', 4326)) AS distance_m
FROM stores
WHERE location.ST_Distance(ST_GeomFromText('POINT(13.0 52.5)', 4326)) < 100000;
```

---

## 3. HDI (HANA Deployment Infrastructure)

### Container Model

HDI is the deployment mechanism for HANA Cloud artifacts. It manages database objects through a design-time/runtime separation:

```
┌─────────────────────────────────────────────┐
│            HANA Cloud Instance               │
│                                             │
│  ┌─────────────┐  ┌─────────────┐          │
│  │ HDI Container│  │ HDI Container│  ...    │
│  │ (App A)      │  │ (App B)      │         │
│  │              │  │              │         │
│  │ Schema:      │  │ Schema:      │         │
│  │ APP_A_HDI_1  │  │ APP_B_HDI_1  │         │
│  │              │  │              │         │
│  │ Design-time: │  │ Design-time: │         │
│  │ .hdbcds      │  │ .hdbtable    │         │
│  │ .hdbview     │  │ .hdbcalcview │         │
│  │ .hdbrole     │  │ .hdbgrants   │         │
│  │              │  │              │         │
│  │ Runtime:     │  │ Runtime:     │         │
│  │ Tables       │  │ Tables       │         │
│  │ Views        │  │ Views        │         │
│  │ Procedures   │  │ Calc Views   │         │
│  └─────────────┘  └─────────────┘          │
└─────────────────────────────────────────────┘
```

Key concepts:
- Each HDI container maps to a **database schema** with its own isolated namespace
- Design-time artifacts (`.hdb*` files) are **deployed** by the HDI deployer, which generates runtime objects
- The container owner has DML access; a separate technical user has DDL access
- **Schema evolution** is automatic: the HDI deployer computes the delta between design-time artifacts and current runtime state

### Design-Time Artifacts

| Artifact | Extension | Purpose |
|---|---|---|
| CDS entities | `.hdbcds` | Define tables and views using CDS syntax |
| Tables | `.hdbtable` | Define tables using HANA SQL DDL |
| Views | `.hdbview` | SQL views |
| Calculation views | `.hdbcalculationview` | Graphical multi-level analytical views |
| Procedures | `.hdbprocedure` | Stored procedures (SQLScript) |
| Table types | `.hdbtabletype` | Reusable table type definitions |
| Roles | `.hdbrole` | Container-level roles |
| Grants | `.hdbgrants` | Cross-container access grants |
| Synonyms | `.hdbsynonym` | Access objects outside the container |

### CDS vs hdbtable

```sql
-- .hdbcds syntax (SAP's CDS for HANA, not CAP CDS)
context my.model {
    entity Orders {
        key id       : Integer;
        customer     : String(100);
        amount       : Decimal(15, 2);
        createdAt    : UTCTimestamp;
    };
};

-- Equivalent .hdbtable syntax
COLUMN TABLE "my.model::Orders" (
    "id" INTEGER NOT NULL,
    "customer" NVARCHAR(100),
    "amount" DECIMAL(15, 2),
    "createdAt" TIMESTAMP,
    PRIMARY KEY ("id")
);
```

> **Important distinction:** HDI `.hdbcds` is HANA-native CDS syntax. CAP's CDS (`.cds` files) is a *different* language that compiles *down to* `.hdbcds` or `.hdbtable` artifacts for HANA deployment.

### HDI in Multi-Tenancy

In multi-tenant CAP applications, each tenant gets its own HDI container:

```
Tenant "customer-a" → HDI Container → Schema: CUSTOMER_A_HDI_1
Tenant "customer-b" → HDI Container → Schema: CUSTOMER_B_HDI_1
Tenant "customer-c" → HDI Container → Schema: CUSTOMER_C_HDI_1
```

The `@sap/cds-mtxs` sidecar manages container provisioning and schema upgrades across all tenants.

---

## 4. Connectivity from CF / Kyma Applications

### Service Binding Anatomy

When you bind a HANA Cloud HDI service instance, you receive credentials:

```json
{
  "credentials": {
    "host": "abc123.hana.prod-eu10.hanacloud.ondemand.com",
    "port": "443",
    "driver": "com.sap.db.jdbc.Driver",
    "url": "jdbc:sap://abc123.hana.prod-eu10.hanacloud.ondemand.com:443?encrypt=true&validateCertificate=true",
    "schema": "MY_HDI_CONTAINER",
    "hdi_user": "MY_HDI_CONTAINER#OO",
    "hdi_password": "...",
    "user": "MY_HDI_CONTAINER#OO",
    "password": "...",
    "certificate": "-----BEGIN CERTIFICATE-----\n..."
  }
}
```

### JDBC Connection (Java)

```java
// Direct JDBC (not recommended for CAP apps)
import com.sap.db.jdbc.Driver;

String url = credentials.getString("url");
String user = credentials.getString("user");
String password = credentials.getString("password");

Connection conn = DriverManager.getConnection(url, user, password);
```

### CAP's Built-In Persistence Layer

CAP Java abstracts HANA connectivity entirely. You define CDS models and CAP generates SQL:

```cds
// db/schema.cds
namespace my.bookshop;

entity Books {
    key ID    : UUID;
    title     : String(200);
    author    : Association to Authors;
    price     : Decimal(10, 2);
    stock     : Integer;
}

entity Authors {
    key ID    : UUID;
    name      : String(100);
    books     : Association to many Books on books.author = $self;
}
```

CAP compiles this to HDI artifacts and handles connection pooling, schema resolution, and multi-tenancy automatically. The CQN query engine translates programmatic queries to HANA SQL:

```java
// CAP Java handler — no raw SQL needed
@On(event = CqnService.EVENT_READ, entity = "my.bookshop.Books")
public void onReadBooks(CdsReadEventContext ctx) {
    CqnSelect query = Select.from("my.bookshop.Books")
        .columns(b -> b.title(), b -> b.price(), b -> b.author().expand(a -> a.name()))
        .where(b -> b.stock().gt(0))
        .orderBy(b -> b.title().asc())
        .limit(20);
    ctx.setResult(db.run(query));
}
```

Generated SQL:

```sql
SELECT T0."TITLE", T0."PRICE", T1."NAME" AS "author_name"
FROM "MY_BOOKSHOP_BOOKS" T0
LEFT JOIN "MY_BOOKSHOP_AUTHORS" T1 ON T0."AUTHOR_ID" = T1."ID"
WHERE T0."STOCK" > 0
ORDER BY T0."TITLE" ASC
LIMIT 20
```

### Connection from Kyma

In Kyma, use the BTP Operator to create a ServiceBinding that injects HANA credentials:

```yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceInstance
metadata:
  name: hana-hdi
spec:
  serviceOfferingName: hana
  servicePlanName: hdi-shared
---
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: hana-hdi-binding
spec:
  serviceInstanceName: hana-hdi
  secretName: hana-hdi-secret
```

Mount the secret into your pod:

```yaml
volumes:
  - name: hana-credentials
    secret:
      secretName: hana-hdi-secret
containers:
  - name: app
    volumeMounts:
      - name: hana-credentials
        mountPath: /etc/secrets/sapcp/hana/hdi
        readOnly: true
```

---

## 5. Performance Tuning

### Explain Plans

```sql
-- Text-based explain plan
EXPLAIN PLAN FOR
SELECT customer_id, SUM(amount) 
FROM orders 
WHERE created_at > '2024-01-01'
GROUP BY customer_id;

SELECT * FROM EXPLAIN_PLAN_TABLE;
```

Key operators to watch:
- `COLUMN SEARCH` — efficient columnar scan
- `COLUMN TABLE FILTER` — predicate pushdown to column engine
- `HASH JOIN` / `MERGE JOIN` — join strategies
- `ROW TABLE SCAN` — row store access (potential bottleneck for analytics)
- `CPEXTRACT` — extraction from column store to row processing (bad sign for large datasets)

### Delta Merge

HANA's column store uses a write-optimized **delta store** and a read-optimized **main store**:

```
Write → Delta Store (row-oriented, small, fast inserts)
              │
              ▼ (async merge)
         Main Store (column-oriented, compressed, fast reads)
```

Monitor delta merge:

```sql
-- Check tables with large delta stores
SELECT TABLE_NAME, 
       RECORD_COUNT,
       RAW_RECORD_COUNT_IN_DELTA,
       ROUND(RAW_RECORD_COUNT_IN_DELTA * 100.0 / NULLIF(RECORD_COUNT, 0), 2) AS delta_pct
FROM M_CS_TABLES
WHERE SCHEMA_NAME = 'MY_SCHEMA'
  AND RAW_RECORD_COUNT_IN_DELTA > 10000
ORDER BY RAW_RECORD_COUNT_IN_DELTA DESC;

-- Trigger manual merge if needed
MERGE DELTA OF "MY_SCHEMA"."MY_TABLE";
```

### Partition Pruning

For large tables, partitioning enables the query engine to skip irrelevant data:

```sql
-- Range partitioning by date
CREATE TABLE events (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    event_type VARCHAR(50),
    payload NCLOB,
    created_at DATE NOT NULL
)
PARTITION BY RANGE (created_at) (
    PARTITION '2024-01-01' <= VALUES < '2024-04-01',
    PARTITION '2024-04-01' <= VALUES < '2024-07-01',
    PARTITION '2024-07-01' <= VALUES < '2024-10-01',
    PARTITION '2024-10-01' <= VALUES < '2025-01-01',
    PARTITION OTHERS
);

-- Hash partitioning for even distribution
CREATE TABLE user_events (
    user_id BIGINT,
    event_data NVARCHAR(5000)
)
PARTITION BY HASH (user_id) PARTITIONS 8;
```

### Common Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Row-by-row processing in loops | Bypasses columnar vectorization | Use set-based SQL or array processing |
| `SELECT *` on wide tables | Loads all columns from column store | Select only needed columns |
| Overusing calculation views | Layer upon layer of calc views creates complex execution plans | Flatten where possible, use SQL views for simple projections |
| Missing partition pruning | Full table scans on partitioned tables | Ensure WHERE clauses include the partition column |
| Excessive use of row store | Row store doesn't benefit from columnar compression | Move to column store unless OLTP pattern requires row store |
| Not monitoring delta merge | Large delta stores degrade read performance | Set up monitoring, tune auto-merge thresholds |

### Memory Consumption Analysis

```sql
-- Top memory consumers
SELECT SCHEMA_NAME, TABLE_NAME, TABLE_TYPE,
       ROUND(MEMORY_SIZE_IN_TOTAL / 1024 / 1024, 2) AS total_mb,
       ROUND(MEMORY_SIZE_IN_MAIN / 1024 / 1024, 2) AS main_mb,
       ROUND(MEMORY_SIZE_IN_DELTA / 1024 / 1024, 2) AS delta_mb,
       RECORD_COUNT
FROM M_CS_TABLES
ORDER BY MEMORY_SIZE_IN_TOTAL DESC
LIMIT 20;

-- Memory by column for a specific table
SELECT COLUMN_NAME, 
       ROUND(MEMORY_SIZE_IN_MAIN / 1024 / 1024, 2) AS main_mb,
       COMPRESSION_TYPE,
       DISTINCT_COUNT
FROM M_CS_COLUMNS
WHERE SCHEMA_NAME = 'MY_SCHEMA' AND TABLE_NAME = 'MY_TABLE'
ORDER BY MEMORY_SIZE_IN_MAIN DESC;
```

---

## Comparison with PostgreSQL / MySQL

| Concept | PostgreSQL/MySQL | HANA Cloud |
|---|---|---|
| Default storage | Row-oriented (heap) | Column-oriented |
| In-memory | Buffer pool caches pages | All active data in memory (column store) or paged (NSE) |
| Compression | pg: TOAST, InnoDB: page compression | Dictionary encoding, RLE, cluster encoding (automatic) |
| Schema migration | Flyway / Liquibase | HDI deployer (declarative) |
| JSON support | `jsonb` type (PostgreSQL) | Document store (collections) |
| Graph queries | Recursive CTEs or Apache AGE | Native graph engine |
| Spatial | PostGIS extension | Built-in spatial engine |
| Execution plans | `EXPLAIN ANALYZE` | `EXPLAIN PLAN FOR` + `EXPLAIN_PLAN_TABLE` |
| Connection pooling | PgBouncer / HikariCP | HikariCP (JDBC) / CAP internal pool |
| Full-text search | `tsvector` / `tsquery` | Built-in full-text indexes |

---

## Top 5 Pitfalls

1. **Treating HANA like PostgreSQL.** Don't write row-by-row processing loops. HANA's column store excels at set-based operations.
2. **Ignoring NSE.** Keeping cold data in-memory wastes expensive compute units. Use page-loadable for historical data.
3. **Bypassing HDI.** Creating objects directly via SQL (not through `.hdb*` artifacts) means they won't be managed, versioned, or deployable.
4. **Not monitoring delta merge.** A table with 50% data in delta store performs like a row store. Monitor `M_CS_TABLES.RAW_RECORD_COUNT_IN_DELTA`.
5. **Overcomplicating calc views.** Calculation views are powerful but add latency at each layer. For simple joins and projections, SQL views or CDS views are faster and easier to debug.

---

## What to Learn Next

- **Lesson 1.1:** BTP Architecture — how HANA Cloud fits in the BTP service ecosystem
- **Lesson 3.1:** CAP Java core concepts — how CAP abstracts HANA via CDS models and CQN
- **Lesson 3.4:** Advanced CAP patterns — connection pooling, batch processing, deployment topologies
- SAP HANA Cloud SQL Reference Guide — full DDL/DML reference
- SAP HANA Cloud Performance Guide — deep tuning documentation