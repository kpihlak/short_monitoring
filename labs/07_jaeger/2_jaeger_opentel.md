# Lab 2: Distributed Tracing with Jaeger and OpenTelemetry

In this lab, you'll set up Jaeger to collect and visualize traces from a sample application. You'll explore distributed tracing through Jaeger's web UI and investigate how spans are collected across multiple services.

## Step 1: Run Jaeger

We will start by launching a **simple Jaeger instance** where all components run in one container.

> labs/jaeger/jaeger.yml
```yaml
version: '3.7'

services:
  jaeger:
    image: courselabs/jaeger:1.24
    ports:
      - "16686:16686"
    networks:
      - tracing  

networks:
  tracing:
    name: obsfun-tracing
```

**Command:**
```bash
docker-compose -f labs/jaeger/jaeger.yml up -d
```

Open your browser and go to:
```
http://localhost:16686
```

📋 **Exercise 1:**
Select the `jaeger-query` service from the dropdown, click **Find Traces**, and open the first trace.

```hint
Jaeger records traces itself - you are seeing the call to the API from the web UI to list stored operations.
Tags like `http.method`, `http.status_code`, and `span.kind` give you useful metadata.
Switch to Trace Statistics or Trace JSON to see alternative data views.
```

## Step 2: Run the Application

The application has three components:
- Web frontend
- Document service API
- Authorization service API

Configured to send spans to Jaeger via OpenTelemetry.

> labs/jaeger/apps.yml
```yaml
version: '3.7'

services:

  fulfilment-api:
    image: courselabs/fulfilment-api
    ports:
      - "8071:80"
    environment:
      - OPENTRACING_JAEGER_ENABLED=true
    networks:
      - tracing 
      - fulfilment 

  fulfilment-authz:
    image: courselabs/fulfilment-authz
    ports:
      - "8072:80"
    environment:
      - Observability__Trace__Console=false
      - Observability__Trace__Jaeger=true
    networks:
      - tracing 
      - fulfilment 

  fulfilment-web:
    image: courselabs/fulfilment-web
    ports:
      - "8070:80"
    environment:
      - Observability__Trace__Console=false
      - Observability__Trace__Jaeger=true
      - Documents__Authz__Url=http://fulfilment-authz/check
    depends_on:
      - fulfilment-api
      - fulfilment-authz
    networks:
      - tracing 
      - fulfilment 

networks:
  fulfilment:
  tracing:
    name: obsfun-tracing
```

**Command:**
```bash
docker-compose -f labs/jaeger/apps.yml up -d
```

Web app:
```
http://localhost:8070
```

📋 **Exercise 2:**
Open the trace for the homepage request to `Fulfilment.Web`.

```hint
There is likely just one span - a single request and response.
Traces with a single span are valid if no further calls happen.
```
## Step 3: Explore More Complex Traces

1. Navigate to the **Document List** page in the app.
2. Click **Go** using default settings.

📋 **Exercise 3:**
Find the trace for this interaction.

```hint
You'll now see multiple services involved!
Expand all spans:
- Web receives POST from user
- Web calls Authorization API (client+server spans)
- Authorization API makes an external GET request
- Web calls Document API (client+server spans)

Longest call is the external blog request.
```
## Step 4: Submit a Document

1. Go to the **Submit Document** page.
2. Click **Go**.

📋 **Exercise 4:**
Find the call stack for this request.

```hint
Trace call flow:
- POST /Submit (Web app)
- Web -> Authz (check permissions)
- Authz -> external site (identity check)
- Web -> Document API (submit document)

Find the final span (Document API POST) and check logs:
- handler.class_simple_name: DocumentsController
- handler.method_name: submitDocument
```

## Cleanup

To stop and remove all containers:
```bash
docker rm -f $(docker ps -aq)
```

# References
- [Jaeger official site](https://www.jaegertracing.io/)
- [Jaeger architecture](https://www.jaegertracing.io/docs/1.24/architecture/)
- [Getting Started with Jaeger](https://www.jaegertracing.io/docs/1.24/getting-started/#all-in-one)
- [OpenTelemetry instrumentation](https://opentelemetry.io/docs/concepts/instrumenting/)
