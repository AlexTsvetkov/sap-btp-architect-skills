# Lesson 4.1 — CI/CD for SAP BTP Applications

## Table of Contents

- [1. CI/CD Landscape for SAP BTP](#1-cicd-landscape-for-sap-btp)
- [2. MTA Build Pipeline](#2-mta-build-pipeline)
- [3. Helm + Docker Pipeline for Kyma](#3-helm--docker-pipeline-for-kyma)
- [4. Quality Gates](#4-quality-gates)
- [5. Deployment Strategies](#5-deployment-strategies)
- [Top 5 Pitfalls](#top-5-pitfalls)
- [What to Learn Next](#what-to-learn-next)

---

> **Summary:** CI/CD for SAP BTP applications differs significantly from standard Java pipelines because of the MTA build tooling, HDI database artifact deployment, BTP-specific credential injection, and the dual deployment targets (Cloud Foundry and Kyma). This lesson covers the full pipeline from commit to production, including MTA builds, Docker/Helm workflows for Kyma, quality gates tailored to CAP projects, and safe deployment strategies like blue-green on CF and canary releases on Kyma.

---

## 1. CI/CD Landscape for SAP BTP

### Tool Comparison

| Capability | SAP CI/CD Service | GitHub Actions | Jenkins (project "Piper") |
|---|---|---|---|
| Hosting | SAP-managed (BTP) | GitHub-managed / self-hosted | Self-hosted |
| MTA build support | Built-in | Via `mbt` CLI in container | Via SAP Piper library |
| CF deployment | Built-in `cf deploy` | Via `cf` CLI action | Via Piper steps |
| Kyma deployment | Limited | Full (kubectl/helm) | Via Piper steps |
| Secrets management | SAP Credential Store | GitHub Secrets | Jenkins Credentials |
| Custom pipeline logic | Limited (predefined stages) | Full (YAML workflow) | Full (Groovy/YAML) |
| BTP integration | Native | Manual setup | Via Piper defaults |
| Cost | Included in BTP | Free for public repos | Self-managed |

### When to Use Each

```
┌─────────────────────────────────────────────────────────────────┐
│                    CI/CD Decision Tree                           │
│                                                                  │
│  Need full pipeline customization?                               │
│  ├── Yes → GitHub Actions or Jenkins                             │
│  │   ├── Already using GitHub? → GitHub Actions                  │
│  │   └── Enterprise Jenkins infra? → Jenkins + SAP Piper         │
│  └── No → SAP CI/CD Service                                     │
│      └── Simple CAP app with standard stages?                    │
│          ├── Yes → SAP CI/CD Service (fastest setup)             │
│          └── No (multi-target Kyma + CF) → GitHub Actions        │
└─────────────────────────────────────────────────────────────────┘
```

### SAP CI/CD Service — Quick Setup

The SAP CI/CD Service provides predefined pipelines for CAP applications. Configuration is minimal:

```yaml
# .pipeline/config.yml (SAP CI/CD Service)
general:
  buildTool: "mta"
service:
  buildToolVersion: "MBTJ11N14"
stages:
  Build:
    mavenExecuteStaticCodeChecks: true
  Acceptance:
    cloudFoundryDeploy: true
    cfOrg: "my-org"
    cfSpace: "dev"
    mtarFilePath: "mta_archives/my-app.mtar"
  Release:
    cloudFoundryDeploy: true
    cfOrg: "my-org"
    cfSpace: "prod"
```

> **Limitation:** SAP CI/CD Service does not natively support Kyma deployments. If you target Kyma, use GitHub Actions or Jenkins.

---

## 2. MTA Build Pipeline

### MTA Archive Structure

The Multi-Target Application (MTA) is SAP's deployment packaging format. The `mbt build` command produces an `.mtar` archive:

```
my-app.mtar
├── META-INF/
│   └── MANIFEST.MF
│   └── mtad.yaml        # Deployment descriptor (generated from mta.yaml)
├── my-srv/               # Java backend (JAR/WAR)
├── my-db-deployer/       # HDI deployer (Node.js module with CDS artifacts)
├── my-app-content/       # Fiori UI (zipped)
└── my-mtx-sidecar/       # MTX sidecar (if multi-tenant)
```

### mta.yaml — Module Types

```yaml
_schema-version: "3.1"
ID: my-cap-app
version: 1.0.0

modules:
  # Java backend (CAP service)
  - name: my-srv
    type: java
    path: srv
    parameters:
      memory: 1024M
      disk-quota: 512M
      buildpack: sap_java_buildpack_jakarta
    build-parameters:
      builder: custom
      commands:
        - mvn -B clean package -DskipTests
      build-result: target/*.jar
    requires:
      - name: my-xsuaa
      - name: my-hdi
      - name: my-destination
    provides:
      - name: srv-api
        properties:
          url: ${default-url}

  # HDI database deployer
  - name: my-db-deployer
    type: hdb
    path: gen/db
    parameters:
      buildpack: nodejs_buildpack
    build-parameters:
      builder: custom
      commands:
        - npx cds build --for hana
    requires:
      - name: my-hdi

  # Fiori UI deployer
  - name: my-app-content
    type: com.sap.application.content
    path: app
    build-parameters:
      builder: custom
      commands:
        - npm run build
    requires:
      - name: my-html5-repo-host

resources:
  - name: my-xsuaa
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json

  - name: my-hdi
    type: com.sap.xs.hdi-container
    parameters:
      service: hana
      service-plan: hdi-shared

  - name: my-destination
    type: org.cloudfoundry.managed-service
    parameters:
      service: destination
      service-plan: lite
```

### GitHub Actions — Full MTA Build + CF Deploy

```yaml
# .github/workflows/cf-deploy.yml
name: Build and Deploy to Cloud Foundry

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  MBT_VERSION: "1.2.27"
  CF_API: "https://api.cf.eu10.hana.ondemand.com"

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'sapmachine'

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install MBT
        run: npm install -g mbt@${MBT_VERSION}

      - name: Install CDS
        run: npm install -g @sap/cds-dk

      - name: Install dependencies
        run: npm ci

      - name: Build MTA
        run: mbt build -t mta_archives

      - name: Upload MTAR artifact
        uses: actions/upload-artifact@v4
        with:
          name: mtar
          path: mta_archives/*.mtar

  test:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'sapmachine'

      - name: Run unit tests
        run: cd srv && mvn -B test

      - name: Run integration tests
        run: cd srv && mvn -B verify -Pintegration-tests

  deploy-dev:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    environment: development
    steps:
      - name: Download MTAR
        uses: actions/download-artifact@v4
        with:
          name: mtar

      - name: Install CF CLI
        run: |
          wget -q "https://packages.cloudfoundry.org/stable?release=linux64-binary&version=v8&source=github" -O cf-cli.tgz
          sudo tar -xzf cf-cli.tgz -C /usr/local/bin
          cf install-plugin multiapps -f

      - name: Deploy to Dev
        env:
          CF_USERNAME: ${{ secrets.CF_USERNAME }}
          CF_PASSWORD: ${{ secrets.CF_PASSWORD }}
        run: |
          cf login -a $CF_API -u "$CF_USERNAME" -p "$CF_PASSWORD" -o my-org -s dev
          cf deploy *.mtar -f

  deploy-prod:
    runs-on: ubuntu-latest
    needs: deploy-dev
    environment: production
    steps:
      - name: Download MTAR
        uses: actions/download-artifact@v4
        with:
          name: mtar

      - name: Install CF CLI
        run: |
          wget -q "https://packages.cloudfoundry.org/stable?release=linux64-binary&version=v8&source=github" -O cf-cli.tgz
          sudo tar -xzf cf-cli.tgz -C /usr/local/bin
          cf install-plugin multiapps -f

      - name: Blue-Green Deploy to Prod
        env:
          CF_USERNAME: ${{ secrets.CF_PROD_USERNAME }}
          CF_PASSWORD: ${{ secrets.CF_PROD_PASSWORD }}
        run: |
          cf login -a $CF_API -u "$CF_USERNAME" -p "$CF_PASSWORD" -o my-org -s prod
          cf bg-deploy *.mtar -f --no-confirm
```

---

## 3. Helm + Docker Pipeline for Kyma

### Multi-Stage Dockerfile for CAP Java

```dockerfile
# Stage 1: Build
FROM maven:3.9-sapmachine-17 AS builder
WORKDIR /app
COPY pom.xml .
COPY srv/pom.xml srv/
RUN mvn -B dependency:go-offline -pl srv -am
COPY . .
RUN mvn -B clean package -pl srv -am -DskipTests

# Stage 2: Runtime
FROM sapmachine:17-jre-headless
WORKDIR /app
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
COPY --from=builder /app/srv/target/*.jar app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:+UseG1GC", \
  "-XX:+ExitOnOutOfMemoryError", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

### Helm Chart Structure

```
chart/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-staging.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── hpa.yaml
    ├── apirule.yaml
    ├── service-instance.yaml
    ├── service-binding.yaml
    └── _helpers.tpl
```

### GitHub Actions — Kyma Pipeline

```yaml
# .github/workflows/kyma-deploy.yml
name: Build and Deploy to Kyma

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/my-srv

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  deploy-dev:
    runs-on: ubuntu-latest
    needs: build-and-push
    environment: development
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KYMA_KUBECONFIG_DEV }}" | base64 -d > $HOME/.kube/config

      - name: Deploy with Helm
        run: |
          helm upgrade --install my-srv ./chart \
            --namespace my-app \
            --create-namespace \
            --values ./chart/values-dev.yaml \
            --set image.tag=${{ github.sha }} \
            --wait --timeout 5m

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/my-srv -n my-app --timeout=3m
          kubectl get pods -n my-app -l app=my-srv

  deploy-prod:
    runs-on: ubuntu-latest
    needs: deploy-dev
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KYMA_KUBECONFIG_PROD }}" | base64 -d > $HOME/.kube/config

      - name: Canary deploy (10% traffic)
        run: |
          helm upgrade --install my-srv-canary ./chart \
            --namespace my-app \
            --values ./chart/values-prod.yaml \
            --set image.tag=${{ github.sha }} \
            --set replicaCount=1 \
            --wait --timeout 5m

      # Apply Istio traffic split
      - name: Apply traffic split
        run: |
          cat <<EOF | kubectl apply -f -
          apiVersion: networking.istio.io/v1beta1
          kind: VirtualService
          metadata:
            name: my-srv-canary
            namespace: my-app
          spec:
            hosts:
              - my-srv
            http:
              - route:
                  - destination:
                      host: my-srv
                      port:
                        number: 8080
                    weight: 90
                  - destination:
                      host: my-srv-canary
                      port:
                        number: 8080
                    weight: 10
          EOF

      - name: Monitor canary (5 min)
        run: |
          echo "Monitoring canary for 5 minutes..."
          sleep 300
          # Check error rate from metrics/logs
          ERROR_COUNT=$(kubectl logs -l app=my-srv-canary -n my-app --since=5m | grep -c "ERROR" || true)
          if [ "$ERROR_COUNT" -gt 10 ]; then
            echo "Canary failed with $ERROR_COUNT errors. Rolling back."
            helm rollback my-srv-canary 0 -n my-app
            exit 1
          fi

      - name: Promote canary to full
        run: |
          helm upgrade --install my-srv ./chart \
            --namespace my-app \
            --values ./chart/values-prod.yaml \
            --set image.tag=${{ github.sha }} \
            --wait --timeout 5m
          # Remove canary
          helm uninstall my-srv-canary -n my-app || true
          kubectl delete virtualservice my-srv-canary -n my-app || true
```

---

## 4. Quality Gates

### CAP-Specific Quality Checks

```yaml
# Quality gate job (add to any pipeline)
quality:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'

    - name: Install CDS tools
      run: npm install -g @sap/cds-dk

    - name: Install dependencies
      run: npm ci

    # 1. CDS Lint — validates CDS model conventions
    - name: CDS Lint
      run: npx cds lint

    # 2. Java static analysis
    - name: Maven static checks
      run: cd srv && mvn -B spotbugs:check pmd:check

    # 3. OWASP Dependency Check
    - name: OWASP Dependency Check
      run: |
        cd srv && mvn -B org.owasp:dependency-check-maven:check \
          -DfailBuildOnCVSS=7

    # 4. Unit tests with coverage
    - name: Unit tests + coverage
      run: |
        cd srv && mvn -B test jacoco:report
        # Extract coverage percentage
        COVERAGE=$(grep -oP 'Total.*?(\d+)%' srv/target/site/jacoco/index.html | grep -oP '\d+' | tail -1)
        echo "Code coverage: ${COVERAGE}%"
        if [ "${COVERAGE}" -lt 70 ]; then
          echo "Coverage ${COVERAGE}% is below threshold 70%"
          exit 1
        fi

    # 5. OData metadata validation
    - name: Validate OData metadata
      run: |
        npx cds compile srv --to edmx > /dev/null 2>&1
        echo "OData metadata generation successful"
```

### Test Pyramid for CAP Applications

```
                    ┌───────────┐
                    │  E2E /    │  Fiori launchpad tests
                    │  UI Tests │  (few, slow, fragile)
                  ┌─┴───────────┴─┐
                  │  Integration   │  OData API tests against
                  │  Tests         │  running CAP app + H2/PG
                ┌─┴───────────────┴─┐
                │  Component Tests   │  Handler tests with mocked
                │                    │  PersistenceService
              ┌─┴────────────────────┴─┐
              │    Unit Tests           │  Pure Java logic,
              │                         │  validators, mappers
              └─────────────────────────┘
```

### Integration Test Example

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class CatalogServiceIT {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldReturnBooks() throws Exception {
        mockMvc.perform(get("/odata/v4/CatalogService/Books")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.value").isArray())
            .andExpect(jsonPath("$.value[0].title").exists());
    }

    @Test
    void shouldRejectUnauthorizedAccess() throws Exception {
        mockMvc.perform(post("/odata/v4/CatalogService/Authors")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"name\": \"Test\"}"))
            .andExpect(status().isForbidden());
    }

    @Test
    void shouldValidateStockRange() throws Exception {
        mockMvc.perform(post("/odata/v4/CatalogService/Books")
                .with(SecurityMockMvcRequestPostProcessors
                    .jwt().authorities(new SimpleGrantedAuthority("Admin")))
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"title\":\"Test\",\"stock\":-1}"))
            .andExpect(status().isBadRequest());
    }
}
```

---

## 5. Deployment Strategies

### Blue-Green Deployment on Cloud Foundry

```
┌─────────────────────────────────────────────────────────────┐
│  cf bg-deploy my-app.mtar                                    │
│                                                              │
│  Step 1: Deploy "green" version alongside "blue" (live)      │
│  ┌──────────┐    ┌──────────┐                                │
│  │  Blue     │    │  Green   │                                │
│  │  (v1.0)   │    │  (v1.1)  │                                │
│  │  LIVE ◄───┤    │  IDLE    │                                │
│  └──────────┘    └──────────┘                                │
│                                                              │
│  Step 2: Route switch (zero-downtime)                        │
│  ┌──────────┐    ┌──────────┐                                │
│  │  Blue     │    │  Green   │                                │
│  │  (v1.0)   │    │  (v1.1)  │                                │
│  │  IDLE     │    │  LIVE ◄──┤                                │
│  └──────────┘    └──────────┘                                │
│                                                              │
│  Step 3: Cleanup old version                                 │
│  ┌──────────┐                                                │
│  │  Green   │                                                │
│  │  (v1.1)  │                                                │
│  │  LIVE ◄──┤                                                │
│  └──────────┘                                                │
└─────────────────────────────────────────────────────────────┘
```

Key commands:

```bash
# Standard blue-green deploy
cf bg-deploy my-app.mtar -f --no-confirm

# With manual validation step (deploy green, keep blue running)
cf bg-deploy my-app.mtar -f --skip-idle-start false

# Rollback (if green is bad)
cf bg-deploy my-app.mtar -i <process-id> -a abort

# Check deployment status
cf mta-ops
```

### Database Migration Safety

HDI container deployment is the most dangerous part of the pipeline. It runs as a separate module before the Java backend starts:

```
┌────────────────┐
│  mbt build     │
│  (generates    │
│   .mtar)       │
└───────┬────────┘
        │
        ▼
┌────────────────┐     ┌─────────────────────────────┐
│  cf deploy     │────→│  1. Deploy db-deployer       │
│  (processes    │     │     (runs HDI deploy)         │
│   modules in   │     │     - Creates/alters tables   │
│   order)       │     │     - Deploys calc views      │
│                │     │     - CANNOT rollback DDL      │
│                │     ├─────────────────────────────┤
│                │     │  2. Deploy srv (Java app)     │
│                │     │     (binds to same HDI)       │
│                │     ├─────────────────────────────┤
│                │     │  3. Deploy UI content         │
│                │     └─────────────────────────────┘
└────────────────┘
```

> **Critical rule:** HDI deployments are **not transactional**. If a table is altered (column dropped, type changed), there is no automatic rollback. Always:
> 1. Test schema migrations against a copy of production data
> 2. Use `hdi-deploy --dry-run` to preview changes
> 3. Keep backward-compatible schema changes (add columns, don't rename/drop)
> 4. For breaking changes, use a two-phase migration (deploy new column → migrate data → drop old column)

### Environment Promotion Strategy

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│   Dev    │────→│ Staging  │────→│   Prod   │
│          │     │          │     │          │
│ CF space │     │ CF space │     │ CF space │
│ or Kyma  │     │ or Kyma  │     │ or Kyma  │
│ namespace│     │ namespace│     │ namespace│
│          │     │          │     │          │
│ Auto-    │     │ Manual   │     │ Manual   │
│ deploy   │     │ approval │     │ approval │
│ on push  │     │ required │     │ required │
└──────────┘     └──────────┘     └──────────┘
     │                │                │
     ▼                ▼                ▼
  Same MTAR      Same MTAR       Same MTAR
  Different      Different       Different
  service keys   service keys    service keys
```

---

## Top 5 Pitfalls

1. **Running `mbt build` without `npm ci` first.** CDS compilation requires Node.js dependencies. The `@sap/cds-dk` and model dependencies must be installed before `mbt build`.
2. **Storing BTP credentials in pipeline YAML.** Always use secrets management (GitHub Secrets, Jenkins Credentials, SAP Credential Store). Never commit `cf login` passwords.
3. **Skipping HDI dry-run in production pipelines.** HDI schema changes are irreversible. A column drop in production cannot be rolled back by redeploying the old MTAR.
4. **Not pinning `mbt` and `cds-dk` versions.** Different versions of these tools can produce different build outputs. Pin versions in `package.json` and pipeline env vars.
5. **Using `cf push` instead of `cf deploy` for MTA apps.** `cf push` deploys a single app. `cf deploy` handles the full MTA lifecycle (services, bindings, deploy order, blue-green).

---

## What to Learn Next

- **Lesson 4.2:** Observability & Operations — logging, tracing, metrics on BTP
- **Lesson 4.3:** Spring Boot to CAP Migration Guide — mapping CI/CD patterns
- **Lesson 2.2:** Deploying Java Microservices on Kyma — Helm chart deep dive
- **Lesson 3.4:** CAP Advanced Patterns — deployment topologies and MTA structure