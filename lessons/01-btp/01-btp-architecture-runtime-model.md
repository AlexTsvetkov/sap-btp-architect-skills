# Lesson 1.1 — SAP BTP Architecture & Runtime Model

## Table of Contents

- [1. Multi-Environment Architecture](#1-multi-environment-architecture)
- [2. Subaccount, Directory, and Entitlement Hierarchy](#2-subaccount-directory-and-entitlement-hierarchy)
- [3. BTP Service Marketplace & Open Service Broker API](#3-btp-service-marketplace--open-service-broker-api)
- [4. BTP Connectivity: Cloud Connector & Destinations](#4-btp-connectivity-cloud-connector--destinations)
- [5. BTP Security Model — XSUAA Internals](#5-btp-security-model--xsuaa-internals)
- [Top 5 Pitfalls for Java Developers on BTP](#top-5-pitfalls-for-java-developers-on-btp)
- [What to Learn Next](#what-to-learn-next)

---

> **Summary:** SAP Business Technology Platform (BTP) is a PaaS offering that hosts three distinct runtime environments — Cloud Foundry, Kyma (Kubernetes), and ABAP — under a unified commercial and governance model. Understanding BTP's internal structure is critical for Java architects because it dictates how services are consumed, how security tokens flow, and how on-premise connectivity works. This lesson dissects BTP's architecture layer by layer, from the organizational hierarchy down to the VCAP_SERVICES binding mechanism.

---

## 1. Multi-Environment Architecture

BTP offers three runtime environments within a single subaccount. Each serves a different workload profile:

```
┌─────────────────────────────────────────────────────────────────┐
│                     SAP BTP Global Account                      │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      Subaccount                           │  │
│  │                                                           │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │  │
│  │  │ Cloud       │  │ Kyma        │  │ ABAP            │   │  │
│  │  │ Foundry     │  │ Runtime     │  │ Environment     │   │  │
│  │  │             │  │             │  │                 │   │  │
│  │  │ - Buildpacks│  │ - K8s pods  │  │ - ABAP stack    │   │  │
│  │  │ - Diego     │  │ - Istio     │  │ - Steampunk     │   │  │
│  │  │   cells     │  │ - Eventing  │  │ - RAP / CDS     │   │  │
│  │  │ - cf push   │  │ - Helm      │  │ - ADT (Eclipse) │   │  │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘   │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Cloud Foundry Environment

- **Runtime model:** Application instances run inside Diego cells (Linux containers managed by the Diego orchestrator). You deploy via `cf push` or MTA (`cf deploy`).
- **Scaling:** Horizontal scaling by instance count. Each instance is a separate container with its own memory/disk quota. No shared filesystem between instances.
- **Buildpacks:** SAP provides `sap_java_buildpack` (based on SAP JVM or SapMachine) and the community `java_buildpack`. SAP's buildpack includes built-in support for BTP service credentials injection.
- **Best for:** Stateless web applications, CAP applications, batch jobs, and traditional 12-factor apps.

### Kyma Environment

- **Runtime model:** A managed Kubernetes cluster with pre-installed modules (Istio, NATS Eventing, API Gateway, Serverless). You deploy via `kubectl apply`, Helm charts, or Kyma-specific CRDs.
- **Scaling:** Full Kubernetes HPA, VPA, and pod-level resource management. You control CPU/memory requests and limits directly.
- **SAP Additions:** BTP Operator (manages service instances/bindings as K8s CRDs), Kyma API Gateway (APIRule CRD), Kyma Eventing (Subscription CRD).
- **Best for:** Microservices architectures, event-driven extensions, workloads requiring sidecar patterns, teams already invested in Kubernetes.

### ABAP Environment (Steampunk)

- **Runtime model:** A full ABAP application server instance running in the cloud. You develop using the ABAP RESTful Application Programming Model (RAP) via ABAP Development Tools (ADT) in Eclipse.
- **Best for:** Extending S/4HANA with ABAP-native code, building reusable ABAP APIs, teams with deep ABAP expertise.

### When to Use Each

| Criteria | Cloud Foundry | Kyma | ABAP |
|---|---|---|---|
| Team skills | Java/Node.js, CF CLI | Java + Kubernetes | ABAP, RAP |
| Deployment model | `cf push` / MTA | Helm / kubectl | ADT / abapGit |
| Sidecar support | No | Yes (Istio) | No |
| Event-driven extensions | Via Event Mesh client lib | Native Subscription CRD | Enterprise Event Enablement |
| Container control | Low (buildpack-managed) | Full (custom Dockerfiles) | None |
| Cold start | Low (~2-5s) | Varies (depends on image) | N/A (always running) |

> **Pitfall for Spring Boot devs:** Cloud Foundry is *not* Docker. You don't write a Dockerfile for CF — you use buildpacks. If you need custom container images, use Kyma.

---

## 2. Subaccount, Directory, and Entitlement Hierarchy

BTP's organizational model has four levels:

```
Global Account
├── Directory (optional grouping)
│   ├── Subaccount (dev)
│   ├── Subaccount (staging)
│   └── Subaccount (prod)
└── Subaccount (sandbox)
```

### Key Concepts

- **Global Account:** The commercial boundary. All entitlements (service quotas) are assigned here. You get a global account when you purchase BTP.
- **Directory:** Optional organizational grouping. Useful for managing entitlements and access across multiple subaccounts. Directories can contain sub-directories.
- **Subaccount:** The isolation boundary. Each subaccount gets its own:
  - Identity zone (XSUAA tenant)
  - Cloud Foundry org (if CF is enabled)
  - Kyma cluster (if Kyma is enabled)
  - Network isolation
  - Service instances
- **Entitlements:** Service plans are allocated from Global Account → Directory → Subaccount. An entitlement grants the *right* to create service instances of a specific plan.

### How Quota Works

```
Global Account: 10 units of HANA Cloud (memory)
├── Directory "Production"
│   ├── Subaccount "prod-eu" → entitled: 6 units
│   └── Subaccount "prod-us" → entitled: 4 units
└── Directory "Non-Production"
    ├── Subaccount "dev" → entitled: 2 units (from remaining pool)
    └── Subaccount "staging" → entitled: 2 units
```

Some services use **numeric quotas** (e.g., HANA Cloud memory GB, runtime memory MB). Others use **boolean entitlements** (entitled or not, e.g., Destination service plan "lite").

> **Pitfall:** Entitlements are not the same as service instances. You must first entitle a subaccount to a service plan, *then* create a service instance in that subaccount. Forgetting the entitlement step is one of the most common BTP onboarding errors.

---

## 3. BTP Service Marketplace & Open Service Broker API

### How Service Brokers Work

BTP's service marketplace is built on the [Open Service Broker API](https://www.openservicebrokerapi.org/) (OSB API). Every service in the marketplace (HANA Cloud, XSUAA, Destination, Event Mesh, etc.) is backed by a service broker.

```
┌──────────────┐     OSB API      ┌──────────────────┐
│  cf / BTP    │ ────────────────→ │  Service Broker  │
│  Controller  │  PUT /v2/        │  (e.g., XSUAA    │
│              │  service_instances│   broker)        │
│              │  /v2/            │                  │
│              │  service_bindings │  Provisions the  │
│              │                  │  actual resource  │
└──────────────┘                  └──────────────────┘
```

The OSB API flow:

1. **Catalog:** `GET /v2/catalog` — broker advertises its service plans
2. **Provision:** `PUT /v2/service_instances/:id` — creates the resource (e.g., an XSUAA tenant, a HANA schema)
3. **Bind:** `PUT /v2/service_instances/:id/service_bindings/:bid` — creates credentials for an app to consume the resource
4. **Unbind / Deprovision:** Cleanup

### Service Binding Injection

When you bind a service to a CF application, the platform injects credentials via the `VCAP_SERVICES` environment variable:

```json
{
  "VCAP_SERVICES": {
    "xsuaa": [
      {
        "label": "xsuaa",
        "plan": "application",
        "name": "my-xsuaa",
        "credentials": {
          "clientid": "sb-my-app!t12345",
          "clientsecret": "SECRET",
          "url": "https://mysubaccount.authentication.eu10.hana.ondemand.com",
          "uaadomain": "authentication.eu10.hana.ondemand.com",
          "verificationkey": "-----BEGIN PUBLIC KEY-----\n...",
          "xsappname": "my-app!t12345",
          "identityzone": "mysubaccount",
          "tenantid": "abcd-1234-..."
        }
      }
    ],
    "hana": [
      {
        "label": "hana",
        "plan": "hdi-shared",
        "name": "my-hdi",
        "credentials": {
          "host": "abcdef.hana.prod-eu10.hanacloud.ondemand.com",
          "port": "443",
          "user": "HDI_CONTAINER_USER",
          "password": "...",
          "schema": "MY_HDI_CONTAINER_SCHEMA",
          "url": "jdbc:sap://..."
        }
      }
    ]
  }
}
```

**In Kyma**, the BTP Operator creates Kubernetes secrets from service bindings and mounts them into pods as files:

```yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: my-xsuaa-binding
spec:
  serviceInstanceName: my-xsuaa
  secretName: my-xsuaa-secret  # K8s Secret created automatically
```

The secret is then mounted as a volume or injected as env vars into your deployment.

> **Pitfall for AWS/Azure devs:** Unlike AWS where you use IAM roles and instance metadata, BTP services always require explicit binding. There's no implicit credential resolution — you *must* create a service binding and parse VCAP_SERVICES (CF) or mounted secrets (Kyma).

---

## 4. BTP Connectivity: Cloud Connector & Destinations

### Cloud Connector Architecture

The Cloud Connector (SCC) enables access to on-premise systems from BTP without opening inbound firewall ports. It works as a reverse-invoke tunnel:

```
┌─────────────────────┐          ┌─────────────────────┐
│   On-Premise         │          │    SAP BTP           │
│                      │          │                      │
│  ┌────────────────┐  │  TLS    │  ┌────────────────┐  │
│  │ Cloud          │──┼────────→│  │ Connectivity    │  │
│  │ Connector      │  │ outbound│  │ Service         │  │
│  │ (Java process) │  │ only    │  │                 │  │
│  └───────┬────────┘  │          │  └────────┬───────┘  │
│          │           │          │           │          │
│  ┌───────▼────────┐  │          │  ┌────────▼───────┐  │
│  │ SAP S/4HANA    │  │          │  │ Your CF/Kyma   │  │
│  │ (RFC, HTTP,    │  │          │  │ Application    │  │
│  │  LDAP)         │  │          │  │                │  │
│  └────────────────┘  │          │  └────────────────┘  │
└─────────────────────┘          └─────────────────────┘
```

Key points:
- SCC opens an **outbound** TLS tunnel to BTP's Connectivity Service. No inbound ports needed.
- SCC runs as a Java process on-premise (on the same network as your backend systems).
- Virtual host mapping: SCC maps `virtual-host:virtual-port` to actual `on-prem-host:port`, hiding internal topology.
- **High availability:** Deploy two SCC instances in master-shadow configuration.

### Principal Propagation

Principal propagation forwards the end-user identity from BTP through the Cloud Connector to the on-premise system:

```
User → BTP App (JWT from XSUAA)
         │
         ▼
    Connectivity Service (exchanges JWT for short-lived X.509 cert)
         │
         ▼
    Cloud Connector (presents client certificate via mutual TLS)
         │
         ▼
    On-Premise SAP System (maps X.509 subject to SAP user via trust config)
```

This requires:
1. XSUAA configured to include user attributes in the JWT
2. Cloud Connector configured with a system certificate trusted by the on-premise system
3. On-premise system's trust store configured to accept the SCC's CA certificate
4. User mapping rules (e.g., email → SAP user ID)

### Destination Service

The Destination Service stores connection details (URL, authentication method, proxy settings) as named configurations:

```json
{
  "Name": "S4HANA_ONPREM",
  "Type": "HTTP",
  "URL": "http://virtual-host:443",
  "ProxyType": "OnPremise",
  "Authentication": "PrincipalPropagation",
  "sap-client": "100"
}
```

Your application resolves destinations at runtime:

```java
// Using SAP Cloud SDK
HttpDestination destination = DestinationAccessor
    .getDestination("S4HANA_ONPREM")
    .asHttp();

// All requests via this destination go through Cloud Connector
List<BusinessPartner> partners = new DefaultBusinessPartnerService()
    .getAllBusinessPartner()
    .executeRequest(destination);
```

> **Pitfall:** Destinations are cached. If you rotate credentials in the destination configuration, running application instances may still use stale credentials until the cache expires (default 5 minutes). In production, use `CacheExpirationStrategy` or restart instances after credential rotation.

---

## 5. BTP Security Model — XSUAA Internals

### XSUAA Overview

XSUAA (Extended Services for UAA) is BTP's OAuth2 authorization server, based on Cloud Foundry's UAA (User Account and Authentication). It is the central component for:
- Issuing JWT tokens
- Managing OAuth2 clients (service instances)
- Defining and checking scopes/authorities
- Multi-tenant identity zone isolation

### JWT Token Structure

A typical XSUAA JWT token:

```json
{
  "jti": "abc123...",
  "ext_attr": {
    "enhancer": "XSUAA",
    "subaccountid": "tenant-guid",
    "zdn": "mysubaccount"
  },
  "sub": "user-guid",
  "scope": [
    "my-app!t12345.Admin",
    "my-app!t12345.Viewer",
    "openid"
  ],
  "client_id": "sb-my-app!t12345",
  "cid": "sb-my-app!t12345",
  "azp": "sb-my-app!t12345",
  "grant_type": "authorization_code",
  "user_id": "user-guid",
  "origin": "sap.default",
  "user_name": "john.doe@example.com",
  "email": "john.doe@example.com",
  "given_name": "John",
  "family_name": "Doe",
  "zid": "tenant-guid",
  "aud": ["my-app!t12345", "openid"],
  "iss": "https://mysubaccount.authentication.eu10.hana.ondemand.com/oauth/token",
  "exp": 1700000000,
  "iat": 1699996400
}
```

Key claims:
- `zid` — tenant ID (identity zone). Critical for multi-tenancy.
- `scope` — array of granted scopes. Format: `<xsappname>.<scope-name>`
- `ext_attr.subaccountid` — the subaccount that owns this tenant
- `origin` — identity provider origin (e.g., `sap.default`, `corporate-idp`)

### Scope and Authority Mapping

Defined in `xs-security.json`:

```json
{
  "xsappname": "my-app",
  "tenant-mode": "dedicated",
  "scopes": [
    { "name": "$XSAPPNAME.Admin", "description": "Administrator" },
    { "name": "$XSAPPNAME.Viewer", "description": "Read-only access" }
  ],
  "role-templates": [
    {
      "name": "AdminRole",
      "scope-references": ["$XSAPPNAME.Admin", "$XSAPPNAME.Viewer"]
    },
    {
      "name": "ViewerRole",
      "scope-references": ["$XSAPPNAME.Viewer"]
    }
  ]
}
```

The mapping chain:

```
xs-security.json → Scopes → Role Templates → Role Collections → Users/Groups
                    (code)    (deploy-time)    (BTP cockpit)     (IDP)
```

1. **Scopes** are defined in `xs-security.json` and deployed with the XSUAA service instance
2. **Role Templates** group scopes and are also in `xs-security.json`
3. **Role Collections** are created in BTP Cockpit (or via API) and reference role templates
4. **Users or IDP Groups** are assigned to role collections

### OAuth2 Flows

| Flow | Use Case | Token Endpoint |
|---|---|---|
| `authorization_code` | End-user login via browser (Fiori, approuter) | `/oauth/token` with `code` |
| `client_credentials` | Service-to-service communication (no user context) | `/oauth/token` with `client_id` + `client_secret` |
| `password` (deprecated) | Testing only — avoid in production | `/oauth/token` with `username` + `password` |
| `urn:ietf:params:oauth:grant-type:jwt-bearer` | User token exchange (propagate user identity between services) | `/oauth/token` with `assertion` (existing JWT) |

**User token exchange** is the most misunderstood flow. It's used when:
- Service A receives a user request with a JWT
- Service A needs to call Service B on behalf of the same user
- Service A exchanges its JWT for a new JWT scoped to Service B

```java
// Using SAP Cloud SDK for user token exchange
HttpDestination dest = DestinationAccessor
    .getDestination("SERVICE_B")
    .asHttp();
// Cloud SDK automatically handles token exchange if destination
// is configured with Authentication=OAuth2UserTokenExchange
```

> **Pitfall from Spring Security background:** XSUAA scopes are *not* the same as Spring Security authorities by default. The `spring-xsuaa` library maps scopes to Spring authorities by stripping the `xsappname` prefix. So `my-app!t12345.Admin` becomes authority `Admin`. If you're using `@PreAuthorize("hasAuthority('Admin')")`, this works. But if you compare raw JWT scopes, you'll see the full qualified name.

---

## Top 5 Pitfalls for Java Developers on BTP

1. **Confusing entitlements with service instances.** You must entitle a subaccount before you can provision services.
2. **Using Docker assumptions in Cloud Foundry.** CF uses buildpacks, not Dockerfiles. Use Kyma if you need custom container images.
3. **Ignoring VCAP_SERVICES parsing.** Libraries like `@sap/xssec` and `spring-xsuaa` handle this, but custom code that reads env vars directly often breaks when multiple bindings of the same service type exist.
4. **Not configuring principal propagation end-to-end.** Missing any step in the chain (XSUAA → Connectivity Service → Cloud Connector → on-prem trust) results in opaque `403 Forbidden` errors.
5. **Assuming scopes are globally unique.** Scopes are namespaced by `xsappname`. In multi-tenant scenarios, the `xsappname` includes a tenant-specific suffix.

---

## What to Learn Next

- **Lesson 1.2:** HANA Cloud deep dive — HDI containers, multi-model engine, performance tuning
- **Lesson 1.4:** Multi-tenancy on BTP — SaaS provisioning, tenant-aware routing, XSUAA identity zones
- **Lesson 3.1:** CAP Java architecture — how CAP abstracts over BTP services
- SAP Cloud SDK documentation: how the SDK integrates with Destination and Connectivity services