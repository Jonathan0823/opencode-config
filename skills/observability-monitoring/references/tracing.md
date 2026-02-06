# Distributed Tracing

## OpenTelemetry Setup

### Python

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

# Configure provider
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Configure exporter
otlp_exporter = OTLPSpanExporter(endpoint="http://jaeger:4317")
span_processor = BatchSpanProcessor(otlp_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

# Instrument frameworks
app = FastAPI()
FastAPIInstrumentor.instrument_app(app)
RequestsInstrumentor().instrument()
```

### Manual Instrumentation

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

@app.get("/orders/{order_id}")
async def get_order(order_id: str):
    # Create span
    with tracer.start_as_current_span("get_order") as span:
        # Add attributes
        span.set_attribute("order.id", order_id)
        span.set_attribute("order.source", "api")
        
        # Nested span for database query
        with tracer.start_as_current_span("fetch_from_db") as db_span:
            db_span.set_attribute("db.system", "postgresql")
            db_span.set_attribute("db.statement", "SELECT * FROM orders WHERE id = %s")
            order = await db.get_order(order_id)
        
        # Nested span for external call
        with tracer.start_as_current_span("fetch_user_data") as user_span:
            user_span.set_attribute("http.url", "http://user-service/users/{}")
            user_span.set_attribute("http.method", "GET")
            
            async with aiohttp.ClientSession() as session:
                async with session.get(f"http://user-service/users/{order['user_id']}") as resp:
                    user_data = await resp.json()
                    user_span.set_attribute("http.status_code", resp.status)
        
        # Add events
        span.add_event("order_enriched", {
            "item_count": len(order['items']),
            "total_value": order['total']
        })
        
        return order
```

### Context Propagation

```python
from opentelemetry.propagate import extract, inject
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator

# Extract context from incoming request
def process_message(headers):
    context = extract(lambda key: headers.get(key))
    
    with tracer.start_as_current_span("process_message", context=context):
        # Process message
        pass

# Inject context into outgoing request
def call_external_service():
    headers = {}
    inject(headers)
    
    response = requests.get(
        "http://external-service/api",
        headers=headers
    )
    return response
```

## Jaeger

### Docker Compose

```yaml
version: '3.8'
services:
  jaeger:
    image: jaegertracing/all-in-one:1.45
    ports:
      - "16686:16686"  # UI
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
      - "9411:9411"    # Zipkin
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.45
          ports:
            - containerPort: 16686
            - containerPort: 4317
          env:
            - name: COLLECTOR_OTLP_ENABLED
              value: "true"
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger
spec:
  selector:
    app: jaeger
  ports:
    - name: ui
      port: 16686
      targetPort: 16686
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
```

## Tracing Best Practices

### Span Naming

```python
# Good - Action + target
"get_order"
"update_user_profile"
"send_email_notification"

# Bad - Vague names
"process"
"handle_request"
"do_work"
```

### Attributes

```python
# Standard attributes
span.set_attribute("http.method", "POST")
span.set_attribute("http.url", "http://api.example.com/users")
span.set_attribute("http.status_code", 200)
span.set_attribute("http.response_size", 1024)

# Database attributes
span.set_attribute("db.system", "postgresql")
span.set_attribute("db.name", "myapp")
span.set_attribute("db.statement", "SELECT * FROM users WHERE id = %s")
span.set_attribute("db.operation", "SELECT")

# Business attributes
span.set_attribute("order.id", "12345")
span.set_attribute("user.id", "67890")
span.set_attribute("payment.amount", 99.99)
span.set_attribute("payment.currency", "USD")
```

### Events

```python
# Add events for important milestones
span.add_event("cache_miss", {"key": user_id})
span.add_event("database_query_completed", {"duration_ms": 25})
span.add_event("validation_failed", {"errors": ["email_invalid"]})

# Exception events
import traceback

try:
    risky_operation()
except Exception as e:
    span.record_exception(e)
    span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
```

## Sampling

### Head-Based Sampling

```python
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased, ALWAYS_ON, ALWAYS_OFF

# Sample 10% of traces
sampler = TraceIdRatioBased(0.1)

provider = TracerProvider(sampler=sampler)
trace.set_tracer_provider(provider)
```

### Tail-Based Sampling (Jaeger)

```yaml
# jaeger-collector.yml
collector:
  sampling:
    strategies:
      - service: order-service
        type: probabilistic
        param: 0.1
      - service: payment-service
        type: probabilistic
        param: 1.0  # Sample all
```

## Trace Analysis

### Critical Path

```python
# Identify bottlenecks in trace
def analyze_trace(trace):
    spans = trace.spans
    
    # Find longest span
    longest = max(spans, key=lambda s: s.duration)
    
    # Find spans with high latency
    slow_spans = [s for s in spans if s.duration > 1000]  # > 1s
    
    return {
        'critical_path': longest,
        'slow_operations': slow_spans,
        'total_duration': sum(s.duration for s in spans)
    }
```
