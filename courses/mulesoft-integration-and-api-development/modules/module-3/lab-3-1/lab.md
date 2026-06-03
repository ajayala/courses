# Error Handling and Resilience Patterns

In this lab you will add production-grade error handling to the Products System API. You will learn the difference between Mule 4's two error scope types, build a global default error handler, add structured logging with correlation IDs, and implement an Until Successful retry strategy for unreliable external HTTP calls.

**Prerequisites:** Lab 2-2 complete. The `products-system-api` project running locally.

---

## Step 1 — Understand Mule 4 Error Types

Every error in Mule 4 has a **namespace:type** identifier. This is what makes error handling composable — you can catch a broad namespace or a specific type.

| Namespace | Covers |
|---|---|
| `MULE` | Core Mule runtime errors |
| `HTTP` | HTTP connector errors (4xx, 5xx) |
| `DB` | Database connector errors |
| `APIKIT` | APIkit routing errors (400, 404, 405, 406, 415) |
| `APP` | Custom errors you raise with `raise-error` |
| `CONNECTIVITY` | Connection failures across all connectors |

Common specific types:

| Error type | When it occurs |
|---|---|
| `HTTP:UNAUTHORIZED` | 401 from an outbound HTTP call |
| `HTTP:NOT_FOUND` | 404 from an outbound HTTP call |
| `DB:CONNECTIVITY` | Cannot connect to database |
| `DB:QUERY_EXECUTION` | SQL error (bad syntax, constraint violation) |
| `MULE:EXPRESSION` | DataWeave expression evaluation failed |
| `APIKIT:NOT_FOUND` | Request path not matched by any flow |
| `APIKIT:BAD_REQUEST` | Request body fails RAML schema validation |

```xml
<!-- Raise a custom application error -->
<raise-error type="APP:PRODUCT_NOT_FOUND"
             description='#["Product " ++ vars.productId ++ " does not exist"]'/>
```

---

## Step 2 — Add On Error Continue and On Error Propagate

Mule 4 provides two error handler behaviours inside an `<error-handler>` block:

| Scope | Behaviour |
|---|---|
| `on-error-continue` | Handles the error and **continues** — the flow completes normally, returning whatever payload the error handler sets |
| `on-error-propagate` | Handles the error and **re-throws** it — the error propagates to the caller or a parent error handler |

Add a local error handler to the `GET /products/{productId}` flow:

```xml
<flow name="get:\products\{productId}:products-system-api-config">

    <db:select config-ref="Database_Config" doc:name="Select product by ID">
        <db:sql>SELECT id, sku, name, price, stock_level, category
                FROM products WHERE id = :id</db:sql>
        <db:input-parameters>#[{ id: attributes.uriParams.productId as Integer }]</db:input-parameters>
    </db:select>

    <validation:is-not-empty
        value="#[payload]"
        message="Product not found"
        doc:name="Assert product exists"/>

    <ee:transform doc:name="Map to response">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
var row = payload[0]
---
{ id: row.id, sku: row.sku, name: row.name,
  price: row.price, stockLevel: row.stock_level, category: row.category }]]>
            </ee:set-payload>
        </ee:message>
    </ee:transform>

    <error-handler>
        <on-error-continue
            enableNotifications="false"
            type="MULE:VALIDATION"
            doc:name="Handle not found">
            <set-payload
                value='#[{ "code": "PRODUCT_NOT_FOUND",
                           "message": "Product " ++ attributes.uriParams.productId ++ " not found" }]'
                mimeType="application/json"/>
            <http:set-response-attribute
                doc:name="Set 404 status" statusCode="404"/>
        </on-error-continue>

        <on-error-propagate type="DB:CONNECTIVITY" doc:name="Propagate DB errors"/>
    </error-handler>

</flow>
```

Test both paths:

```bash
# Existing product
curl -s "http://localhost:8081/api/v1/products/1" | python3 -m json.tool

# Non-existent product — should return 404 with error body
curl -sv "http://localhost:8081/api/v1/products/999"
```

---

## Step 3 — Build a Global Default Error Handler

A global error handler catches any error not handled by a local handler. Define it in `products-system-api-config.xml` and register it as the application default:

```xml
<error-handler name="Global_Error_Handler">

    <!-- APIKIT validation errors → 400 Bad Request -->
    <on-error-continue type="APIKIT:BAD_REQUEST" enableNotifications="false">
        <ee:transform>
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{ code: "BAD_REQUEST", message: error.description }]]>
                </ee:set-payload>
            </ee:message>
        </ee:transform>
        <http:set-response-attribute statusCode="400" doc:name="Set 400"/>
    </on-error-continue>

    <!-- APIKIT routing errors → 404/405/406 -->
    <on-error-continue type="APIKIT:NOT_FOUND,APIKIT:METHOD_NOT_ALLOWED,APIKIT:NOT_ACCEPTABLE">
        <ee:transform>
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{ code: "ROUTING_ERROR", message: error.description }]]>
                </ee:set-payload>
            </ee:message>
        </ee:transform>
        <http:set-response-attribute statusCode="#[error.errorType.identifier == 'NOT_FOUND' ? 404 : 405]"/>
    </on-error-continue>

    <!-- Database connectivity → 503 Service Unavailable -->
    <on-error-continue type="DB:CONNECTIVITY,CONNECTIVITY">
        <logger level="ERROR" message='#["DB connectivity error: " ++ error.description]'/>
        <ee:transform>
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{ code: "SERVICE_UNAVAILABLE", message: "Database is temporarily unavailable" }]]>
                </ee:set-payload>
            </ee:message>
        </ee:transform>
        <http:set-response-attribute statusCode="503"/>
    </on-error-continue>

    <!-- Catch-all → 500 Internal Server Error -->
    <on-error-continue>
        <logger level="ERROR" message='#["Unhandled error [" ++ error.errorType ++ "]: " ++ error.description]'/>
        <ee:transform>
            <ee:message>
                <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
{ code: "INTERNAL_ERROR", message: "An unexpected error occurred" }]]>
                </ee:set-payload>
            </ee:message>
        </ee:transform>
        <http:set-response-attribute statusCode="500"/>
    </on-error-continue>

</error-handler>

<!-- Register as the application default -->
<configuration defaultErrorHandler-ref="Global_Error_Handler"/>
```

> **Never expose internal error details to API consumers.** Log the full stack trace internally, but return a sanitised message. The catch-all above deliberately omits the raw error message from the response body.

---

## Step 4 — Add Correlation IDs for Distributed Tracing

In a microservices architecture, a single user request may pass through many Mule applications. Correlation IDs allow you to trace the full journey across logs.

Mule automatically generates a `correlationId` for each event. Expose it in response headers and log it consistently:

```xml
<flow name="get:\products:products-system-api-config">

    <logger
        level="INFO"
        message='#["[" ++ correlationId ++ "] GET /products - category=" ++ (attributes.queryParams.category default "all")]'
        doc:name="Request log"/>

    <db:select config-ref="Database_Config">
        <db:sql>SELECT * FROM products
                WHERE (:category IS NULL OR category = :category)</db:sql>
        <db:input-parameters>#[{ category: attributes.queryParams.category default null }]</db:input-parameters>
    </db:select>

    <ee:transform doc:name="Map response">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
payload map (r) -> { id: r.id, sku: r.sku, name: r.name,
                     price: r.price, stockLevel: r.stock_level, category: r.category }]]>
            </ee:set-payload>
        </ee:message>
        <ee:attributes><![CDATA[%dw 2.0
output application/java
---
{ "X-Correlation-Id": correlationId }]]>
        </ee:attributes>
    </ee:transform>

    <logger
        level="INFO"
        message='#["[" ++ correlationId ++ "] Returning " ++ sizeOf(payload) ++ " products"]'
        doc:name="Response log"/>

</flow>
```

Test that the correlation ID appears in the response header:

```bash
curl -sv "http://localhost:8081/api/v1/products" 2>&1 | grep -i "x-correlation"
```

---

## Step 5 — Retry Flaky External Calls with Until Successful

The `until-successful` scope retries a block of processors until it succeeds or a maximum attempt count is reached. This is the right tool for transient failures in external HTTP calls.

Wrap the exchange rate HTTP call from Lab 2-2 in a retry scope:

```xml
<until-successful
    maxRetries="3"
    millisBetweenRetries="2000"
    doc:name="Retry exchange rate call">

    <http:request
        method="GET"
        config-ref="ExchangeRate_Config"
        path="/v6/latest/USD"
        doc:name="Get exchange rate"/>

</until-successful>
```

Add an error handler that catches `MULE:RETRY_EXHAUSTED` (thrown when all retries fail):

```xml
<error-handler>
    <on-error-continue type="MULE:RETRY_EXHAUSTED">
        <logger level="WARN"
                message='#["Exchange rate service unreachable after retries — using fallback rate"]'/>
        <set-variable variableName="eurRate" value="#[0.92]"
                      doc:name="Fallback EUR rate"/>
    </on-error-continue>
</error-handler>
```

| Attribute | Meaning |
|---|---|
| `maxRetries` | Number of retry attempts after the first failure |
| `millisBetweenRetries` | Fixed delay between attempts (ms) |
| `failDeployment` | If `true`, deployment fails if the first call fails (useful for startup checks) |

> **For exponential backoff** — which is preferable for production HTTP retries — use the HTTP Requester's built-in `reconnection` strategy or integrate with a Mule Reconnection Policy rather than `until-successful`.

---

## Step 6 — Write an MUnit Test for the Error Handler

MUnit is Mule's built-in testing framework. Write a test that verifies the 404 response for a missing product:

```xml
<!-- src/test/munit/products-test-suite.xml -->
<munit:test name="get-product-not-found-returns-404"
            description="Verify 404 is returned for an unknown product ID">

    <munit:behavior>
        <!-- Mock the DB to return an empty list -->
        <munit-tools:mock-when processor="db:select">
            <munit-tools:with-attributes>
                <munit-tools:with-attribute attributeName="doc:name" whereValue="Select product by ID"/>
            </munit-tools:with-attributes>
            <munit-tools:then-return>
                <munit-tools:payload value="#[[]]" mediaType="application/java"/>
            </munit-tools:then-return>
        </munit-tools:mock-when>
    </munit:behavior>

    <munit:execution>
        <http:request method="GET" config-ref="HTTP_Request_test_config"
                      path="/api/v1/products/999"/>
    </munit:execution>

    <munit:validation>
        <munit-tools:assert-that
            expression="#[attributes.statusCode]"
            is="#[MunitTools::equalTo(404)]"/>
        <munit-tools:assert-that
            expression="#[payload.code]"
            is="#[MunitTools::equalTo('PRODUCT_NOT_FOUND')]"/>
    </munit:validation>

</munit:test>
```

Run the test suite:

```bash
mvn test -pl products-system-api
```

---

## Step 7 — Checkpoint and What's Next

The Products System API is now resilient and observable:

- Typed error handlers (`on-error-continue` / `on-error-propagate`) return correct HTTP status codes
- A global default error handler covers all unhandled error types
- Correlation IDs flow through logs and response headers for distributed tracing
- The exchange rate call retries automatically with a fallback if all retries are exhausted
- An MUnit test verifies the 404 error path using a mocked database

In the final lab you will make the API secure and deploy it — applying Client ID Enforcement and JWT validation policies in API Manager, encrypting credentials with Secure Properties, and deploying to CloudHub 2.0 via both the UI and a CI/CD Maven pipeline.
