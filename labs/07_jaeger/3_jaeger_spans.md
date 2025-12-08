# Lab 3: Custom Spans and Baggage Lab

## Reference Materials
- [OpenTelemetry Tracing API](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/api.md)
- [Tags, Logs, and Baggage Overview](https://opentracing.io/docs/overview/tags-logs-baggage/)
- [Increasing Trace Cardinality Blog](https://jimmybogard.com/increasing-trace-cardinality-with-tags-and-baggage/)

## 🛠 **1. Setup: Start Services with Custom Spans**

Use the provided `compose.yml` to launch the environment:

**compose.yml**

```yaml
version: '3.7'

services:

  jaeger:
    image: courselabs/jaeger:1.24
    ports:
      - "16686:16686"
    networks:
      - tracing  

  fulfilment-api:
    image: courselabs/fulfilment-api
    ports:
      - "8071:80"
    environment:
      - OPENTRACING_JAEGER_ENABLED=true
      - TRACE_CUSTOM_SPANS=true
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
      - Observability__Trace__CustomSpans=true
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

### ▶️ Start containers:

```bash
docker-compose -f labs/custom-spans/compose.yml up -d
```

## 🌐 **2. Access the Application**

- Web app: [http://localhost:8070](http://localhost:8070)
- Jaeger UI: [http://localhost:16686](http://localhost:16686)

Click **Go** on the web app to load the document list.

## 🔍 **3. Explore Traces**

In Jaeger UI:
- Service: **Fulfilment.Web**
- Operation: (Any)
- Look for a trace with **10 spans**.

You’ll find custom spans:

| Span Name             | Notes |
|:----------------------|:------|
| `list-documents`       | Groups the entire request |
| `authz-check`          | Around authorization call (tags: user ID, authz type, authz response) |
| `document-api-call`    | Around document service call (tags: API action type) |

## ✍️ **4. Custom Span Code Examples**

**C# (ListDocumentsService.cs)**  
👉 [View on GitHub](https://observability.courselabs.co/src/fulfilment-frontend/fulfilment-web/Services/ListDocumentsService.cs)

```csharp
apiSpan = _activitySource.StartActivity("document-api-call");
apiSpan.AddTag("span.kind", "internal")
       .AddTag("action.type", $"{DocumentAction.List}");
```

**Java (DocumentsController.java)**  
👉 [View on GitHub](https://github.com/courselabs/fulfilment/blob/main/src/fulfilment-api/src/main/java/com/obsfun/controllers/DocumentsController.java#L95)

```java
dbLoadSpan = tracer.buildSpan("database-load").start();
dbLoadSpan.setTag("span.kind", "internal");
```

Both examples use:
- `try/finally` to ensure spans close properly even on errors.

## 🧠 **5. Find Problems Using Tags**

Change the User ID to `1042` in the web app. Click **Go**.

You will get a **Documents service unavailable** error.

In Jaeger UI:

1. Search Service: **Fulfilment.Web**
2. Add tag:  
   ```text
   user.id = 1042
   ```
3. Find the trace.

You’ll see:
- AuthZ span shows a **denied authorization** tag.

---

## 🛠 **6. Setup Baggage for Full Trace Context**

Update with baggage support:

**update.yml**

```yaml
version: '3.7'

services:

  fulfilment-api:
    image: courselabs/fulfilment-api
    ports:
      - "8071:80"
    environment:
      - OPENTRACING_JAEGER_ENABLED=true
      - TRACE_CUSTOM_SPANS=true
      - TRACE_BAGGAGE_TAG=true
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
      - Observability__Trace__Baggage__Tag=true
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
      - Observability__Trace__CustomSpans=true
      - Observability__Trace__Baggage__Tag=true
    depends_on:
      - fulfilment-authz
    networks:
      - tracing 
      - fulfilment 

networks:
  fulfilment:
  tracing:
    name: obsfun-tracing
```

▶️ Update containers:

```bash
docker-compose -f labs/custom-spans/update.yml up -d
```
## 🧩 **7. Explore Traces with Baggage**

Browse the document list page again (user `0421`).

In Jaeger, you’ll now see **transaction.id** attached (copied as a tag from baggage)!

## ✨ **8. BONUS: Skipping Authorization Using Baggage**

Use a `curl` command to force skip authorization:

```bash
curl -H "baggage: authz.skip=true" "http://localhost:8070/index?userId=1024"
```

- You’ll **successfully get the documents** even though normally user `1024` is unauthorized.
- The `authz-check` span in Jaeger will show **authorization skipped**.

## 📄 **Summary: Key Concepts**

| Feature | Use |
|:--------|:----|
| Custom Spans | Group related operations, add logical visibility |
| Tags | Add searchable context to spans |
| Baggage | Attach metadata that travels through call chains (headers) |
| Skipping Authorization | Dangerous but possible via baggage manipulation |

# 🔥 You're Done!