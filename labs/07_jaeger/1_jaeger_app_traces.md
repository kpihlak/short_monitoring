# Lab 1: App tracing

## Pre-requisites

Before you start:

- 🐳 Docker is installed (already in VM).
- 🖥️ You're inside a VSCode-connected VM.
- 📁 Create necessary folders:
  ```bash
  mkdir -p labs/tracing
  ```
- 📄 Create the YAML files:

### `web.yml`
```bash
cat <<EOF > labs/tracing/web.yml
version: '3.7'
services:
  fulfilment-web:
    image: courselabs/fulfilment-web
    ports:
      - "8070:80"
    networks:
      - fulfilment
networks:
  fulfilment:
EOF
```

### `api.yml`
```bash
cat <<EOF > labs/tracing/api.yml
version: '3.7'
services:
  fulfilment-api:
    image: courselabs/fulfilment-api
    ports:
      - "8071:80"
    environment:
      - OPENTRACING_JAEGER_ENABLED=true
    networks:
      - fulfilment
networks:
  fulfilment:
EOF
```

✅ You're ready!

# 🌟 Application Tracing Lab with Jaeger and OpenTelemetry

You’ll run a simple application, generate trace logs, and learn about context propagation.

# 📚 Application Traces

Tracing (along with Metrics and Logs) forms the three pillars of observability:
- Metrics → Health
- Logs → Events
- Traces → Communication flow

To trace, applications must emit spans and propagate context across services.

📖 References:
- [OpenTelemetry Specification](https://github.com/open-telemetry/opentelemetry-specification)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)

# 🚀 Step 1: Run the Web Application

Start the web app:
```bash
docker-compose -f labs/tracing/web.yml up -d
docker logs -f obsfun_fulfilment-web_1
```
Visit: [http://localhost:8070](http://localhost:8070)

🛠️ You’ll see trace spans in logs:

```
Activity.Id: 00-<trace-id>-<span-id>-01
Activity.DisplayName: /
Activity.Kind: Server
```

📋 **Checkpoint:** What does this `Activity` represent?  
> Hint: It records HTTP request details like method, path, response time.

# 🛠️ Step 2: Trigger an Error

Click “Go” on the site.  
You’ll see:

> ❌ **Documents service unavailable!**

📋 **Checkpoint:** What caused the error?

> Hint: The web app tries to call an API that isn’t running yet.

# 🚚 Step 3: Start the API Application

Start the API server:
```bash
docker-compose -f labs/tracing/api.yml up -d
docker logs -f obsfun_fulfilment-api_1
```
Visit: [http://localhost:8071/documents](http://localhost:8071/documents)

✅ Now the web app can connect to the API.

# 🛜 Context Propagation

Every HTTP call carries the trace context using `traceparent` headers.
- Clients send trace IDs.
- Servers read and forward them.

This links all spans into a full distributed trace.

# 🧪 Step 4: Inject Your Own Trace ID

Use `curl` to inject a custom trace:

```bash
curl -H "traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01" localhost:8070/index?all
```

Check the logs:
```bash
docker logs obsfun_fulfilment-web_1 -n 20
docker logs obsfun_fulfilment-api_1 -n 10
```

📋 **Checkpoint:** Did both apps use your custom trace ID?

> Hint: Parent span and child spans should have the same trace ID.

# 📜 W3C Trace Context Format

`traceparent: 00-<trace-id>-<span-id>-<flags>`

- `00`: version
- `trace-id`: 16-byte hex ID
- `span-id`: 8-byte hex ID
- `flags`: 01 means sampled

🔗 [Learn more](https://www.w3.org/TR/trace-context/)

# 🧹 Cleanup

Remove all running containers:

```bash
docker rm -f $(docker ps -aq)
```

# 🎁 (Optional) Visualize with Jaeger

Create `jaeger.yml`:
```bash
cat <<EOF > labs/tracing/jaeger.yml
version: '3.7'
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "6831:6831/udp"
    networks:
      - fulfilment
networks:
  fulfilment:
    external: true
EOF
```
Start Jaeger:
```bash
docker-compose -f labs/tracing/jaeger.yml up -d
```
Visit: [http://localhost:16686](http://localhost:16686)

✅ View your traces in Jaeger UI!

# 🎯 Summary

You learned:
- How to run instrumented apps
- How to view and understand traces
- How context propagation connects services
- How to inject traces manually
- (Bonus) How to visualize traces in Jaeger

# 🏆 Congratulations!

You now understand **Application Tracing** with **OpenTelemetry**, **Jaeger**, and the **W3C Trace Context**!

🎉 Happy tracing!