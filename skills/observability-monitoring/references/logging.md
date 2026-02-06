# Logging

## Structured Logging

### Python Implementation

```python
import structlog
import logging
import sys
from datetime import datetime

# Configure logging
logging.basicConfig(
    format="%(message)s",
    stream=sys.stdout,
    level=logging.INFO,
)

# Configure structlog
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

# Get logger
logger = structlog.get_logger()

# Basic logging
logger.info("application_started", version="1.0.0", environment="production")

# Error logging with exception info
try:
    risky_operation()
except Exception as e:
    logger.error("operation_failed",
                error=str(e),
                error_type=type(e).__name__,
                exc_info=True)

# Context binding
log = logger.bind(user_id="123", request_id="abc-456")
log.info("user_action", action="login", ip="192.168.1.1")
log.info("user_action", action="view_profile")
```

### JavaScript/Node.js

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp({ format: 'ISO8601' }),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'order-service' },
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

// Usage
logger.info('order_created', { orderId: '123', userId: '456', total: 100.00 });
logger.error('payment_failed', { 
  orderId: '123', 
  error: 'Insufficient funds',
  retryCount: 3 
});
```

## Log Aggregation with ELK Stack

### Filebeat Configuration

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/myapp/*.json
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      service: order-service
      environment: production
    fields_under_root: true

output.elasticsearch:
  hosts: ["http://elasticsearch:9200"]
  index: "myapp-%{+yyyy.MM.dd}"

logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
```

### Logstash Pipeline

```ruby
# logstash.conf
input {
  beats {
    port => 5044
  }
}

filter {
  # Parse timestamp
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
  }
  
  # Add environment field if missing
  if ![environment] {
    mutate {
      add_field => { "environment" => "unknown" }
    }
  }
  
  # Parse user agent
  if [user_agent] {
    useragent {
      source => "user_agent"
      target => "user_agent_parsed"
    }
  }
  
  # GeoIP lookup for IP addresses
  if [client_ip] {
    geoip {
      source => "client_ip"
      target => "geoip"
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "%{[service]}-%{+yyyy.MM.dd}"
  }
}
```

### Fluentd Configuration

```xml
# fluent.conf
<source>
  @type tail
  path /var/log/myapp/*.json
  pos_file /var/log/fluent/myapp.log.pos
  tag myapp
  <parse>
    @type json
    time_key timestamp
    time_format %iso8601
  </parse>
</source>

<filter myapp>
  @type record_transformer
  <record>
    hostname ${hostname}
  </record>
</filter>

<match myapp>
  @type elasticsearch
  host elasticsearch
  port 9200
  index_name myapp
  type_name _doc
  logstash_format true
  logstash_prefix myapp
  flush_interval 10s
</match>
```

## Log Levels Guide

```python
# DEBUG - Detailed information for debugging
logger.debug("database_query", 
             sql="SELECT * FROM users WHERE id = %s",
             params=[123],
             duration_ms=5.2)

# INFO - General information
logger.info("user_login", 
            user_id="123", 
            method="oauth",
            ip="192.168.1.1")

# WARNING - Something unexpected but not an error
logger.warning("slow_query",
              sql="SELECT * FROM large_table",
              duration_ms=2500,
              threshold_ms=1000)

# ERROR - Error occurred but application can continue
logger.error("payment_failed",
            order_id="456",
            error="Card declined",
            retry_count=2)

# CRITICAL - Serious error, application may not be able to continue
logger.critical("database_connection_lost",
               database="primary",
               attempts=5,
               action="switching_to_replica")
```

## Log Rotation

```python
# Python logging with rotation
from logging.handlers import RotatingFileHandler, TimedRotatingFileHandler

# Size-based rotation
file_handler = RotatingFileHandler(
    'app.log',
    maxBytes=10*1024*1024,  # 10MB
    backupCount=5
)

# Time-based rotation
file_handler = TimedRotatingFileHandler(
    'app.log',
    when='midnight',
    interval=1,
    backupCount=30
)
```

```yaml
# Logrotate configuration
/var/log/myapp/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0644 appuser appgroup
    sharedscripts
    postrotate
        /bin/kill -HUP $(cat /var/run/myapp.pid 2>/dev/null) 2>/dev/null || true
    endscript
}
```
