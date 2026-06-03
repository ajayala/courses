# Your First Mule Application

In this lab you will install Anypoint Studio, create your first Mule 4 project, and build a simple HTTP API flow that receives a request, transforms the payload, and returns a JSON response. Along the way you will learn how the Mule Event model works and how to use the built-in debugger to inspect message data at runtime.

**Prerequisites:** Java 8 or Java 11 JDK installed. At least 8 GB RAM. A free Anypoint Platform account (signup at anypoint.mulesoft.com).

---

## Step 1 — Install Anypoint Studio

Anypoint Studio is the Eclipse-based IDE for building, testing, and debugging Mule applications. Download it from the Anypoint Platform downloads page.

```bash
# On macOS with Homebrew Cask
brew install --cask anypoint-studio

# Or download manually and extract:
# https://www.mulesoft.com/lp/dl/anypoint-studio
```

After launching, select a workspace directory (e.g. `~/mule-workspace`) and sign in with your Anypoint Platform credentials via **Anypoint Studio → Preferences → Anypoint Studio → Authentication**.

Verify the bundled Mule runtime is available:

```
Help → About Anypoint Studio → Installation Details
```

You should see **Mule Server 4.x** listed as an installed feature.

> **Tip:** Anypoint Studio bundles its own JDK. If you see JDK errors on launch, set `JAVA_HOME` to your installed JDK 11 path and ensure it appears first in your `PATH`.

---

## Step 2 — Create a New Mule Project

Create your first project using the New Project wizard:

```
File → New → Mule Project
  Project Name:  hello-mule
  Mule Runtime:  4.x (bundled)
  Runtime:       Mule Server 4.x.x EE
```

Anypoint Studio generates the following project skeleton:

```
hello-mule/
├── src/
│   ├── main/
│   │   ├── mule/            ← flow XML files live here
│   │   └── resources/       ← application.properties, log4j2.xml
│   └── test/
│       ├── munit/           ← MUnit test flows
│       └── resources/
├── pom.xml                  ← Maven build file
└── mule-artifact.json       ← runtime descriptor
```

Every Mule application is a Maven project. `pom.xml` declares the Mule runtime BOM and connector dependencies. You can build and deploy from the command line as well as from the IDE.

---

## Step 3 — Build Your First HTTP Listener Flow

Open `src/main/mule/hello-mule.xml`. Drag components from the **Mule Palette** onto the canvas to build this flow:

```
HTTP Listener → Set Payload → Logger
```

Switch to the XML view and enter (or verify) this configuration:

```xml
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:http="http://www.mulesoft.org/schema/mule/http"
      xmlns:doc="http://www.mulesoft.org/schema/mule/documentation">

    <!-- Shared HTTP listener config — reused across flows -->
    <http:listener-config name="HTTP_Listener_config"
                          doc:name="HTTP Listener config">
        <http:listener-connection host="0.0.0.0" port="8081"/>
    </http:listener-config>

    <flow name="hello-flow">
        <http:listener
            config-ref="HTTP_Listener_config"
            path="/hello"
            doc:name="Listener"/>

        <set-payload
            value='#[{ "message": "Hello, " ++ (attributes.queryParams.name default "World") ++ "!", "timestamp": now() }]'
            mimeType="application/json"
            doc:name="Set Payload"/>

        <logger
            level="INFO"
            message='#["Request from: " ++ attributes.remoteAddress]'
            doc:name="Logger"/>
    </flow>
</mule>
```

Key concepts in this snippet:

| Component | Role |
|---|---|
| `http:listener-config` | Global, reusable connection config (host + port) |
| `http:listener` | Entry point — listens on `GET /hello` |
| `set-payload` | Replaces the event payload; uses DataWeave expression |
| `logger` | Writes to the console log; does not modify the event |

> **DataWeave expressions** are written inside `#[ ]`. `attributes.queryParams.name` accesses the `?name=` query parameter. `default` provides a fallback if the parameter is absent.

---

## Step 4 — Understand the Mule Event Model

Every message flowing through a Mule application is wrapped in a **Mule Event**. Understanding its structure is fundamental to writing DataWeave expressions and routing logic.

```
Mule Event
├── payload          — the message body (JSON, XML, bytes, Java object, etc.)
├── attributes       — metadata from the inbound source (HTTP headers, query params, URI params, method)
├── variables        — flow-scoped key/value store (set with Set Variable)
├── error            — populated only inside an error handler scope
└── authentication   — security context (populated by security connectors)
```

You can inspect the live event at any point in a flow using the **Mule Debugger**. Add a breakpoint by right-clicking a component on the canvas and selecting **Add Breakpoint**:

```xml
<!-- Add this after set-payload to inspect the transformed event -->
<logger
    level="DEBUG"
    message='#["Payload type: " ++ typeOf(payload)]'
    doc:name="Debug Logger"/>
```

> **`payload` vs `attributes`:** A common mistake is trying to access HTTP headers via `payload`. HTTP headers and query parameters are always in `attributes` — `payload` is only the request body. For a `GET` request with no body, `payload` will be `null`.

---

## Step 5 — Run and Test the Application

Run the application from Anypoint Studio:

```
Right-click hello-mule project → Run As → Mule Application
```

Watch the console for the startup message:

```
INFO  ...MuleContext: Mule is up and kicking (every 5 seconds)
INFO  ...Server: Started ServerConnector ... {http://0.0.0.0:8081}
```

Test the endpoint with curl:

```bash
# Without name parameter — uses default "World"
curl -s "http://localhost:8081/hello" | python3 -m json.tool

# With name parameter
curl -s "http://localhost:8081/hello?name=Alice" | python3 -m json.tool
```

Expected response:

```json
{
    "message": "Hello, Alice!",
    "timestamp": "2026-06-03T10:23:45.123Z"
}
```

Test an error case — request a path that doesn't exist:

```bash
curl -v "http://localhost:8081/goodbye"
```

You should receive a `404 Not Found` with a Mule error body. You will learn to customise error responses in Module 3.

---

## Step 6 — Add a Flow Variable and Sub-Flow

Flow variables survive for the lifetime of a single flow execution. Use `Set Variable` to store intermediate values so you don't need to recompute them.

Add a sub-flow that formats a greeting message, then call it from the main flow:

```xml
<flow name="hello-flow">
    <http:listener config-ref="HTTP_Listener_config" path="/hello"/>

    <set-variable
        variableName="callerName"
        value='#[attributes.queryParams.name default "World"]'
        doc:name="Store caller name"/>

    <flow-ref name="build-greeting-subflow" doc:name="Build greeting"/>

    <logger level="INFO" message='#["Served greeting to: " ++ vars.callerName]'/>
</flow>

<sub-flow name="build-greeting-subflow">
    <set-payload
        value='#[{ "message": "Hello, " ++ vars.callerName ++ "!", "timestamp": now() }]'
        mimeType="application/json"
        doc:name="Build response"/>
</sub-flow>
```

`vars.callerName` accesses the flow variable from the sub-flow — variables are inherited by called sub-flows.

```bash
curl -s "http://localhost:8081/hello?name=Bob" | python3 -m json.tool
```

---

## Step 7 — Checkpoint and What's Next

You have built and run your first Mule 4 application:

- Installed Anypoint Studio and created a Maven-based Mule project
- Wired together an HTTP Listener, Set Payload, and Logger using the canvas and XML editor
- Understood the Mule Event model — payload, attributes, and variables
- Used DataWeave inline expressions to build a dynamic JSON response
- Split logic across a main flow and a reusable sub-flow

In the next module you will step up to the full API design lifecycle — writing a formal RAML specification in Anypoint Design Center, then using APIkit to scaffold and implement the API in Anypoint Studio.
