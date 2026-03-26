# Lesson 2.2 — Deploying Java Microservices on Kyma

## Table of Contents

- [1. Containerizing a Spring Boot / CAP Java Application](#1-containerizing-a-spring-boot-cap-java-application)
- [2. Helm Chart Structure](#2-helm-chart-structure)
- [3. Service Credential Consumption](#3-service-credential-consumption)
- [4. Health Probes and Graceful Shutdown](#4-health-probes-and-graceful-shutdown)
- [5. Deployment Commands](#5-deployment-commands)
- [Top 5 Pitfalls](#top-5-pitfalls)
- [What to Learn Next](#what-to-learn-next)

---

> **Summary:** Deploying Java applications on Kyma requires Docker containerization, Helm chart packaging, BTP service binding via the BTP Operator, and production-hardening with health probes, resource limits, and HPA. This lesson walks through the full deployment lifecycle — from Dockerfile to production-ready Helm chart — with comparisons to Cloud Foundry deployment patterns.

---

## 1. Containerizing a Spring Boot / CAP Java Application

### Dockerfile (Multi-Stage Build)

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
COPY srv/ srv/
COPY db/ db/
RUN mvn clean package -DskipTests -B

# Stage 2: Runtime
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# Security: run as non-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=builder /app/srv/target/*.jar app.jar

EXPOSE 8080
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### Key Differences from Cloud Foundry

| Aspect | Cloud Foundry | Kyma |
|---|---|---|
| Container | Buildpack creates it for you | You write the Dockerfile |
| Base image | SAP JVM / SapMachine (in buildpack) | Your choice (Temurin, SapMachine, etc.) |
| Memory tuning | CF sets `-Xmx` from memory quota | You set via `JAVA_OPTS` and K8s resource limits |
| Service credentials | `VCAP_SERVICES` env var | Mounted K8s Secrets (files or env vars) |
| Port binding | `$PORT` env var (assigned by CF) | You control (typically 8080) |

### Building and Pushing the Image

```bash
# Build the image
docker build -t my-registry.example.com/my-app:1.0.0 .

# Push to container registry (SAP BTP uses your own registry)
docker push my-registry.example.com/my-app:1.0.0
```

---

## 2. Helm Chart Structure

```
my-app-chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── apirule.yaml
│   ├── service-instance.yaml    # BTP ServiceInstance
│   ├── service-binding.yaml     # BTP ServiceBinding
│   ├── hpa.yaml
│   └── _helpers.tpl
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-java-app
description: SAP BTP Java microservice on Kyma
type: application
version: 1.0.0
appVersion: "1.0.0"
```

### values.yaml

```yaml
replicaCount: 2

image:
  repository: my-registry.example.com/my-app
  tag: "1.0.0"
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: registry-secret

service:
  port: 8080

resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilization: 70

xsuaa:
  serviceOfferingName: xsuaa
  servicePlanName: application
  parameters:
    xsappname: my-app
    tenant-mode: dedicated

hana:
  serviceOfferingName: hana
  servicePlanName: hdi-shared

apiRule:
  host: my-app
```

### Deployment Template

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-java-app.fullname" . }}
  labels:
    {{- include "my-java-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-java-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-java-app.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
              protocol: TCP
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "cloud"
          volumeMounts:
            - name: xsuaa-secret
              mountPath: /etc/secrets/sapcp/xsuaa/my-xsuaa
              readOnly: true
            - name: hana-secret
              mountPath: /etc/secrets/sapcp/hana/my-hana
              readOnly: true
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: {{ .Values.service.port }}
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: {{ .Values.service.port }}
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: {{ .Values.service.port }}
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30  # 10s + 30*5s = up to 160s startup time
      volumes:
        - name: xsuaa-secret
          secret:
            secretName: my-xsuaa-secret
        - name: hana-secret
          secret:
            secretName: my-hana-secret
```

### BTP Service Binding Templates

```yaml
# templates/service-instance.yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceInstance
metadata:
  name: my-xsuaa
spec:
  serviceOfferingName: {{ .Values.xsuaa.serviceOfferingName }}
  servicePlanName: {{ .Values.xsuaa.servicePlanName }}
  parameters:
    {{- toYaml .Values.xsuaa.parameters | nindent 4 }}
---
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: my-xsuaa-binding
spec:
  serviceInstanceName: my-xsuaa
  secretName: my-xsuaa-secret
```

---

## 3. Service Credential Consumption

### SAP BTP Credential Mounting Convention

SAP libraries (Cloud SDK, spring-xsuaa, CAP) expect credentials at:
```
/etc/secrets/sapcp/<service-name>/<instance-name>/
```

The mounted secret files contain JSON credentials. Example file tree:
```
/etc/secrets/sapcp/xsuaa/my-xsuaa/
├── clientid
├── clientsecret
├── url
├── xsappname
└── ...  (one file per credential key)
```

### Spring Boot Configuration for Kyma

```yaml
# application-cloud.yaml (activated by SPRING_PROFILES_ACTIVE=cloud)
sap:
  cloud:
    security:
      xsuaa:
        credential-type: binding-secret
```

CAP Java automatically discovers credentials from mounted secrets when running on Kyma.

---

## 4. Health Probes and Graceful Shutdown

### Spring Boot Actuator Configuration

```yaml
# application.yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true
      show-details: always
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true

server:
  shutdown: graceful
  
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

### Graceful Shutdown Flow

```
1. K8s sends SIGTERM to container
2. K8s removes pod from Service endpoints (no new traffic)
3. Spring Boot stops accepting new requests
4. Spring Boot waits for in-flight requests (up to 30s)
5. Spring Boot closes connections and shuts down
6. K8s waits for terminationGracePeriodSeconds (default 30s)
7. If still running, K8s sends SIGKILL
```

```yaml
# In deployment spec
spec:
  terminationGracePeriodSeconds: 45  # > spring shutdown timeout
```

---

## 5. Deployment Commands

```bash
# Create namespace with Istio injection
kubectl create namespace my-app
kubectl label namespace my-app istio-injection=enabled

# Create image pull secret
kubectl create secret docker-registry registry-secret \
  --docker-server=my-registry.example.com \
  --docker-username=user \
  --docker-password=password \
  --namespace my-app

# Deploy with Helm
helm install my-app ./my-app-chart \
  --namespace my-app \
  --set image.tag=1.0.0

# Upgrade
helm upgrade my-app ./my-app-chart \
  --namespace my-app \
  --set image.tag=1.1.0

# Check status
kubectl get pods -n my-app
kubectl get servicebindings -n my-app
kubectl get apirules -n my-app
```

---

## Top 5 Pitfalls

1. **Running as root in containers.** Always create a non-root user in your Dockerfile. Kyma may enforce PodSecurityPolicies.
2. **Not setting `-XX:MaxRAMPercentage`.** Without it, the JVM may try to use more memory than the container limit, causing OOMKill.
3. **Missing startup probes for Java apps.** Spring Boot can take 30-60s to start. Without startup probes, K8s kills the pod thinking it's unhealthy.
4. **Hardcoding secrets in values.yaml.** Use sealed secrets, external secrets operator, or pass secrets via `--set` at deploy time.
5. **Forgetting Istio sidecar memory overhead.** Each pod gets an Envoy proxy (~50-100MB). Factor this into resource requests.

---

## What to Learn Next

- **Lesson 2.1:** Kyma Architecture — understanding the platform components
- **Lesson 2.3:** Kyma Serverless Functions — lightweight alternatives to microservices
- **Lesson 4.1:** CI/CD — automating Docker builds and Helm deployments
- **Lesson 4.2:** Observability — Prometheus metrics, distributed tracing on Kyma