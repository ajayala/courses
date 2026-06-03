# Designing APIs with RAML and APIkit

In this lab you will design a RESTful API specification using RAML 1.0 in Anypoint Design Center, publish it to Anypoint Exchange, and then use the APIkit router in Anypoint Studio to automatically scaffold the implementation flows. This API-first approach separates interface from implementation and lets teams agree on a contract before writing a single line of integration logic.

**Prerequisites:** Lab 1-1 complete. Anypoint Platform account with access to Design Center and Exchange.

---

## Step 1 — Understand API-Led Connectivity

MuleSoft promotes an architectural pattern called **API-Led Connectivity**, which organises integrations into three layers:

| Layer | Purpose | Example |
|---|---|---|
| **Experience API** | Tailored for a specific consumer (mobile app, partner portal) | `GET /mobile/v1/orders` |
| **Process API** | Orchestrates business logic across system APIs | `POST /process/v1/checkout` |
| **System API** | Thin wrapper around a backend system of record | `GET /erp/v1/inventory` |

This layering means changes to backend systems only affect System APIs — Experience and Process APIs remain stable for consumers. In this lab you will design a **System API** for a product catalogue.

```
Consumer (mobile app)
    ↓  Experience API  — /mobile/products
    ↓  Process API     — /process/product-search
    ↓  System API      — /sap/products  ← you are building this
    ↓  SAP / Database
```

> **Why RAML?** RAML (RESTful API Modeling Language) is a YAML-based specification language. Anypoint Platform uses it natively — the Design Center editor validates RAML in real time, generates mock servers, and can publish interactive documentation to Exchange automatically.

---

## Step 2 — Create the API in Design Center

Log in to Anypoint Platform and navigate to **Design Center → + Create → New API Spec**.

```
API Name:     products-system-api
API Language: RAML 1.0
```

In the editor, create `products-system-api.raml`:

```yaml
#%RAML 1.0
title: Products System API
version: v1
baseUri: https://api.example.com/{version}
mediaType: application/json

types:
  Product:
    type: object
    properties:
      id:         integer
      sku:        string
      name:       string
      price:      number
      stockLevel: integer
      category:   string

  ProductList:
    type: Product[]

  ErrorResponse:
    type: object
    properties:
      code:    string
      message: string

/products:
  get:
    description: Retrieve all products, optionally filtered by category.
    queryParameters:
      category:
        type:     string
        required: false
        example:  electronics
      limit:
        type:     integer
        required: false
        default:  20
        maximum:  100
    responses:
      200:
        body:
          application/json:
            type: ProductList
            example: !include examples/products-list.json
      500:
        body:
          application/json:
            type: ErrorResponse

  post:
    description: Create a new product.
    body:
      application/json:
        type: Product
    responses:
      201:
        body:
          application/json:
            type: Product
      400:
        body:
          application/json:
            type: ErrorResponse

/products/{productId}:
  uriParameters:
    productId:
      type:    integer
      example: 42
  get:
    description: Retrieve a single product by ID.
    responses:
      200:
        body:
          application/json:
            type: Product
      404:
        body:
          application/json:
            type: ErrorResponse
  put:
    description: Update an existing product.
    body:
      application/json:
        type: Product
    responses:
      200:
        body:
          application/json:
            type: Product
  delete:
    description: Delete a product.
    responses:
      204: {}
```

---

## Step 3 — Add Examples and a Data Type Library

Good API specs include realistic examples. Create `examples/products-list.json` in Design Center:

```json
[
  {
    "id": 1,
    "sku": "ELEC-001",
    "name": "Wireless Headphones",
    "price": 79.99,
    "stockLevel": 142,
    "category": "electronics"
  },
  {
    "id": 2,
    "sku": "ELEC-002",
    "name": "USB-C Hub",
    "price": 34.95,
    "stockLevel": 67,
    "category": "electronics"
  }
]
```

RAML 1.0 also supports extracting types into a reusable **library** — useful when the same types are shared across multiple API specs:

```yaml
# fragments/product-types.raml
#%RAML 1.0 Library

types:
  Product:
    type: object
    properties:
      id:
        type:     integer
        minimum:  1
      sku:
        type:     string
        pattern:  "^[A-Z]+-[0-9]+$"
      name:
        type:     string
        minLength: 1
        maxLength: 200
      price:
        type:    number
        minimum: 0
      stockLevel:
        type:    integer
        minimum: 0
      category:
        type: string
        enum: [electronics, clothing, food, books, other]
```

Import it at the top of the main spec:

```yaml
uses:
  types: fragments/product-types.raml
```

Then reference it as `types.Product`.

---

## Step 4 — Use the Mocking Service

Before implementing anything, Design Center can spin up a **Mocking Service** that returns your example responses. This lets front-end teams start building against the API while you implement the backend.

In Design Center, click **Publish to Exchange → Mocking Service**.

Test the mock:

```bash
# Replace <mock-url> with the URL shown in Design Center
curl -s "https://<mock-url>/api/v1/products" | python3 -m json.tool
```

You should get back the `products-list.json` example. Any request to a documented endpoint returns the matching example; undocumented endpoints return 404.

> **Tip:** Share the mock URL with your API consumers immediately. They can write integration tests against it before your real implementation is ready, dramatically shortening the feedback loop.

---

## Step 5 — Publish to Exchange and Import into Anypoint Studio

Publish the API spec to Anypoint Exchange so Anypoint Studio can import it:

```
Design Center → Publish → Publish to Exchange
  Asset version: 1.0.0
  API version:   v1
```

In Anypoint Studio, create a new project from the published spec:

```
File → New → Mule Project
  Project Name: products-system-api
  Import RAML from Exchange: products-system-api (1.0.0)
```

APIkit scaffolds the following structure automatically:

```
products-system-api/
└── src/main/mule/
    ├── products-system-api.xml        ← APIkit router + error handlers
    └── products-system-api-config.xml ← HTTP listener config
```

The generated XML includes:

```xml
<apikit:router config-ref="products-system-api-config" doc:name="APIkit Router"/>
```

And one flow per resource+method combination:

```
get:\products:products-system-api-config
post:\products:products-system-api-config
get:\products\{productId}:products-system-api-config
put:\products\{productId}:products-system-api-config
delete:\products\{productId}:products-system-api-config
```

---

## Step 6 — Implement the GET /products Flow

Each scaffolded flow starts with `<apikit:router>` routing to it and ends with a response that the router sends back to the HTTP listener. Implement the `GET /products` flow:

```xml
<flow name="get:\products:products-system-api-config">

    <!-- In the next lab we replace this with a real Database connector -->
    <set-payload
        value='#[[
            { id: 1, sku: "ELEC-001", name: "Wireless Headphones",
              price: 79.99, stockLevel: 142, category: "electronics" },
            { id: 2, sku: "ELEC-002", name: "USB-C Hub",
              price: 34.95, stockLevel: 67,  category: "electronics" }
        ]]'
        mimeType="application/json"
        doc:name="Mock product list"/>

    <!-- Filter by category query param if present -->
    <ee:transform doc:name="Filter by category">
        <ee:message>
            <ee:set-payload><![CDATA[%dw 2.0
output application/json
var category = attributes.queryParams.category
---
if (category != null)
    payload filter ($.category == category)
else
    payload]]>
            </ee:set-payload>
        </ee:message>
    </ee:transform>

</flow>
```

Run the application and test:

```bash
curl -s "http://localhost:8081/api/v1/products" | python3 -m json.tool
curl -s "http://localhost:8081/api/v1/products?category=electronics" | python3 -m json.tool
```

---

## Step 7 — Checkpoint and What's Next

You have completed the API-first design cycle:

- Defined a typed, documented RAML 1.0 specification for a Products System API
- Used the Design Center Mocking Service to share a working API before writing code
- Published the spec to Exchange and scaffolded an implementation project with APIkit
- Implemented a filtered `GET /products` flow using a DataWeave transform

In the next lab you will replace the hardcoded mock payload with a real **Database connector** that queries a PostgreSQL database, and explore **DataWeave 2.0** in depth — mapping, filtering, grouping, and reformatting data between systems.
