# Securing and Deploying to CloudHub

In this final lab you will secure the Products System API using Anypoint API Manager policies, encrypt sensitive credentials with Secure Properties, and deploy the application to CloudHub 2.0 — both manually through the Anypoint Platform UI and automatically via a Maven-based CI/CD pipeline.

**Prerequisites:** Lab 3-1 complete. Anypoint Platform account with API Manager access. Maven 3.8+.

---

## Step 1 — Register the API in API Manager

API Manager is the control plane where you apply policies, rate limits, and SLA tiers to APIs. Before you can apply policies, the Mule application must be linked to an API Manager instance.

In Anypoint Platform, navigate to **API Manager → + Add API → Add new API**.

```
Runtime:        Mule Gateway
Proxy type:     Deploy a proxy application (Mule 4)
API name:       products-system-api
API version:    v1
Asset type:     REST API (RAML)
```

After saving, note the **API Instance ID** (e.g. `12345678`). You will add it to `application.properties`:

```properties
anypoint.platform.client_id=${ANYPOINT_CLIENT_ID}
anypoint.platform.client_secret=${ANYPOINT_CLIENT_SECRET}
api.id=12345678
```

Add the Mule API Gateway autodiscovery config to `products-system-api-config.xml`:

```xml
<api-gateway:autodiscovery
    apiId="${api.id}"
    flowRef="products-system-api-main"
    doc:name="API Autodiscovery"/>
```

This causes the Mule runtime to register with API Manager on startup and receive applied policies automatically — no redeployment needed when you change policies.

---

## Step 2 — Apply Client ID Enforcement Policy

Client ID Enforcement requires all callers to present a registered `client_id` and `client_secret`. It is the baseline policy for any externally exposed API.

In API Manager → your API → **Policies → + Apply New Policy**:

```
Policy:          Client ID Enforcement
Credentials Origin: HTTP Basic Authentication header
                 (or: query params client_id / client_secret)
```

Register a test client application in **Exchange → Request Access**. Note the generated `client_id` and `client_secret`.

Test enforcement:

```bash
# No credentials — should return 401
curl -sv "http://localhost:8081/api/v1/products"

# With credentials — should return 200
curl -s -u "<client_id>:<client_secret>" "http://localhost:8081/api/v1/products" \
  | python3 -m json.tool
```

> **How it works:** The policy is injected as a filter before your flows execute. If credentials are missing or invalid, the request is rejected at the gateway layer — your flows never execute. This keeps business logic free of authentication concerns.

---

## Step 3 — Add Rate Limiting

Protect the API from abuse by applying a rate limit:

```
Policy:         Rate Limiting – SLA Based
Tier name:      Free
Rate limit:     10 requests per minute
```

Test the rate limit:

```bash
# Send 12 rapid requests — the 11th and 12th should return 429 Too Many Requests
for i in $(seq 1 12); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    -u "<client_id>:<client_secret>" \
    "http://localhost:8081/api/v1/products")
  echo "Request $i: $STATUS"
done
```

The `X-RateLimit-Remaining` response header tells callers how many requests they have left in the current window.

---

## Step 4 — Encrypt Credentials with Secure Properties

Storing plaintext passwords in `application.properties` is unsafe. Mule's Secure Properties module encrypts sensitive values so they are safe to commit.

Add the module to `pom.xml`:

```xml
<dependency>
    <groupId>com.mulesoft.modules</groupId>
    <artifactId>mule-secure-configuration-property-module</artifactId>
    <version>1.2.5</version>
    <classifier>mule-plugin</classifier>
</dependency>
```

Generate an encrypted value using the Mule Credentials Vault CLI tool:

```bash
# Install the tool
mvn dependency:get -Dartifact=com.mulesoft.tools:secure-properties-tool:jar:1.0.0

# Encrypt the DB password
java -jar secure-properties-tool.jar \
  string encrypt AES CBC my-secret-key "mulepass"
```

The output is an encrypted string like `![HEx9WUCMriZkNHtxHRTfuQ==]`. Add it to `application.properties`:

```properties
db.user=mule
db.password=![HEx9WUCMriZkNHtxHRTfuQ==]
```

Replace the standard `<configuration-properties>` tag with the secure version in `products-system-api-config.xml`:

```xml
<secure-properties:config
    name="Secure_Properties"
    file="application.properties"
    key="${enc.key}"
    doc:name="Secure Properties Config">
    <secure-properties:encrypt algorithm="AES" mode="CBC"/>
</secure-properties:config>
```

Pass the encryption key at startup (never commit it):

```bash
# Local run — pass key as JVM arg
mvn spring-boot:run -Denc.key=my-secret-key

# Or as an environment variable via .mvn/jvm.config:
# -Denc.key=${ENC_KEY}
```

---

## Step 5 — Package and Deploy to CloudHub 2.0 via the UI

Package the application as a deployable JAR:

```bash
cd products-system-api
mvn clean package -DskipTests
ls -lh target/*.jar
```

In Anypoint Platform, navigate to **Runtime Manager → Deploy Application**:

```
Application name:   products-system-api
Deployment target:  CloudHub 2.0
Runtime version:    4.x.x
Worker size:        0.1 vCores (sandbox)
Workers:            1
```

Upload the JAR from `target/products-system-api-1.0.0-mule-application.jar`.

Set the following **Application Properties** in the UI (instead of bundling secrets in the JAR):

```
db.user          = mule
db.password      = ![HEx9WUCMriZkNHtxHRTfuQ==]
enc.key          = my-secret-key
anypoint.platform.client_id     = <your-client-id>
anypoint.platform.client_secret = <your-client-secret>
api.id           = 12345678
```

Click **Deploy** and watch the deployment log. A green **Started** status means the app is running.

---

## Step 6 — Automate Deployment with the Mule Maven Plugin

For CI/CD pipelines, use the Mule Maven Plugin to deploy directly from `mvn deploy`:

Add the plugin to `pom.xml`:

```xml
<plugin>
    <groupId>org.mule.tools.maven</groupId>
    <artifactId>mule-maven-plugin</artifactId>
    <version>3.8.7</version>
    <extensions>true</extensions>
    <configuration>
        <cloudHubDeployment>
            <uri>https://anypoint.mulesoft.com</uri>
            <muleVersion>4.6.2</muleVersion>
            <username>${anypoint.username}</username>
            <password>${anypoint.password}</password>
            <applicationName>products-system-api</applicationName>
            <environment>Sandbox</environment>
            <workerType>MICRO</workerType>
            <workers>1</workers>
            <region>us-east-1</region>
            <properties>
                <db.user>mule</db.user>
                <enc.key>${enc.key}</enc.key>
                <api.id>12345678</api.id>
            </properties>
        </cloudHubDeployment>
    </configuration>
</plugin>
```

Create a GitHub Actions workflow at `.github/workflows/deploy.yml`:

```yaml
name: Deploy to CloudHub

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '11'
          distribution: temurin
          cache: maven

      - name: Build and test
        run: mvn clean verify -B

      - name: Deploy to CloudHub
        run: |
          mvn deploy -DskipTests \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }} \
            -Denc.key=${{ secrets.ENC_KEY }}
```

Validate the workflow YAML is valid:

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/deploy.yml'))" && echo "YAML valid"
```

---

## Step 7 — Course Recap: The Full API Lifecycle

Congratulations — you have built, secured, and deployed a production-quality MuleSoft API. Here is the complete journey:

| Lab | What you built |
|---|---|
| **1-1** | HTTP Listener flow, Mule Event model, sub-flows, DataWeave inline expressions |
| **2-1** | RAML 1.0 API spec, Design Center mocking service, APIkit scaffolding |
| **2-2** | Database connector, parameterised SQL, DataWeave map/filter/groupBy, HTTP Requester |
| **3-1** | Typed error handlers, global default handler, correlation IDs, Until Successful retries, MUnit tests |
| **3-2** | API Manager policies, Client ID Enforcement, rate limiting, Secure Properties encryption, CloudHub deployment, CI/CD pipeline |

**Recommended next steps:**

- Explore the **Salesforce** and **Workday** connectors in Anypoint Exchange — MuleSoft's library of 1,000+ pre-built connectors is its biggest productivity advantage
- Learn **Batch Processing** for high-volume file and database integrations — Mule's Batch Job scope handles millions of records with built-in checkpointing and error record handling
- Investigate **AsyncAPI** specification support for event-driven integration patterns with Apache Kafka and Anypoint MQ
- Study the **MuleSoft Certified Developer** exam objectives to formalise your knowledge
