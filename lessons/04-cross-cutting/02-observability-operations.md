# Lesson 4.2 — Observability & Operations on BTP

## Table of Contents

- [1. SAP Cloud Logging Service](#1-sap-cloud-logging-service)
- [2. Metrics & Monitoring](#2-metrics--monitoring)
- [3. Distributed Tracing](#3-distributed-tracing)
- [4. Alerting & Dashboards](#4-alerting--dashboards)
- [5. Incident Response & Diagnostics](#5-incident-response--diagnostics)
- [Top 5 Pitfalls](#top-5-pitfalls)
- [What to Learn Next](#what-to-learn-next)

---

> **Summary:** Observability on SAP BTP requires understanding a specific stack: SAP Cloud Logging (based on OpenSearch) for logs, Prometheus/Dynatrace for metrics, OpenTelemetry for distributed tracing, and Cloud ALM for alerting. This lesson covers how to instrument CAP Java applications running on Cloud Foundry and Kyma for full observability, including structured logging with correlation IDs, custom Prometheus metrics, W3C Trace Context propagation across service boundaries, and practical incident response techniques for diagnosing JVM issues in BTP containers.

---

## 1. SAP Cloud Logging Service

### Architecture

SAP Cloud Logging Service is built on the OpenSearch stack (the open-source fork of Elasticsearch + Kibana):

```
┌─────────────────────────────────────────────────────────────────┐
│                    SAP Cloud Logging Service                     │
│                                                                  │
│  ┌────────────┐     ┌──────────────┐     ┌────────────────────┐ │
│  │ Ingest     │     │ OpenSearch   │     │ OpenSearch         │ │
│  │ Pipeline   │────→│ Cluster      │────→│ Dashboards         │ │
│  │ (Fluent    │     │ (index,      │     │ (visualization,    │ │
│  │  Bit)      │     │  search)     │     │  alerting)         │ │
│  └──────▲─────┘     └──────────────┘     └────────────────────┘ │
│         │                                                        │
└─────────┼────────────────────────────────────────────────────────┘
          │
          │ stdout / syslog
          │
┌─────────┴────────────────────────────────────────────────────────┐
│  ┌─────────────────┐    ┌─────────────────┐                      │
│  │ CF App Instances │    │ Kyma Pods        │                      │
│  │ (stdout → Diego  │    │ (stdout → Fluent │                      │
│  │  loggregator)    │    │  Bit DaemonSet)  │                      │
│  └─────────────────┘    └─────────────────┘                      │
└──────────────────────────────────────────────────────────────────┘
```

### Structured Logging for CAP Java

The key to useful logs on BTP is **structured JSON logging**. CF and Kyma both capture stdout — structured logs enable filtering, searching, and correlation in OpenSearch.

**Logback configuration (`logback-spring.xml`):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- JSON layout for production -->
    <springProfile name="cloud">
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="com.sap.hcp.cf.logback.encoder.JsonEncoder">
                <!-- SAP Cloud Logging fields -->
                <customField>
                    <key>component</key>
                    <value>my-cap-srv</value>
                </customField>
            </encoder>
        </appender>
    </springProfile>

    <!-- Human-readable for local development -->
    <springProfile name="!cloud">
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
    </springProfile>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>

    <!-- CAP runtime logging -->
    <logger name="com.sap.cds" level="INFO"/>
    <!-- SQL query logging (DEBUG for development only) -->
    <logger name="com.sap.cds.persistence" level="WARN"/>
</configuration>
```

**Maven dependency for SAP logging library:**

```xml
<dependency>
    <groupId>com.sap.hcp.cf.logging</groupId>
    <artifactId>cf-java-logging-support-logback</artifactId>
    <version>3.8.3</version>
</dependency>
```

### Correlation IDs

Correlation IDs allow tracing a single request across multiple services. SAP's logging library automatically propagates the `X-CorrelationID` header:

```java
@Component
@ServiceName(CatalogService_.CDS_NAME)
public class CatalogServiceHandler implements EventHandler {

    private static final Logger log = LoggerFactory.getLogger(CatalogServiceHandler.class);

    @Before(event = CqnService.EVENT_CREATE, entity = Books_.CDS_NAME)
    public void beforeCreateBook(CdsCreateEventContext context) {
        // Correlation ID is automatically included in JSON log output
        log.info("Creating book: title={}", context.getCqn().entries().get(0).get("title"));

        // Access correlation ID programmatically if needed
        String correlationId = LogContext.getCorrelationId();
        log.info("Processing with correlation ID: {}", correlationId);
    }
}
```

**Resulting JSON log entry:**

```json
{
  "written_at": "2024-01-15T10:30:00.000Z",
  "written_ts": 1705312200000000,
  "msg": "Creating book: title=Clean Architecture",
  "level": "INFO",
  "logger": "com.sap.demo.handlers.CatalogServiceHandler",
  "thread": "http-nio-8080-exec-1",
  "correlation_id": "abc123-def456-ghi789",
  "request_id": "req-001",
  "tenant_id": "tenant-xyz",
  "component": "my-cap-srv",
  "layer": "[JAVA]"
}
```

### Log Levels Strategy

| Level | Usage | Example |
|---|---|---|
| `ERROR` | Unrecoverable failures requiring action | Database connection failed, external API 5xx |
| `WARN` | Degraded but recoverable | Cache miss, retry attempt, deprecated API used |
| `INFO` | Business events and lifecycle | Order created, tenant provisioned, app started |
| `DEBUG` | Technical details (never in production) | SQL queries, HTTP headers, CQN details |
| `TRACE` | Very fine-grained (development only) | Method entry/exit, variable values |

> **Pitfall:** Enabling `DEBUG` for `com.sap.cds.persistence` in production will log every SQL query with parameters, causing massive log volume and potentially exposing sensitive data.

---

## 2. Metrics & Monitoring

### Spring Boot Actuator on BTP

CAP Java applications run on Spring Boot, so the full Actuator metrics stack is available:

```yaml
# application.yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      space: ${vcap.application.space_name:local}
```

### Custom Business Metrics

```java
@Component
@ServiceName(CatalogService_.CDS_NAME)
public class CatalogServiceHandler implements EventHandler {

    private final MeterRegistry meterRegistry;
    private final Counter orderCounter;
    private final Timer orderProcessingTimer;

    public CatalogServiceHandler(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.orderCounter = Counter.builder("app.orders.total")
            .description("Total number of orders submitted")
            .tag("service", "catalog")
            .register(meterRegistry);
        this.orderProcessingTimer = Timer.builder("app.orders.processing.time")
            .description("Order processing duration")
            .tag("service", "catalog")
            .register(meterRegistry);
    }

    @On(event = "submitOrder")
    public void onSubmitOrder(SubmitOrderContext context) {
        orderProcessingTimer.record(() -> {
            // Business logic here
            processOrder(context);
            orderCounter.increment();
        });
    }

    // Gauge example — track active book count
    @PostConstruct
    public void registerGauges() {
        Gauge.builder("app.books.active", this, handler -> getActiveBookCount())
            .description("Number of books with stock > 0")
            .register(meterRegistry);
    }
}
```

### Metrics on Cloud Foundry vs Kyma

| Aspect | Cloud Foundry | Kyma |
|---|---|---|
| Built-in metrics | Diego container metrics (CPU, memory, disk) | K8s container metrics via cAdvisor |
| Custom metrics | Expose `/actuator/prometheus`, scrape via SAP Cloud Logging or Dynatrace | Prometheus ServiceMonitor CRD |
| Autoscaling | SAP Application Autoscaler (CPU, memory, throughput, latency) | Kubernetes HPA (CPU, memory, custom) |
| APM | Dynatrace OneAgent (BTP integration) | Dynatrace Operator on K8s |

### Prometheus on Kyma — ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-srv-metrics
  namespace: my-app
  labels:
    app: my-srv
spec:
  selector:
    matchLabels:
      app: my-srv
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 30s
      scrapeTimeout: 10s
```

### BTP Health Check Pattern

```java
@Component
public class BtpHealthIndicator implements HealthIndicator {

    private final PersistenceService db;
    private final DestinationAccessor destinationAccessor;

    public BtpHealthIndicator(PersistenceService db) {
        this.db = db;
        this.destinationAccessor = null; // inject if needed
    }

    @Override
    public Health health() {
        Health.Builder builder = new Health.Builder();

        // Check database connectivity
        try {
            db.run(Select.from("DUMMY").columns("1"));
            builder.withDetail("database", "UP");
        } catch (Exception e) {
            return builder.down()
                .withDetail("database", "DOWN: " + e.getMessage())
                .build();
        }

        // Check XSUAA connectivity
        try {
            // Verify XSUAA public key endpoint is reachable
            builder.withDetail("xsuaa", "UP");
        } catch (Exception e) {
            builder.withDetail("xsuaa", "DEGRADED: " + e.getMessage());
        }

        return builder.up().build();
    }
}
```

---

## 3. Distributed Tracing

### OpenTelemetry Instrumentation

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ Approuter│────→│ CAP Java │────→│ S/4HANA  │     │  Event   │
│          │     │ Service  │────→│ OData API│     │  Mesh    │
│          │     │          │     │          │     │          │
│ trace-id │     │ trace-id │     │ trace-id │     │ trace-id │
│ span-a   │     │ span-b   │     │ span-c   │     │ span-d   │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
     │                │                │                │
     └────────────────┴────────────────┴────────────────┘
                              │
                     ┌────────▼────────┐
                     │  Trace Backend  │
                     │  (Jaeger /      │
                     │   Cloud Logging)│
                     └─────────────────┘
```

**Maven dependencies:**

```xml
<!-- OpenTelemetry auto-instrumentation -->
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
    <version>2.1.0</version>
</dependency>
<!-- OTLP exporter -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

**Configuration:**

```yaml
# application.yaml
otel:
  service:
    name: my-cap-srv
  traces:
    exporter: otlp
  exporter:
    otlp:
      endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4317}
      protocol: grpc
  instrumentation:
    spring-web:
      enabled: true
    jdbc:
      enabled: true
```

### Custom Span for Business Operations

```java
@Component
@ServiceName(CatalogService_.CDS_NAME)
public class CatalogServiceHandler implements EventHandler {

    private final Tracer tracer;

    public CatalogServiceHandler(Tracer tracer) {
        this.tracer = tracer;
    }

    @On(event = "submitOrder")
    public void onSubmitOrder(SubmitOrderContext context) {
        Span span = tracer.spanBuilder("submitOrder")
            .setAttribute("book.id", context.getBook())
            .setAttribute("order.quantity", context.getQuantity())
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // Step 1: Validate stock
            Span validateSpan = tracer.spanBuilder("validateStock").startSpan();
            try {
                validateStock(context.getBook(), context.getQuantity());
            } finally {
                validateSpan.end();
            }

            // Step 2: Process order
            Span processSpan = tracer.spanBuilder("processOrder").startSpan();
            try {
                processOrder(context);
            } finally {
                processSpan.end();
            }

            span.setStatus(StatusCode.OK);
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### W3C Trace Context Propagation

When calling remote services (S/4HANA via Cloud SDK, Event Mesh), propagate the trace context:

```java
// Cloud SDK automatically propagates W3C traceparent header
// when OpenTelemetry is on the classpath

HttpDestination destination = DestinationAccessor
    .getDestination("S4HANA")
    .asHttp();

// The traceparent header is automatically injected:
// traceparent: 00-<trace-id>-<span-id>-01
List<BusinessPartner> partners = new DefaultBusinessPartnerService()
    .getAllBusinessPartner()
    .executeRequest(destination);
```

For Event Mesh, include trace context as CloudEvent extension:

```java
// When publishing events, include trace context
Map<String, String> extensions = new HashMap<>();
extensions.put("traceparent", Span.current().getSpanContext().getTraceId());
messagingService.emit("OrderCreated", orderData, extensions);
```

---

## 4. Alerting & Dashboards

### SAP Cloud ALM Integration

```
┌─────────────────────────────────────────────────────────────┐
│                    SAP Cloud ALM                             │
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐ │
│  │ Health       │   │ Integration  │   │ Real User        │ │
│  │ Monitoring   │   │ & Exception  │   │ Monitoring       │ │
│  │              │   │ Monitoring   │   │                  │ │
│  │ - App health │   │ - iFlow logs │   │ - Fiori UX       │ │
│  │ - Service    │   │ - API errors │   │ - Page load      │ │
│  │   availability   │ - Event proc │   │ - User sessions  │ │
│  └──────────────┘   └──────────────┘   └──────────────────┘ │
└──────────────────────────┬──────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │  Open ALM API           │
              │  /api/calm/v1/events    │
              └─────────────────────────┘
```

### Alert Configuration Examples

**OpenSearch-based alerting (Cloud Logging):**

```json
{
  "trigger": {
    "name": "high-error-rate",
    "severity": "1",
    "condition": {
      "script": {
        "source": "ctx.results[0].hits.total.value > 50",
        "lang": "painless"
      }
    },
    "actions": [
      {
        "name": "notify-team",
        "destination_id": "slack-webhook-id",
        "message_template": {
          "source": "High error rate detected: {{ctx.results[0].hits.total.value}} errors in last 5 minutes"
        }
      }
    ]
  },
  "inputs": [
    {
      "search": {
        "indices": ["logs-*"],
        "query": {
          "bool": {
            "must": [
              { "match": { "level": "ERROR" } },
              { "match": { "component": "my-cap-srv" } },
              { "range": { "written_at": { "gte": "now-5m" } } }
            ]
          }
        }
      }
    }
  ]
}
```

### Key Dashboards to Build

| Dashboard | Metrics | Purpose |
|---|---|---|
| **Service Health** | HTTP status codes, response time P50/P95/P99, error rate | Real-time service overview |
| **Business KPIs** | Orders/min, active users, tenant provisioning rate | Business impact monitoring |
| **Infrastructure** | CPU, memory, disk, container restarts, pod evictions | Capacity planning |
| **Database** | Connection pool utilization, query duration, HDI container count | HANA performance |
| **Integration** | Event Mesh queue depth, failed deliveries, API call latency | External dependency health |

---

## 5. Incident Response & Diagnostics

### Cloud Foundry Diagnostics

```bash
# View recent logs (last 100 lines)
cf logs my-srv --recent

# Stream live logs
cf logs my-srv

# Check application events (crashes, restarts)
cf events my-srv

# Application environment (VCAP_SERVICES, memory)
cf env my-srv

# Scale up for debugging (temporary)
cf scale my-srv -m 2G -k 1G

# SSH into a running container
cf ssh my-srv
# Inside the container:
#   /tmp/lifecycle/shell   # access the app's environment
#   kill -3 <pid>          # trigger thread dump to stdout
#   jcmd <pid> GC.heap_info  # heap summary

# Download heap dump (CF)
cf ssh my-srv -c "jcmd 1 GC.heap_dump /tmp/heap.hprof"
cf ssh my-srv --command "cat /tmp/heap.hprof" > heap.hprof
```

### Kyma / Kubernetes Diagnostics

```bash
# Pod status and recent events
kubectl get pods -n my-app -l app=my-srv -o wide
kubectl describe pod <pod-name> -n my-app

# View logs (current and previous crashed container)
kubectl logs <pod-name> -n my-app
kubectl logs <pod-name> -n my-app --previous  # crashed container logs

# Stream logs from all pods
kubectl logs -f -l app=my-srv -n my-app --all-containers

# Execute commands in running pod
kubectl exec -it <pod-name> -n my-app -- /bin/sh

# Thread dump
kubectl exec <pod-name> -n my-app -- jcmd 1 Thread.print

# Heap dump
kubectl exec <pod-name> -n my-app -- jcmd 1 GC.heap_dump /tmp/heap.hprof
kubectl cp my-app/<pod-name>:/tmp/heap.hprof ./heap.hprof

# Ephemeral debug container (when no shell in image)
kubectl debug -it <pod-name> -n my-app \
  --image=busybox:latest \
  --target=my-srv

# Resource consumption
kubectl top pods -n my-app
kubectl top nodes
```

### Common Java Performance Issues on BTP

#### 1. Memory Limit OOM Kill

```
┌────────────────────────────────────────────────┐
│  Container Memory Limit: 1024 MB               │
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │  JVM Heap: 750 MB (-XX:MaxRAMPercentage │   │
│  │            =75%)                         │   │
│  ├─────────────────────────────────────────┤   │
│  │  Metaspace: ~100 MB                     │   │
│  ├─────────────────────────────────────────┤   │
│  │  Thread stacks: ~100 MB (200 threads    │   │
│  │                  × 512 KB)              │   │
│  ├─────────────────────────────────────────┤   │
│  │  Direct buffers, JIT, native: ~74 MB    │   │
│  └─────────────────────────────────────────┘   │
│  Total: ~1024 MB → OOM kill by cgroup!         │
│                                                 │
│  Fix: Use -XX:MaxRAMPercentage=65.0            │
│  or increase container memory limit             │
└────────────────────────────────────────────────┘
```

#### 2. Connection Pool Exhaustion

```java
// application.yaml — HikariCP tuning for CAP
spring:
  datasource:
    hikari:
      maximum-pool-size: 15        # default 10, tune per CF instance count
      minimum-idle: 5
      connection-timeout: 5000     # fail fast, don't block thread
      idle-timeout: 300000         # 5 min
      max-lifetime: 600000         # 10 min (below HANA timeout)
      leak-detection-threshold: 30000  # log warning if connection held > 30s

# Monitor with metrics
management:
  metrics:
    enable:
      hikaricp: true
```

#### 3. Thread Starvation

```
Symptom: Requests timing out, thread pool exhausted
Cause:   Blocking calls to remote services on the servlet thread pool

Fix: Ensure proper timeout configuration
```

```yaml
# application.yaml
cds:
  remote:
    services:
      '[API_BUSINESS_PARTNER]':
        http:
          timeout: 10000          # 10s timeout for remote calls
server:
  tomcat:
    threads:
      max: 200                     # default, increase if needed
      min-spare: 20
    connection-timeout: 20000      # 20s
```

### Diagnostic Checklist

```
Application unresponsive?
├── Check: kubectl top pod / cf app <name>
│   ├── Memory > 90% → likely OOM, check heap dump
│   ├── CPU > 90% → thread dump, check for infinite loops / GC pressure
│   └── Normal → check connection pools and thread pools
│
├── Check: kubectl logs / cf logs --recent
│   ├── OOM errors → increase memory or reduce MaxRAMPercentage
│   ├── Connection refused → downstream service down
│   ├── 401/403 → XSUAA token expired or scope mismatch
│   └── Timeout → remote service latency, check circuit breaker
│
├── Check: Database
│   ├── HikariCP pool exhausted → increase pool size or fix connection leaks
│   ├── Slow queries → HANA explain plan, check delta merge
│   └── Lock contention → check for long transactions in @On handlers
│
└── Check: Event Mesh (if event-driven)
    ├── Queue depth growing → consumer not keeping up, scale out
    ├── Dead-letter queue → inspect failed messages
    └── No events received → check subscription and topic configuration
```

---

## Top 5 Pitfalls

1. **Not using structured JSON logging.** Plain text logs are almost useless in OpenSearch. Switch to JSON format (`cf-java-logging-support`) for all BTP deployments.
2. **Setting `-Xmx` instead of `-XX:MaxRAMPercentage`.** Hard-coded heap sizes break when container memory limits change. Use percentage-based settings for container-aware JVM tuning.
3. **Ignoring correlation IDs.** Without correlation IDs, tracing a request across approuter → CAP service → remote API → Event Mesh is nearly impossible. SAP's logging library handles this automatically — don't disable it.
4. **Exposing Actuator endpoints without authentication.** In production, `/actuator/prometheus` and `/actuator/health` must be protected or only exposed on an internal port not reachable from the APIRule/route.
5. **Not setting connection pool leak detection.** Connection leaks in CAP handlers (especially with custom JDBC code alongside CQN) are silent killers. HikariCP's `leak-detection-threshold` surfaces them immediately.

---

## What to Learn Next

- **Lesson 4.1:** CI/CD for SAP BTP Applications — pipeline observability integration
- **Lesson 4.3:** Spring Boot to CAP Migration Guide — mapping monitoring patterns
- **Lesson 1.1:** BTP Architecture — connectivity and security model (prerequisite for understanding traces)
- **Lesson 2.1:** Kyma Architecture — Istio observability features (Kiali, Jaeger)