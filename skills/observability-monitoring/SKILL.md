---
name: observability-monitoring
description: Observability and monitoring patterns including logging, metrics, distributed tracing, alerting, and monitoring stack setup. Use when setting up monitoring, implementing logging strategies, configuring alerts, or debugging production issues.
license: MIT
compatibility: opencode
---

# Observability & Monitoring Skill

## Overview

This skill provides comprehensive observability patterns including structured logging, metrics collection, distributed tracing, alerting, and setting up monitoring stacks (Prometheus, Grafana, ELK, Jaeger).

## Quick Start

### Observability Checklist

- [ ] Structured logging implemented
- [ ] Request/response logging
- [ ] Error tracking with context
- [ ] Performance metrics exposed
- [ ] Health checks configured
- [ ] Distributed tracing enabled
- [ ] Alerts configured
- [ ] Dashboards created

### Three Pillars of Observability

1. **Logs** - Discrete events (what happened)
2. **Metrics** - Numeric data over time (how much/how often)
3. **Traces** - Request flow through system (where time spent)

## Structured Logging

### Python with structlog

```python
import structlog
import logging
from datetime import datetime

# Configure structured logging
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    wrapper_class=structlog.stdlib.BoundLogger,
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# Usage
logger.info("user_login", user_id="123", ip="192.168.1.1", user_agent="Mozilla/5.0")
logger.error("database_connection_failed", 
             error="Connection timeout", 
             retry_count=3,
             connection_string="postgresql://localhost:5432/mydb")
```

### Request Logging Middleware

```python
import time
import uuid
from fastapi import Request

class RequestLoggingMiddleware:
    async def __call__(self, request: Request, call_next):
        request_id = str(uuid.uuid4())
        start_time = time.time()
        
        # Bind request context
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            request_id=request_id,
            method=request.method,
            path=request.url.path,
            user_agent=request.headers.get('user-agent'),
            ip=request.client.host
        )
        
        logger.info("request_started")
        
        try:
            response = await call_next(request)
            
            duration = time.time() - start_time
            logger.info("request_completed",
                       status_code=response.status_code,
                       duration_ms=round(duration * 1000, 2))
            
            response.headers['X-Request-ID'] = request_id
            return response
            
        except Exception as e:
            duration = time.time() - start_time
            logger.error("request_failed",
                        error=str(e),
                        error_type=type(e).__name__,
                        duration_ms=round(duration * 1000, 2))
            raise
```

## Metrics

### Prometheus Metrics

```python
from prometheus_client import Counter, Histogram, Gauge, Info, generate_latest
from fastapi import FastAPI, Response

app = FastAPI()

# Define metrics
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status_code']
)

REQUEST_DURATION = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

ACTIVE_CONNECTIONS = Gauge(
    'active_connections',
    'Number of active connections'
)

APP_INFO = Info('app', 'Application information')

# Set app info
APP_INFO.info({'version': '1.0.0', 'build_date': '2024-01-15'})

@app.middleware("http")
async def metrics_middleware(request, call_next):
    ACTIVE_CONNECTIONS.inc()
    
    start_time = time.time()
    response = await call_next(request)
    duration = time.time() - start_time
    
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path,
        status_code=response.status_code
    ).inc()
    
    REQUEST_DURATION.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)
    
    ACTIVE_CONNECTIONS.dec()
    return response

@app.get("/metrics")
def metrics():
    return Response(generate_latest(), media_type="text/plain")
```

## Distributed Tracing

### OpenTelemetry

```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

# Configure tracing
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Export to Jaeger/Tempo
otlp_exporter = OTLPSpanExporter(endpoint="http://jaeger:4317")
span_processor = BatchSpanProcessor(otlp_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

# Instrument FastAPI
app = FastAPI()
FastAPIInstrumentor.instrument_app(app)
RequestsInstrumentor().instrument()

# Manual instrumentation
@app.get("/orders/{order_id}")
async def get_order(order_id: str):
    with tracer.start_as_current_span("get_order") as span:
        span.set_attribute("order.id", order_id)
        
        with tracer.start_as_current_span("fetch_from_database"):
            order = await db.get_order(order_id)
        
        with tracer.start_as_current_span("enrich_order_data"):
            order['user'] = await user_service.get_user(order['user_id'])
        
        span.set_attribute("order.status", order['status'])
        return order
```

## Detailed References

See comprehensive guides in references/:

- **[Logging](references/logging.md)** - Structured logging, log aggregation, ELK stack
- **[Metrics](references/metrics.md)** - Prometheus, Grafana dashboards, custom metrics
- **[Tracing](references/tracing.md)** - OpenTelemetry, Jaeger, distributed tracing patterns
- **[Alerting](references/alerting.md)** - Prometheus Alertmanager, PagerDuty, SLOs/SLIs

## When to Use This Skill

Use this skill when:
- Setting up application monitoring
- Implementing structured logging
- Configuring metrics collection
- Debugging production issues
- Setting up distributed tracing
- Creating dashboards and alerts
- Defining SLOs and SLIs
- Investigating performance issues

## Related Skills

- `@kubernetes-patterns` - Kubernetes monitoring and logging
- `@docker-patterns` - Container monitoring
- `@microservices-patterns` - Distributed tracing for microservices
- `@performance-optimization` - Performance profiling
- `@security-best-practices` - Security monitoring
