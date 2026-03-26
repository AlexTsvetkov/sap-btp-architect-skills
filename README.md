# SAP BTP Backend Architect Skills

A comprehensive learning path for Java/Spring Boot developers transitioning to SAP Business Technology Platform (BTP) architecture. Each lesson is designed for experienced backend engineers and covers BTP-specific concepts, patterns, and pitfalls in depth.

---

## Curriculum Overview

### Module 1 · SAP BTP Foundations

| # | Lesson | Topics |
|---|--------|--------|
| 1.1 | [BTP Architecture & Runtime Model](lessons/01-btp/01-btp-architecture-runtime-model.md) | Multi-environment architecture (CF, Kyma, ABAP), subaccount hierarchy, service brokers, VCAP_SERVICES, Cloud Connector, XSUAA internals |
| 1.2 | [HANA Cloud Deep Dive](lessons/01-btp/02-hana-cloud-deep-dive.md) | HANA Cloud vs on-premise, multi-model engine, HDI containers, CF/Kyma connectivity, performance tuning |
| 1.3 | [Event Mesh & Integration](lessons/01-btp/03-event-mesh-integration.md) | Event Mesh architecture, CloudEvents spec, enterprise messaging patterns, Integration Suite comparison |
| 1.4 | [Multi-Tenancy](lessons/01-btp/04-multi-tenancy.md) | BTP multi-tenancy model, tenant isolation patterns, XSUAA identity zones, CAP MTX, operational concerns |

### Module 2 · Kyma Runtime & Kubernetes

| # | Lesson | Topics |
|---|--------|--------|
| 2.1 | [Kyma Architecture & Core Components](lessons/02-kyma/01-kyma-architecture-core-components.md) | Kyma vs vanilla K8s, cluster architecture, Istio service mesh, API Gateway, eventing infrastructure |
| 2.2 | [Deploying Java Microservices on Kyma](lessons/02-kyma/02-deploying-java-microservices.md) | Containerizing Spring Boot/CAP apps, Helm charts, BTP service bindings, health probes, deployment commands |
| 2.3 | [Serverless Functions](lessons/02-kyma/03-serverless-functions.md) | Functions vs microservices decision framework, Function CRD, BTP service binding, event triggers |
| 2.4 | [Eventing & Extension Patterns](lessons/02-kyma/04-eventing-extension-patterns.md) | Side-by-side extensions, S/4HANA event integration, extension patterns, resilience patterns |

### Module 3 · CAP (Cloud Application Programming Model)

| # | Lesson | Topics |
|---|--------|--------|
| 3.1 | [CAP Java Architecture & Core Concepts](lessons/03-cap/01-cap-java-architecture.md) | Architecture overview, CDS compilation pipeline, event handler model, CQN query API, project structure |
| 3.2 | [CAP Security & Authorization](lessons/03-cap/02-cap-security.md) | Authentication architecture, CDS-based authorization, programmatic auth, multi-tenant security, mock users |
| 3.3 | [CAP Remote Services & Mashups](lessons/03-cap/03-cap-remote-services.md) | Remote service architecture, importing external APIs, destination config, consuming remote services, resilience |
| 3.4 | [CAP Advanced Patterns & Production Readiness](lessons/03-cap/04-cap-advanced-patterns.md) | Draft handling, localization, temporal data & audit logging, deployment topologies, performance optimization |

### Module 4 · Cross-Cutting Concerns

| # | Lesson | Topics |
|---|--------|--------|
| 4.1 | [CI/CD for SAP BTP Applications](lessons/04-cross-cutting/01-cicd-for-btp.md) | CI/CD landscape, MTA build pipelines, Helm + Docker for Kyma, quality gates, deployment strategies |
| 4.2 | [Observability & Operations](lessons/04-cross-cutting/02-observability-operations.md) | Cloud Logging Service, metrics & monitoring, distributed tracing, alerting & dashboards, incident response |
| 4.3 | [SAP BTP for Spring Boot Developers](lessons/04-cross-cutting/03-spring-boot-migration-guide.md) | Spring MVC → CAP, JPA → CDS/CQN, Spring Security → XSUAA, configuration & messaging, testing patterns |

---

## How to Use This Repository

1. **Follow the modules in order** — later lessons build on concepts from earlier ones.
2. **Each lesson includes:**
   - A table of contents for quick navigation
   - ASCII architecture diagrams and code examples
   - Comparison tables mapping familiar concepts (Spring Boot, AWS, plain K8s) to BTP equivalents
   - A "Top 5 Pitfalls" section with common mistakes
   - "What to Learn Next" pointers
3. **Start with Module 1** if you're new to BTP. If you already know BTP basics, jump to Module 2 (Kyma) or Module 3 (CAP) based on your project needs.

## Prerequisites

- Solid Java / Spring Boot experience
- Basic understanding of cloud platforms (any of AWS, Azure, GCP)
- Familiarity with containers and Kubernetes concepts (for Module 2)
- Access to an SAP BTP trial or enterprise account (for hands-on practice)

## License

This material is provided for learning purposes.