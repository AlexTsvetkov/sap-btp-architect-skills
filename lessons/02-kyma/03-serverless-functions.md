# Lesson 2.3 — Kyma Serverless Functions

> **Summary:** Kyma Serverless Functions provide a lightweight, event-driven compute model for simple tasks — webhook handlers, data transformations, event processors — without the overhead of building Docker images and Helm charts. This lesson covers when to use functions vs microservices, the Function CRD, runtime environments, BTP service binding, and practical patterns.

---

## 1. Functions vs Microservices: Decision Framework

| Criteria | Kyma Function | Java Microservice |
|---|---|---|
| Complexity | Simple logic (< 500 LOC) | Complex business logic |
| Language | Node.js, Python | Java, Node.js, Go, any |
| Startup time | ~1-3s (cold start) | ~15-60s (Spring Boot) |
| Dependencies | npm/pip packages only | Full Maven/Gradle ecosystem |
| Container control | None (managed) | Full Dockerfile control |
| Scaling | Auto (0 to N, scale-to-zero) | HPA (min 1 replica) |
| State | Stateless only | Stateless (can use external state) |
| Testing | Limited local testing | Full unit/integration testing |
| Debugging | Console logs only | Full IDE debugging |
| Best for | Event handlers, webhooks, glue code | Business services, APIs, CAP apps |

**Rule of thumb:** If you'd write it as an AWS Lambda, write it as a Kyma Function. If you'd write it as an ECS service, write it as a Kyma microservice.

---

## 2. Function CRD

### Basic Node.js Function

```yaml
apiVersion: serverless.kyma-project.io/v1alpha2
kind: Function
metadata:
  name: order-validator
  namespace: my-app
spec:
  runtime: nodejs20
  source:
    inline:
      source: |
        module.exports = {
          main: async function (event, context) {
            const order = JSON.parse(event.data);
            
            // Validate order
            if (!order.customerId || !order.items?.length) {
              return { statusCode: 400, body: { error: "Invalid order" } };
            }
            
            // Check total
            const total = order.items.reduce((sum, item) => sum + item.price * item.qty, 0);
            
            return {
              statusCode: 200,
              body: { 
                valid: true, 
                orderId: order.id, 
                total: total 
              }
            };
          }
        };
      dependencies: |
        {
          "name": "order-validator",
          "version": "1.0.0",
          "dependencies": {}
        }
  scaleConfig:
    minReplicas: 0
    maxReplicas: 5
  resourceConfiguration:
    function:
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
```

### Python Function

```yaml
apiVersion: serverless.kyma-project.io/v1alpha2
kind: Function
metadata:
  name: data-transformer
  namespace: my-app
spec:
  runtime: python312
  source:
    inline:
      source: |
        import json
        
        def main(event, context):
            data = json.loads(event["data"])
            
            # Transform S/4HANA BP format to internal format
            transformed = {
                "id": data.get("BusinessPartner"),
                "name": data.get("BusinessPartnerFullName"),
                "email": data.get("EmailAddress", "").lower(),
                "region": map_country_to_region(data.get("Country", ""))
            }
            
            return {"statusCode": 200, "body": json.dumps(transformed)}
        
        def map_country_to_region(country):
            regions = {"DE": "EMEA", "US": "NA", "JP": "APJ"}
            return regions.get(country, "UNKNOWN")
      dependencies: |
        requests==2.31.0
```

### Git Source (Production Pattern)

For production, store function source in Git instead of inline:

```yaml
apiVersion: serverless.kyma-project.io/v1alpha2
kind: Function
metadata:
  name: order-validator
spec:
  runtime: nodejs20
  source:
    gitRepository:
      url: "https://github.com/my-org/my-functions.git"
      baseDir: "/functions/order-validator"
      reference: "main"
      auth:
        type: key
        secretName: git-creds
```

---

## 3. Binding BTP Services to Functions

### Using Environment Variables from Secrets

```yaml
apiVersion: serverless.kyma-project.io/v1alpha2
kind: Function
metadata:
  name: bp-event-handler
spec:
  runtime: nodejs20
  source:
    inline:
      source: |
        const axios = require('axios');
        
        module.exports = {
          main: async function (event, context) {
            // Credentials from BTP service binding (mounted as env vars)
            const xsuaaUrl = process.env.XSUAA_URL;
            const clientId = process.env.XSUAA_CLIENTID;
            const clientSecret = process.env.XSUAA_CLIENTSECRET;
            
            // Get OAuth token
            const tokenResponse = await axios.post(
              `${xsuaaUrl}/oauth/token`,
              'grant_type=client_credentials',
              { auth: { username: clientId, password: clientSecret } }
            );
            
            // Call S/4HANA via destination
            const bpId = JSON.parse(event.data).BusinessPartner;
            // ... fetch full BP data
            
            return { statusCode: 200, body: { processed: bpId } };
          }
        };
      dependencies: |
        { "dependencies": { "axios": "^1.6.0" } }
  env:
    - name: XSUAA_URL
      valueFrom:
        secretKeyRef:
          name: my-xsuaa-secret
          key: url
    - name: XSUAA_CLIENTID
      valueFrom:
        secretKeyRef:
          name: my-xsuaa-secret
          key: clientid
    - name: XSUAA_CLIENTSECRET
      valueFrom:
        secretKeyRef:
          name: my-xsuaa-secret
          key: clientsecret
```

---

## 4. Event-Triggered Functions

### Connecting a Function to Kyma Eventing

```yaml
# Step 1: Deploy the function
apiVersion: serverless.kyma-project.io/v1alpha2
kind: Function
metadata:
  name: bp-event-processor
  namespace: my-app
spec:
  runtime: nodejs20
  source:
    inline:
      source: |
        module.exports = {
          main: async function (event, context) {
            console.log("Received CloudEvent:", JSON.stringify(event.extensions));
            console.log("Event data:", event.data);
            
            const bpId = JSON.parse(event.data).BusinessPartner;
            console.log(`Processing BP: ${bpId}`);
            
            // Your processing logic here
            
            return { statusCode: 200, body: { processed: true } };
          }
        };
---
# Step 2: Create subscription pointing to the function's service
apiVersion: eventing.kyma-project.io/v1alpha2
kind: Subscription
metadata:
  name: bp-changed-sub
  namespace: my-app
spec:
  source: ""
  types:
    - sap.s4.beh.businesspartner.changed.v1
  sink: http://bp-event-processor.my-app.svc.cluster.local
  config:
    maxInFlightMessages: "10"
```

### HTTP-Triggered Functions (API Gateway)

```yaml
# Expose function via APIRule
apiVersion: gateway.kyma-project.io/v1beta1
kind: APIRule
metadata:
  name: order-validator-api
  namespace: my-app
spec:
  host: order-validator.kyma.ondemand.com
  service:
    name: order-validator  # auto-created Service for the Function
    port: 80
  gateway: kyma-system/kyma-gateway
  rules:
    - path: /.*
      methods: ["POST"]
      accessStrategies:
        - handler: jwt
          config:
            jwks_urls:
              - "https://mysubaccount.authentication.eu10.hana.ondemand.com/token_keys"
```

---

## 5. Practical Patterns

### Pattern: Event Router

Route different event types to different processing logic:

```javascript
module.exports = {
  main: async function (event, context) {
    const eventType = event.extensions?.type || event.type;
    
    switch (eventType) {
      case 'sap.s4.beh.businesspartner.created.v1':
        return handleBPCreated(event);
      case 'sap.s4.beh.businesspartner.changed.v1':
        return handleBPChanged(event);
      default:
        console.warn(`Unknown event type: ${eventType}`);
        return { statusCode: 200, body: { skipped: true } };
    }
  }
};
```

### Pattern: Webhook Adapter

Convert external webhook formats to CloudEvents for internal consumption:

```javascript
const axios = require('axios');

module.exports = {
  main: async function (event, context) {
    // Receive Stripe webhook
    const stripeEvent = JSON.parse(event.data);
    
    // Transform to CloudEvent and publish to internal topic
    const cloudEvent = {
      specversion: "1.0",
      type: `my.company.payment.${stripeEvent.type}.v1`,
      source: "/stripe/webhook",
      id: stripeEvent.id,
      data: {
        amount: stripeEvent.data.object.amount,
        currency: stripeEvent.data.object.currency,
        customerId: stripeEvent.data.object.customer
      }
    };
    
    // Forward to NATS/Event Mesh
    await axios.post('http://eventing-publisher.eventing-system.svc.cluster.local/publish', cloudEvent);
    
    return { statusCode: 200 };
  }
};
```

---

## Top 5 Pitfalls

1. **Using functions for complex business logic.** If your function exceeds ~300 LOC or needs multiple modules, switch to a microservice.
2. **Ignoring cold start latency.** Scale-to-zero means the first request after idle will be slow (~2-5s). Set `minReplicas: 1` for latency-sensitive endpoints.
3. **Not testing locally.** Kyma functions are hard to test locally. Extract core logic into testable modules and import them.
4. **Storing state in function memory.** Functions can scale to zero and back. Use external storage (Redis, HANA) for any state.
5. **Inline source in production.** Use Git source for version control, code review, and CI/CD integration.

---

## What to Learn Next

- **Lesson 2.1:** Kyma Architecture — understanding the serverless module
- **Lesson 2.4:** Kyma Eventing — event-driven extension patterns
- **Lesson 1.3:** Event Mesh — the messaging backbone for functions