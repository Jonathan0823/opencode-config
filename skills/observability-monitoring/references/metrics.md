# Metrics

## Prometheus Metrics

### Metric Types

```python
from prometheus_client import Counter, Histogram, Gauge, Summary, Info

# Counter - Only increases (e.g., requests, errors)
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status_code']
)

# Gauge - Can increase or decrease (e.g., temperature, queue size)
ACTIVE_CONNECTIONS = Gauge(
    'active_connections',
    'Number of active connections',
    ['service']
)

# Histogram - Samples observations (e.g., request duration, response sizes)
REQUEST_DURATION = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration in seconds',
    ['method', 'endpoint'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
)

# Summary - Similar to histogram but calculates quantiles
REQUEST_LATENCY = Summary(
    'http_request_latency_seconds',
    'Request latency',
    ['method']
)

# Info - Static information
APP_INFO = Info('app', 'Application information')
```

### Application Instrumentation

```python
from prometheus_client import start_http_server, Counter, Histogram, Gauge
from functools import wraps
import time

# Define metrics
REQUESTS = Counter('app_requests_total', 'Total requests', ['method', 'endpoint'])
LATENCY = Histogram('app_request_latency_seconds', 'Request latency')
IN_PROGRESS = Gauge('app_requests_in_progress', 'Requests in progress')
ERRORS = Counter('app_errors_total', 'Total errors', ['type'])

# Decorator for automatic instrumentation
def monitor_request(endpoint):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            IN_PROGRESS.inc()
            start_time = time.time()
            
            try:
                result = func(*args, **kwargs)
                REQUESTS.labels(method='GET', endpoint=endpoint).inc()
                return result
            except Exception as e:
                ERRORS.labels(type=type(e).__name__).inc()
                raise
            finally:
                LATENCY.observe(time.time() - start_time)
                IN_PROGRESS.dec()
        
        return wrapper
    return decorator

# Usage
@monitor_request('/api/users')
def get_users():
    return db.query(User).all()

# Custom business metrics
ORDERS_CREATED = Counter('orders_created_total', 'Total orders created')
ORDER_VALUE = Histogram('order_value_dollars', 'Order value', buckets=[10, 25, 50, 100, 250, 500, 1000])

def create_order(items, total):
    order = Order(items=items, total=total)
    db.session.add(order)
    db.session.commit()
    
    ORDERS_CREATED.inc()
    ORDER_VALUE.observe(total)
```

## Grafana Dashboards

### Dashboard JSON

```json
{
  "dashboard": {
    "title": "Application Metrics",
    "tags": ["production", "myapp"],
    "timezone": "browser",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{endpoint}}"
          }
        ],
        "yAxes": [
          {
            "label": "Requests/sec",
            "min": 0
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total{status_code=~\"5..\"}[5m]) / rate(http_requests_total[5m])",
            "legendFormat": "Error %"
          }
        ],
        "yAxes": [
          {
            "label": "Error Rate",
            "min": 0,
            "max": 1,
            "format": "percentunit"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "p95"
          },
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "p99"
          }
        ],
        "yAxes": [
          {
            "label": "Seconds",
            "format": "s"
          }
        ]
      }
    ]
  }
}
```

## Custom Metrics

### Business Metrics

```python
from prometheus_client import Counter, Histogram

# Business KPIs
SIGNUPS = Counter('user_signups_total', 'Total user signups', ['source'])
PURCHASES = Counter('purchases_total', 'Total purchases', ['category'])
REVENUE = Counter('revenue_dollars_total', 'Total revenue')
CART_ABANDONMENT = Gauge('cart_abandonment_rate', 'Cart abandonment rate')

# Usage
def register_user(source: str):
    # ... registration logic
    SIGNUPS.labels(source=source).inc()

def complete_purchase(cart):
    # ... purchase logic
    PURCHASES.labels(category=cart.category).inc()
    REVENUE.inc(cart.total)
```

### Infrastructure Metrics

```python
import psutil
from prometheus_client import Gauge

# System metrics
CPU_USAGE = Gauge('system_cpu_usage_percent', 'CPU usage percentage')
MEMORY_USAGE = Gauge('system_memory_usage_bytes', 'Memory usage in bytes')
DISK_USAGE = Gauge('system_disk_usage_percent', 'Disk usage percentage', ['path'])

def collect_system_metrics():
    CPU_USAGE.set(psutil.cpu_percent())
    MEMORY_USAGE.set(psutil.virtual_memory().used)
    DISK_USAGE.labels(path='/').set(psutil.disk_usage('/').percent)
```

## RED Method

```python
# Rate - Requests per second
REQUEST_RATE = Gauge('request_rate', 'Requests per second')

# Errors - Error rate
ERROR_RATE = Gauge('error_rate_percent', 'Error rate percentage')

# Duration - Request duration
REQUEST_DURATION = Histogram('request_duration_seconds', 'Request duration')

# Implementation
import time
from functools import wraps

def red_metrics(endpoint: str):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            start = time.time()
            try:
                result = func(*args, **kwargs)
                REQUEST_RATE.set(1)  # Increment counter
                return result
            except Exception:
                ERRORS.labels(endpoint=endpoint).inc()
                raise
            finally:
                duration = time.time() - start
                REQUEST_DURATION.observe(duration)
        return wrapper
    return decorator
```

## USE Method

```python
# Utilization - % time resource is busy
CPU_UTILIZATION = Gauge('cpu_utilization_percent', 'CPU utilization')

# Saturation - Amount of work resource has to do, often queue length
REQUEST_QUEUE_SIZE = Gauge('request_queue_size', 'Number of queued requests')

# Errors - Count of error events
RESOURCE_ERRORS = Counter('resource_errors_total', 'Resource errors', ['type'])
```
