# Connectors and DataWeave Transformations

In this lab you will replace the hardcoded mock payload with a real Database connector backed by PostgreSQL, then use DataWeave 2.0 to transform query results into the API's response format. You will also connect to an external HTTP service and learn DataWeave's full transformation toolkit — mapping, filtering, grouping, and format conversion.

**Prerequisites:** Lab 2-1 complete. Docker installed (to run PostgreSQL locally). The `products-system-api` project open in Anypoint Studio.

---

## Step 1 — Start a Local PostgreSQL Database

Run a PostgreSQL container with seed data for the products catalogue:

```bash
docker run -d \
  --name mule-postgres \
  -e POSTGRES_DB=catalogue \
  -e POSTGRES_USER=mule \
  -e POSTGRES_PASSWORD=mulepass \
  -p 5432:5432 \
  postgres:16-alpine
```

Create the schema and seed data:

```bash
docker exec -i mule-postgres psql -U mule -d catalogue << 'SQL'
CREATE TABLE products (
    id          SERIAL PRIMARY KEY,
    sku         VARCHAR(50)    NOT NULL UNIQUE,
    name        VARCHAR(200)   NOT NULL,
    price       NUMERIC(10,2)  NOT NULL,
    stock_level INTEGER        NOT NULL DEFAULT 0,
    category    VARCHAR(50)    NOT NULL
);

INSERT INTO products (sku, name, price, stock_level, category) VALUES
  ('ELEC-001', 'Wireless Headphones',  79.99, 142, 'electronics'),
  ('ELEC-002', 'USB-C Hub',            34.95,  67, 'electronics'),
  ('BOOK-001', 'Clean Code',           35.00, 200, 'books'),
  ('BOOK-002', 'Designing Data-Intensive Applications', 45.00, 88, 'books'),
  ('CLOTH-001','Running Jacket',       89.95,  34, 'clothing');
SQL
```

Verify the data loaded:

```bash
docker exec -it mule-postgres psql -U mule -d catalogue -c "SELECT * FROM products;"
```

---

## Step 2 — Add the Database Connector Dependency

In `pom.xml`, add the Database connector and PostgreSQL driver:

```xml
<dependency>
    <groupId>org.mule.connectors</groupId>
    <artifactId>mule-db-connector</artifactId>
    <version>1.14.0</version>
    <classifier>mule-plugin</classifier>
</dependency>

<!-- PostgreSQL JDBC driver -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.7.3</version>
</dependency>
```

Add the global database configuration to `products-system-api-config.xml`:

```xml
<db:config name="Database_Config" doc:name="Database Config">
    <db:generic-connection
        url="jdbc:postgresql://localhost:5432/catalogue"
        driverClassName="org.postgresql.Driver"
        user="${db.user}"
        password="${db.password}"/>
</db:config>
```

Create `src/main/resources/application.properties`:

```properties
db.user=mule
db.password=mulepass
http.port=8081
```

> **Never hardcode credentials** in XML config files — always externalise them via properties. In production, CloudHub Secure Properties or a secrets manager replaces these values.

---

## Step 3 — Query the Database in the GET /products Flow

Replace the hardcoded `set-payload` in the `GET /products` flow with a Database Select operation:

```xml
<flow name="get:\products:products-system-api-config">

    <db:select config-ref="Database_Config" doc:name="Select products">
        <db:sql><![CDATA[
            SELECT id, sku, name, price, stock_level, category
            FROM   products
            WHERE  (:category IS NULL OR category = :category)
            ORDER  BY id
            LIMIT  :limit
        ]]></db:sql>
        <db:input-parameters><![CDATA[#[{
            category : attributes.queryParams.category default null,
            limit    : (attributes.queryParams.limit default "20") as Integer
        }]]]></db:input-parameters>
    </db:select>

    <ee:transform doc:name="Map to API response">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
payload map (row) -> {
    id:         row.id,
    sku:        row.sku,
    name:       row.name,
    price:      row.price,
    stockLevel: row.stock_level,
    category:   row.category
}]]>
            </ee:set-payload>
        </ee:message>
    </ee:transform>

</flow>
```

Test the live database endpoint:

```bash
curl -s "http://localhost:8081/api/v1/products" | python3 -m json.tool
curl -s "http://localhost:8081/api/v1/products?category=books" | python3 -m json.tool
```

---

## Step 4 — DataWeave 2.0 Core Concepts

DataWeave is a functional, expression-oriented transformation language. Every DataWeave script has two sections:

```
%dw 2.0
output application/json     ← output MIME type

---                          ← separator

/* transformation body */
```

The most important operators:

| Operator | Purpose | Example |
|---|---|---|
| `map` | Transform each item in an array | `payload map ($.name)` |
| `filter` | Keep items matching a condition | `payload filter ($.price > 50)` |
| `groupBy` | Group array into map by key | `payload groupBy ($.category)` |
| `reduce` | Fold array to a single value | `payload reduce ((acc, item) -> acc + item.price)` |
| `pluck` | Iterate over object key-value pairs | `obj pluck (v, k) -> { (k): upper(v) }` |
| `mapObject` | Transform each key-value in an object | `obj mapObject { ($$): upper($) }` |
| `default` | Fallback when value is null | `payload.name default "Unknown"` |
| `as` | Type coercion | `"42" as Integer` |

Test these in the **DataWeave Playground** (built into Anypoint Studio — Window → Show View → DataWeave):

```
%dw 2.0
output application/json
var products = [
    { sku: "ELEC-001", price: 79.99, category: "electronics" },
    { sku: "BOOK-001", price: 35.00, category: "books"       },
    { sku: "ELEC-002", price: 34.95, category: "electronics" }
]
---
{
    total:        sizeOf(products),
    avgPrice:     (products reduce ((acc, p) -> acc + p.price)) / sizeOf(products),
    byCategory:   products groupBy ($.category),
    expensive:    products filter ($.price > 50) map ($.sku)
}
```

---

## Step 5 — Transform Between Formats: JSON to XML

DataWeave can emit any format by changing the `output` directive. This is MuleSoft's answer to XSLT — far more readable and testable.

Add a new flow that exposes the products as XML for a legacy system integration:

```xml
<flow name="get-products-xml-flow">
    <http:listener config-ref="HTTP_Listener_config" path="/legacy/products"/>

    <db:select config-ref="Database_Config" doc:name="Select products">
        <db:sql>SELECT id, sku, name, price, stock_level, category FROM products</db:sql>
    </db:select>

    <ee:transform doc:name="Map to XML">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/xml
---
{
    catalogue: {
        (payload map (row) -> {
            product @(id: row.id, sku: row.sku): {
                name:       row.name,
                price:      row.price,
                stockLevel: row.stock_level,
                category:   row.category
            }
        })
    }
}]]>
            </ee:set-payload>
        </ee:message>
    </ee:transform>
</flow>
```

```bash
curl -s "http://localhost:8081/legacy/products"
```

Expected output:

```xml
<?xml version='1.0' encoding='UTF-8'?>
<catalogue>
  <product id="1" sku="ELEC-001">
    <name>Wireless Headphones</name>
    <price>79.99</price>
    <stockLevel>142</stockLevel>
    <category>electronics</category>
  </product>
  ...
</catalogue>
```

---

## Step 6 — Call an External HTTP Service and Merge Data

MuleSoft's HTTP Requester connector makes outbound API calls. Chain it with DataWeave to merge data from two sources.

Add the HTTP Requester to `pom.xml`:

```xml
<dependency>
    <groupId>org.mule.connectors</groupId>
    <artifactId>mule-http-connector</artifactId>
    <version>1.9.1</version>
    <classifier>mule-plugin</classifier>
</dependency>
```

Build a flow that fetches exchange rates and enriches product prices:

```xml
<http:request-config name="ExchangeRate_Config">
    <http:request-connection host="open.er-api.com" port="443" protocol="HTTPS"/>
</http:request-config>

<flow name="get-products-with-eur-flow">
    <http:listener config-ref="HTTP_Listener_config" path="/products/eur"/>

    <!-- Fetch products from DB -->
    <db:select config-ref="Database_Config" doc:name="Select products">
        <db:sql>SELECT id, sku, name, price FROM products</db:sql>
    </db:select>
    <set-variable variableName="products" value="#[payload]"/>

    <!-- Fetch live exchange rate -->
    <http:request method="GET" config-ref="ExchangeRate_Config"
                  path="/v6/latest/USD" doc:name="Get exchange rate"/>
    <set-variable variableName="eurRate"
                  value='#[payload.rates.EUR]' doc:name="Store EUR rate"/>

    <!-- Merge: add EUR price to each product -->
    <ee:transform doc:name="Add EUR prices">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
---
vars.products map (p) -> {
    id:       p.id,
    sku:      p.sku,
    name:     p.name,
    priceUSD: p.price,
    priceEUR: (p.price * vars.eurRate) as Number { format: "0.00" }
}]]>
            </ee:set-payload>
        </ee:message>
    </ee:transform>
</flow>
```

```bash
curl -s "http://localhost:8081/products/eur" | python3 -m json.tool
```

---

## Step 7 — Checkpoint and What's Next

You have connected real systems using MuleSoft connectors and mastered the DataWeave transformation language:

- Configured the Database connector with externalised credentials
- Queried PostgreSQL with parameterised SQL and mapped rows to JSON with DataWeave
- Used `map`, `filter`, `groupBy`, and `reduce` in the DataWeave Playground
- Output XML for a legacy system integration using the same DataWeave `output` directive
- Called an external HTTP API with the HTTP Requester and merged results using `vars`

In Module 3 you will make the API production-ready — handling errors gracefully with typed error handlers, adding retry strategies for flaky external calls, enforcing API policies in API Manager, and deploying to CloudHub.
