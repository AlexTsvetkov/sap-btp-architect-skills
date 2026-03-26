# Lesson 2.1 — Kyma Architecture & Core Components

## Table of Contents

- [1. Kyma vs Vanilla Kubernetes](#1-kyma-vs-vanilla-kubernetes)
- [2. Kyma Cluster Architecture](#2-kyma-cluster-architecture)
- [3. Istio Service Mesh in Kyma](#3-istio-service-mesh-in-kyma)
- [4. Kyma API Gateway](#4-kyma-api-gateway)
- [5. Kyma Eventing](#5-kyma-eventing)
- [Top 5 Pitfalls](#top-5-pitfalls)
- [What to Learn Next](#what-to-learn-next)

---

> **Summary:** SAP Kyma Runtime is a managed Kubernetes offering on BTP that adds an opinionated layer of components — Istio service mesh, NATS-based eventing, API Gateway, and the BTP Operator — on top of a standard K8s cluster. This lesson covers what Kyma adds versus vanilla Kubernetes, the modular architecture, Istio internals, API Gateway configuration, and eventing infrastructure.

---

## 1. Kyma vs Vanilla Kubernetes

### What Kyma Adds

```
┌──────────────────────────────────────────────────────────┐
│                    Kyma Runtime                           │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  SAP-Managed Layer                                  │  │
│  │  ┌──────────┐ ┌──────────┐ ┌───────────────────┐   │  │
│  │  │ Istio    │ │ Kyma API │ │ BTP Operator      │   │  │
│  │  │ Service  │ │ Gateway  │ │ (ServiceInstance/  │   │  │
│  │  │ Mesh     │ │ (APIRule)│ │  ServiceBinding)   │   │  │
│  │  └──────────┘ └──────────┘ └───────────────────┘   │  │
│  │  ┌──────────┐ ┌──────────┐ ┌───────────────────┐   │  │
│  │  │ NATS     │ │ Serverless│ │ Lifecycle Manager │   │  │
│  │  │ Eventing │ │ Functions │ │ (module install)  │   │  │
│  │  └──────────┘ └──────────┘ └───────────────────┘   │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Standard Kubernetes                                │  │
│  │  Pods, Deployments, Services, ConfigMaps, Secrets   │  │
│  │  Namespaces, RBAC, HPA, PDB, Ingress               │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

| Component | Vanilla K8s | Kyma |
|---|---|---|
| Service mesh | Install Istio yourself | Pre-installed, SAP-managed Istio |
| API Gateway | Ingress controller (nginx, etc.) | APIRule CRD → Istio VirtualService + Oathkeeper |
| Eventing | Install NATS/Kafka yourself | Kyma Eventing (NATS or Event Mesh backend) |
| BTP services | Manual credential management | BTP Operator: ServiceInstance/ServiceBinding CRDs |
| Serverless | Install Knative/OpenFaaS | Kyma Functions (Node.js, Python) |
| Certificate mgmt | cert-manager manual setup | Integrated cert management |
| DNS | External-DNS manual setup | Managed DNS for *.kyma.ondemand.com |

### Kyma Modules System

Kyma uses a modular installation model. Modules are optional components enabled via the Kyma CR:

```yaml
apiVersion: operator.kyma-project.io/v1beta2
kind: Kyma
metadata:
  name: default
  namespace: kyma-system
spec:
  modules:
    - name: istio
    - name: api-gateway
    - name: eventing
    - name: btp-operator
    - name: serverless       # optional
    - name: telemetry        # optional
```

You can enable/disable modules without affecting the core cluster.

---

## 2. Kyma Cluster Architecture

### System Namespaces

```
kyma-system/          — Kyma core components, lifecycle manager
istio-system/         — Istio control plane (istiod, gateways)
eventing-system/      — NATS server, eventing controller
serverless-system/    — Serverless function controller
btp-operator/         — BTP Operator controller
kube-system/          — Standard K8s system components
```

> **Rule:** Never deploy your workloads into system namespaces. Create dedicated namespaces for your applications.

### BTP Operator for Service Consumption

The BTP Operator lets you manage BTP service instances and bindings as Kubernetes resources:

```yaml
# Create a service instance
apiVersion: services.cloud.sap.com/v1
kind: ServiceInstance
metadata:
  name: my-xsuaa
  namespace: my-app
spec:
  serviceOfferingName: xsuaa
  servicePlanName: application
  parameters:
    xsappname: my-app
    tenant-mode: dedicated
    oauth2-configuration:
      redirect-uris:
        - "https://my-app.kyma.ondemand.com/**"
---
# Create a binding (generates K8s Secret)
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: my-xsuaa-binding
  namespace: my-app
spec:
  serviceInstanceName: my-xsuaa
  secretName: my-xsuaa-secret
```

The BTP Operator communicates with the BTP Service Manager API to provision/bind services and stores credentials in Kubernetes Secrets.

---

## 3. Istio Service Mesh in Kyma

### Sidecar Injection

Istio injects an Envoy proxy sidecar into every pod in labeled namespaces:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    istio-injection: enabled  # enables automatic sidecar injection
```

Each pod gets two containers:
```
Pod: my-service-abc123
├── my-service (your Java app, port 8080)
└── istio-proxy (Envoy sidecar, intercepts all traffic)
```

### Mutual TLS (mTLS)

Istio enables automatic mTLS between all pods in the mesh:

```
Pod A (my-service)          Pod B (other-service)
┌─────────────────┐        ┌─────────────────┐
│ App container    │        │ App container    │
│ (plain HTTP)     │        │ (plain HTTP)     │
│       │          │        │       ▲          │
│       ▼          │        │       │          │
│ Envoy sidecar ───┼──mTLS──┼─→ Envoy sidecar │
│ (encrypts)       │        │ (decrypts)       │
└─────────────────┘        └─────────────────┘
```

Your application code uses plain HTTP; Istio handles encryption transparently.

### Traffic Management

```yaml
# VirtualService for traffic splitting (canary deployment)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-service
spec:
  hosts:
    - my-service
  http:
    - route:
        - destination:
            host: my-service
            subset: v1
          weight: 90
        - destination:
            host: my-service
            subset: v2
          weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-service
spec:
  host: my-service
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

---

## 4. Kyma API Gateway

### APIRule CRD

The APIRule is Kyma's abstraction for exposing services externally with authentication:

```yaml
apiVersion: gateway.kyma-project.io/v1beta1
kind: APIRule
metadata:
  name: my-service-api
  namespace: my-app
spec:
  host: my-service.kyma.ondemand.com
  service:
    name: my-service
    port: 8080
  gateway: kyma-system/kyma-gateway
  rules:
    # Public endpoint (no auth)
    - path: /health
      methods: ["GET"]
      accessStrategies:
        - handler: noop
    # JWT-protected endpoint
    - path: /api/.*
      methods: ["GET", "POST", "PUT", "DELETE"]
      accessStrategies:
        - handler: jwt
          config:
            jwks_urls:
              - "https://mysubaccount.authentication.eu10.hana.ondemand.com/token_keys"
            trusted_issuers:
              - "https://mysubaccount.authentication.eu10.hana.ondemand.com"
```

Under the hood, APIRule creates:
1. An Istio **VirtualService** for routing
2. An Ory **Oathkeeper Rule** for authentication/authorization

### CORS Configuration

```yaml
spec:
  corsPolicy:
    allowOrigins:
      - exact: "https://my-frontend.example.com"
    allowMethods: ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
    allowHeaders: ["Authorization", "Content-Type"]
    allowCredentials: true
```

---

## 5. Kyma Eventing

### NATS vs Event Mesh Backend

Kyma Eventing supports two backends:

| Feature | NATS (default) | Event Mesh |
|---|---|---|
| Deployment | In-cluster NATS server | BTP managed service |
| Durability | In-memory (limited) | Persistent queues |
| S/4HANA integration | Via Event Mesh bridge | Native |
| Scale | Single cluster | Cross-region |
| Cost | Included in Kyma | Additional BTP service cost |

### Subscription CRD

```yaml
apiVersion: eventing.kyma-project.io/v1alpha2
kind: Subscription
metadata:
  name: bp-changed-sub
  namespace: my-app
spec:
  source: ""
  types:
    - sap.s4.beh.businesspartner.changed.v1
  sink: http://my-service.my-app.svc.cluster.local:8080/events
  config:
    maxInFlightMessages: "10"
```

The eventing controller ensures events matching the specified types are delivered to the sink URL via HTTP POST with CloudEvents format.

### Event Type Versioning

Follow the SAP naming convention:
```
<prefix>.<domain>.<business-object>.<operation>.v<version>

Examples:
sap.s4.beh.businesspartner.changed.v1
my.company.order.validated.v1
my.company.order.validated.v2  # new version with breaking changes
```

---

## Top 5 Pitfalls

1. **Forgetting `istio-injection: enabled` label on namespaces.** Without it, pods don't get sidecars and miss mTLS, traffic management, and observability.
2. **Using Ingress instead of APIRule.** Kyma's API Gateway (APIRule) integrates with Istio and Oathkeeper. Standard K8s Ingress won't work correctly on Kyma.
3. **Not configuring resource requests/limits.** Istio sidecars add ~50MB memory per pod. Account for this in your resource planning.
4. **Assuming NATS eventing is durable.** Default NATS backend is in-memory. For production event-driven architectures, use Event Mesh backend.
5. **Deploying into system namespaces.** Kyma upgrades may overwrite your resources. Always use dedicated namespaces.

---

## What to Learn Next

- **Lesson 2.2:** Deploying Java Microservices on Kyma — Dockerfiles, Helm charts, service bindings
- **Lesson 2.3:** Kyma Serverless Functions — when and how to use them
- **Lesson 2.4:** Kyma Eventing & Extension Patterns — S/4HANA side-by-side extensions
- **Lesson 1.1:** BTP Architecture — how Kyma fits in the BTP landscape