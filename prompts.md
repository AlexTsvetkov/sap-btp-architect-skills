# SAP Backend Architect — Learning Prompts

> **Audience:** Java architects and senior developers learning the SAP stack.
> **Goal:** Generate deep, technical lessons with architecture diagrams (textual), code examples, and real-world patterns.

---

## How to Use These Prompts

Copy a prompt into your AI assistant (ChatGPT, Copilot, Claude, etc.).
Each prompt is self-contained and designed to produce a lesson that is:

- **Technical** — code-heavy, architecture-focused, no marketing fluff
- **Deep** — covers internals, not just "getting started" tutorials
- **Practical** — includes working examples, common pitfalls, and production considerations
- **Architect-oriented** — discusses trade-offs, scalability, and design decisions

---

## 1 · SAP Business Technology Platform (BTP)

### 1.1 BTP Architecture & Runtime Model

```
Act as a senior SAP architect. Write a deep technical lesson on the SAP BTP architecture
for an experienced Java developer. Cover:

1. BTP multi-environment architecture: Cloud Foundry vs Kyma vs ABAP environment —
   when to use each, internal differences, and resource isolation model
2. Subaccount, directory, and entitlement hierarchy — how quota and service plans work
   at the platform level
3. The BTP service marketplace internals — how service brokers implement the
   Open Service Broker API, how binding injection works, and the VCAP_SERVICES structure
4. BTP connectivity: Cloud Connector architecture, the reverse-invoke tunnel to on-premise
   systems, principal propagation flow (mutual TLS + SAML assertion), and destination
   service resolution
5. BTP security model: XSUAA internals — JWT token structure, scope/authority mapping,
   role collections, and the OAuth2 flows (client_credentials, authorization_code,
   user_token exchange)

Include architecture diagrams in text/ASCII where helpful.
Highlight pitfalls that Java developers commonly face when coming from Spring Boot / AWS / Azure.
```

### 1.2 BTP Services Deep Dive — SAP HANA Cloud

```
Act as a database architect with SAP HANA expertise. Write a technical lesson covering:

1. SAP HANA Cloud on BTP: provisioning model, elastic scaling, and the difference between
   HANA Cloud and on-premise HANA (memory model, disk-based tables, native storage extension)
2. HANA multi-model engine: column store internals, row store use cases, document store
   (JSON collections), graph engine, and spatial engine — when to use each
3. HDI (HANA Deployment Infrastructure): container model, design-time vs runtime artifacts,
   .hdbcds vs .hdbtable, calculation views, and how HDI containers map to DB schemas
4. Connection from Cloud Foundry / Kyma apps: service binding anatomy, JDBC vs
   node-hdb vs CAP's built-in persistence layer
5. Performance tuning: explain plans, partition pruning, delta merge, memory consumption
   analysis, and common anti-patterns (e.g., row-by-row processing, overusing calc views)

Provide SQL examples. Compare with PostgreSQL / MySQL where it helps a Java developer
understand the differences.
```

### 1.3 BTP Event Mesh & Integration

```
Act as an integration architect. Write a deep technical lesson on event-driven architecture
in SAP BTP for Java developers. Cover:

1. SAP Event Mesh: architecture, queues vs topic subscriptions, QoS levels, the
   webhook registration model, and how it compares to Kafka / RabbitMQ
2. Message protocol: how SAP uses CloudEvents specification — envelope format,
   event types from S/4HANA (e.g., sap.s4.beh.businesspartner.changed.v1),
   and how to subscribe to business object events
3. Enterprise messaging patterns: fan-out, competing consumers, dead-letter queues,
   retry with exponential backoff — how to implement each on Event Mesh
4. SAP Integration Suite vs Event Mesh: when to use iFlow-based integration vs
   event-based integration, and how to combine both in a hybrid architecture
5. Implementing an event-driven microservice in CAP Java that listens to S/4HANA
   business events, processes them, and publishes derived events

Include working code snippets and sequence diagrams (text-based).
```

### 1.4 BTP Multi-Tenancy

```
Act as a senior SaaS architect. Write a deep technical lesson on building multi-tenant
applications on SAP BTP. Cover:

1. BTP multi-tenancy model: tenant provisioning lifecycle, SaaS Provisioning Service (saas-registry),
   the onSubscription callback pattern, and tenant-aware routing
2. Tenant isolation patterns: schema-per-tenant (HDI containers), shared-schema with
   discriminator column, and hybrid approaches — trade-offs for each
3. XSUAA in multi-tenant context: how the approuter resolves tenants from subdomain,
   tenant-specific identity zones, and how JWT tokens carry tenant info (zid claim)
4. CAP's built-in multi-tenancy support: @sap/cds-mtxs module, sidecar deployment,
   extensibility model, and how schema upgrade propagates across tenants
5. Operational concerns: tenant-aware logging, monitoring per tenant, noisy-neighbor
   detection, and data isolation compliance (GDPR per tenant)

Compare with multi-tenancy patterns in Spring Boot / Hibernate to bridge existing knowledge.
```

---

## 2 · SAP Kyma Runtime (Kubernetes-Based)

### 2.1 Kyma Architecture & Core Components

```
Act as a Kubernetes and SAP platform architect. Write a deep technical lesson on
SAP Kyma Runtime for an experienced Java developer familiar with Kubernetes. Cover:

1. Kyma runtime vs vanilla Kubernetes: what Kyma adds on top (Istio service mesh,
   Eventing, Serverless, API Gateway) and what is managed by SAP vs self-managed
2. Kyma cluster architecture: system namespaces, Kyma modules system (modular installation),
   the BTP Operator for service consumption, and the lifecycle manager
3. Istio service mesh in Kyma: sidecar injection, mTLS between pods, VirtualService /
   DestinationRule configuration, traffic management, and observability (Kiali, Jaeger)
4. Kyma API Gateway: APIRule CRD, JWT validation with XSUAA/IAS, CORS configuration,
   rate limiting, and how it maps to Istio VirtualServices under the hood
5. Kyma Eventing: NATS-based vs SAP Event Mesh backend, Subscription CRD, event
   type versioning, and exactly-once delivery guarantees (or lack thereof)

Include YAML manifests and kubectl commands. Highlight differences from plain k8s + Istio setups.
```

### 2.2 Deploying Java Microservices on Kyma

```
Act as a senior Java developer and Kubernetes practitioner. Write a hands-on technical
lesson on deploying Spring Boot / CAP Java microservices to SAP Kyma. Cover:

1. Container image strategy: multi-stage Dockerfile for Java apps, GraalVM native image
   considerations, image size optimization, and pushing to a registry accessible from Kyma
2. Kubernetes manifests: Deployment, Service, HPA, PodDisruptionBudget, resource
   requests/limits tuning for JVM workloads (-XX:MaxRAMPercentage, container-aware GC)
3. Helm chart structure for a Java microservice on Kyma: values.yaml for
   environment-specific config, secrets management with Kubernetes Secrets and
   SAP BTP Operator's ServiceBinding CRD
4. Service binding consumption: how the BTP Operator injects credentials into pods
   (mounted volumes vs env vars), parsing VCAP_SERVICES in Spring Boot, and using
   @sap/xssec or spring-xsuaa for token validation
5. Observability: structured logging with SLF4J + Logback to stdout, Prometheus metrics
   endpoint, distributed tracing with OpenTelemetry, and connecting to SAP Cloud Logging

Include a complete Helm chart example and Dockerfile. Explain CI/CD pipeline integration.
```

### 2.3 Kyma Serverless Functions

```
Act as a cloud-native architect. Write a technical lesson on Kyma Serverless Functions
for a Java developer. Cover:

1. Kyma Function CRD: runtime options (Node.js, Python), resource configuration,
   scaling behavior (scale-to-zero), and cold start implications
2. When to use Functions vs Deployments: latency sensitivity, long-running processes,
   statefulness, and resource efficiency trade-offs
3. Function event triggers: binding a Function to Kyma Eventing Subscription CRD,
   processing CloudEvents, and implementing idempotent handlers
4. API exposure: creating APIRule for a Function, chaining Functions with internal
   service calls, and using Functions as BTP extension points
5. Limitations and workarounds: no Java runtime (how to work around with containers),
   execution timeout, payload size limits, and debugging strategies

Provide YAML manifests and function code examples. Compare with AWS Lambda / Azure Functions
for context.
```

### 2.4 Kyma Eventing & Extension Patterns

```
Act as an SAP extension architect. Write a deep technical lesson on extending SAP
S/4HANA using Kyma eventing. Cover:

1. S/4HANA eventing setup: enabling the Enterprise Event Enablement framework,
   channel configuration, and how business events are emitted from ABAP (BAPI / RAP)
2. Event flow: S/4HANA → Event Mesh → Kyma NATS / Event Mesh backend → Subscription
   CRD → your microservice — full trace with headers and CloudEvents structure
3. Side-by-side extension pattern: receiving a BusinessPartner.Changed event, calling
   back into S/4HANA OData API to read full data, enriching it, and storing in
   a Kyma-hosted HANA Cloud or PostgreSQL instance
4. Error handling: dead-letter topics, retry logic in the eventing infrastructure,
   circuit breaker pattern with Istio, and compensating transactions
5. Real-world example: implement an order validation extension that intercepts
   SalesOrder.Created, runs custom business logic in a Spring Boot service on Kyma,
   and posts approval results back to S/4HANA

Include architecture diagram, YAML manifests, and Java code.
```

---

## 3 · SAP Cloud Application Programming Model (CAP)

### 3.1 CAP Java — Architecture & Core Concepts

```
Act as a senior CAP developer and Java architect. Write a deep technical lesson on
SAP CAP with Java runtime for experienced Java developers. Cover:

1. CAP architecture: the CDS compiler pipeline (parse → CSN → EDMX / SQL / Fiori),
   the runtime dispatch model (event-based), and how CAP Java maps CDS to Java services
2. CDS modeling: entities, types, aspects, compositions vs associations, managed aspects
   (cuid, managed, temporal), and the underlying relational mapping
3. CAP Java service model: CqnService, event handlers (@On, @Before, @After), the
   EventContext object, and the request processing lifecycle
4. Querying with CQN (CDS Query Notation): Select, Insert, Upsert, Delete builders,
   projections, filters, expand for deep reads, and how CQN compiles to SQL
5. Comparison with JPA / Hibernate: what CAP gives you vs what you lose, N+1 problem
   handling, lazy vs eager loading, and transaction management differences

Include CDS model definitions and Java handler code. Explain the mental model shift
from Spring Data JPA to CAP.
```

### 3.2 CAP — Security, Authorization & Authentication

```
Act as a security architect. Write a deep technical lesson on securing CAP Java
applications on BTP. Cover:

1. Authentication setup: XSUAA service binding, spring-security integration via
   cds-starter-cloudfoundry or cds-starter-k8s, and the token validation chain
2. CDS-level authorization: @requires, @restrict with where-clauses, grant/deny per
   event, instance-based authorization, and how these annotations compile to runtime checks
3. Custom authorization logic: writing @Before handlers for fine-grained access control,
   accessing UserInfo from EventContext, and implementing attribute-based access control (ABAC)
4. Multitenancy security: tenant isolation enforcement, cross-tenant access prevention,
   and how CAP ensures CQN queries are scoped to the correct HDI container
5. Common security pitfalls: missing @restrict on auto-exposed entities via compositions,
   OData $expand bypassing entity-level security, bulk operations and authorization checks,
   and how to write integration tests for authorization rules

Include CDS annotations, Java handler code, and XSUAA security descriptor (xs-security.json).
```

### 3.3 CAP — Remote Services & Mashups

```
Act as an integration developer. Write a technical lesson on consuming external services
from SAP CAP Java applications. Cover:

1. Importing external API definitions: using EDMX files for OData services (S/4HANA APIs),
   OpenAPI specs for REST APIs, and how CAP generates typed models from these
2. Remote service projection: modeling external entities in CDS, creating projections
   and mashups that combine local and remote data in a single service
3. Runtime delegation: how CAP routes CQN queries to remote OData services, the
   CloudSDK HTTP client underneath, destination resolution, and principal propagation
4. Error handling and resilience: timeout configuration, retry policies, circuit breaker
   setup with Resilience4j, handling partial failures in mashup scenarios
5. Performance optimization: request batching ($batch), caching strategies for remote
   data, lazy loading of remote associations, and minimizing round trips

Include CDS models, Java code, and mta.yaml / package.json configuration for destinations.
```

### 3.4 CAP — Advanced Patterns & Production Readiness

```
Act as a senior backend architect. Write a deep technical lesson on advanced CAP Java
patterns and production-readiness. Cover:

1. Custom protocol adapters: exposing services via REST (non-OData), gRPC, GraphQL,
   or messaging (Kafka / Event Mesh) — how CAP's protocol-agnostic model enables this
2. Domain-driven design with CAP: bounded contexts as CDS services, aggregate roots
   as compositions, domain events via CAP messaging, and anti-corruption layers for
   external system integration
3. Testing strategies: unit testing handlers with CDS runtime in-memory, integration
   testing with Testcontainers + HANA or PostgreSQL, API testing with OData/REST clients,
   and mocking remote services
4. Performance and scalability: connection pooling (HikariCP config), parallel CQN
   execution, batch processing large datasets, and horizontal scaling considerations
5. Deployment topologies: CF vs Kyma deployment, sidecar model for MTX, blue-green
   deployment with MTA, Helm-based deployment on Kyma, and database migration strategy
   across environments (HDI deployer, Liquibase alternative)

Include code examples, configuration snippets, and architecture decision records (ADRs).
```

### 3.5 CAP — Fiori Elements Integration

```
Act as a full-stack SAP developer. Write a technical lesson on building SAP Fiori Elements
UIs on top of CAP Java services for a backend developer. Cover:

1. OData V4 annotations in CDS: @UI.LineItem, @UI.FieldGroup, @UI.HeaderInfo,
   @UI.SelectionFields, @UI.Chart — how these drive the Fiori Elements UI generation
2. Draft handling: CDS @odata.draft.enabled, the draft lifecycle (edit → patch → prepare →
   activate → save), draft tables, and how CAP manages draft storage automatically
3. Value helps: @Common.ValueList annotation, dependent value helps with filter conditions,
   and free-text search in value help dialogs
4. Custom sections and extensions: embedding custom UI5 fragments in Fiori Elements pages,
   controller extensions, and when to drop down to freestyle UI5
5. Backend considerations: how annotation-driven UIs impact service design, avoiding
   deeply nested compositions that break FE assumptions, pagination ($skip/$top),
   and handling large entity sets with server-side filtering and sorting

Include CDS annotation examples, resulting UI screenshots descriptions, and common issues.
```

---

## 4 · Cross-Cutting Concerns

### 4.1 CI/CD for SAP BTP Applications

```
Act as a DevOps engineer specializing in SAP BTP. Write a technical lesson on CI/CD
pipelines for CAP applications deployed to BTP (CF and Kyma). Cover:

1. SAP CI/CD Service vs GitHub Actions vs Jenkins: capabilities comparison,
   when to use each, and integration with BTP
2. MTA build pipeline: mbt build, MTA modules (Java backend, Node.js sidecar,
   Fiori app, HANA deployer), and how the MTA archive (.mtar) is structured
3. Helm + Docker pipeline for Kyma: building container images, Helm chart packaging,
   environment promotion (dev → staging → prod), and secrets management
4. Quality gates: CAP-specific linting (cds lint), OPA policy checks, OWASP dependency
   scan, unit and integration test execution, and code coverage thresholds
5. Deployment strategies: blue-green with `cf bg-deploy`, canary releases on Kyma with
   Istio traffic splitting, rollback procedures, and database migration safety

Include pipeline YAML examples (GitHub Actions) and explain the full flow from commit to production.
```

### 4.2 Observability & Operations on BTP

```
Act as an SRE / platform engineer. Write a deep technical lesson on observability
for Java applications running on SAP BTP. Cover:

1. SAP Cloud Logging Service: architecture (based on OpenSearch), log ingestion from
   CF and Kyma, structured logging best practices, and correlation IDs across services
2. Metrics: SAP Application Autoscaler, custom Prometheus metrics from Spring Boot
   Actuator, Dynatrace integration, and BTP-specific health check patterns
3. Distributed tracing: OpenTelemetry instrumentation for CAP Java, W3C Trace Context
   propagation, tracing across Event Mesh and remote OData calls, and trace visualization
4. Alerting and dashboards: Cloud ALM integration, setting up alerts on log patterns,
   metric thresholds, and SLA monitoring
5. Incident response: analyzing thread dumps and heap dumps from CF containers,
   accessing Kyma pod diagnostics (kubectl exec, ephemeral containers), and
   common Java performance issues on BTP (memory limits, connection leaks, thread starvation)

Include configuration snippets and real-world troubleshooting scenarios.
```

### 4.3 SAP BTP for Spring Boot Developers — Migration Guide

```
Act as a Java architect who has migrated projects from Spring Boot to SAP CAP Java.
Write a comparative lesson that maps Spring Boot concepts to SAP CAP/BTP equivalents:

1. Spring Web MVC / REST → CAP OData / REST protocol adapters
2. Spring Data JPA + Hibernate → CDS models + CQN queries
3. Spring Security + OAuth2 → XSUAA + @restrict annotations
4. Spring Cloud Config → BTP Destination service + Environment variables
5. Spring Boot Actuator → SAP Cloud Logging + Prometheus + Health checks
6. Spring Cloud Stream (Kafka) → CAP Messaging + Event Mesh
7. Spring Profiles → MTA deployment descriptors + Helm values
8. Testcontainers → CAP testing with in-memory CDS or Testcontainers
9. Flyway / Liquibase → HDI deployer + CDS schema evolution

For each mapping, explain what's similar, what's different, what's better, and what's worse.
Include code comparisons side by side. Be honest about limitations.
```

---

## 5 · Lesson Template Prompt (Generic)

Use this template to generate a lesson on any SAP topic:

```
Act as a senior SAP architect writing for experienced Java developers.

Write a deep technical lesson on: [TOPIC]

Requirements:
- Start with a one-paragraph executive summary
- Include architecture diagrams in ASCII/text format
- Provide working code examples (CDS, Java, YAML, SQL as needed)
- Compare with equivalent non-SAP technology where helpful
- List the top 5 common mistakes / pitfalls
- End with a "What to learn next" section with concrete topics
- Tone: technical, precise, no marketing language
- Length: 2000-3000 words
- Assume the reader knows Java, Spring Boot, Kubernetes, SQL, and REST/OData basics
```

---

## Contributing

Feel free to add new prompts via pull request. Each prompt should:
1. Target a specific, well-scoped topic
2. Specify the expert persona to adopt
3. List 4-6 numbered subtopics for depth
4. Request code examples and comparisons
5. Set the audience context (Java architect / senior developer)