---
name: microservices-patterns
description: Microservices architecture patterns including service discovery, circuit breakers, event-driven architecture, API gateways, and distributed systems. Use when designing microservices, implementing inter-service communication, or managing distributed systems.
license: MIT
compatibility: opencode
---

# Microservices Patterns Skill

## Overview

This skill provides comprehensive microservices architecture patterns including service discovery, circuit breakers, event-driven architecture, saga patterns, API gateways, and distributed data management.

## Quick Start

### Microservices Checklist

- [ ] Service boundaries clearly defined
- [ ] Database per service
- [ ] API Gateway for external communication
- [ ] Async messaging between services
- [ ] Circuit breakers for resilience
- [ ] Distributed tracing implemented
- [ ] Centralized logging
- [ ] Health checks and monitoring

### Communication Patterns

**Synchronous (REST/gRPC):**
- Simple request/response
- Tight coupling
- Blocking
- Use for: Queries, simple operations

**Asynchronous (Message Queue):**
- Event-driven
- Loose coupling
- Non-blocking
- Use for: Commands, long operations, event sourcing

## Core Patterns

### API Gateway

```yaml
# Kong Gateway configuration
services:
  - name: user-service
    url: http://user-service:8080
    routes:
      - name: user-routes
        paths:
          - /api/users
    plugins:
      - name: rate-limiting
        config:
          minute: 100
      - name: jwt
      - name: cors
```

### Circuit Breaker

```python
from circuitbreaker import circuit
import requests

@circuit(failure_threshold=5, recovery_timeout=30, expected_exception=requests.RequestException)
def call_external_service():
    response = requests.get('http://external-service/api')
    return response.json()

# Usage
try:
    result = call_external_service()
except circuitbreaker.CircuitBreakerError:
    # Circuit is open, use fallback
    result = fallback_data()
```

### Service Discovery

```yaml
# Consul service registration
service:
  name: order-service
  tags:
    - orders
    - v1
  port: 8080
  check:
    http: http://localhost:8080/health
    interval: 10s
```

### Saga Pattern

```python
# Choreography-based saga
class OrderSaga:
    def __init__(self, event_bus):
        self.event_bus = event_bus
    
    def start_order(self, order_data):
        # Step 1: Create order (pending)
        order = create_pending_order(order_data)
        
        # Step 2: Reserve inventory
        self.event_bus.publish('InventoryReservationRequested', {
            'order_id': order.id,
            'items': order.items
        })
    
    def on_inventory_reserved(self, event):
        # Step 3: Process payment
        self.event_bus.publish('PaymentRequested', {
            'order_id': event.order_id,
            'amount': event.total
        })
    
    def on_payment_completed(self, event):
        # Step 4: Complete order
        complete_order(event.order_id)
        self.event_bus.publish('OrderCompleted', {
            'order_id': event.order_id
        })
    
    def on_inventory_failed(self, event):
        # Compensate: Cancel order
        cancel_order(event.order_id)
```

## Detailed References

See comprehensive guides in references/:

- **[Service Communication](references/service-communication.md)** - REST, gRPC, GraphQL, message queues
- **[Resilience Patterns](references/resilience.md)** - Circuit breakers, retries, bulkhead, timeouts
- **[Event-Driven Architecture](references/event-driven.md)** - Event sourcing, CQRS, saga patterns
- **[Data Management](references/data-management.md)** - Database per service, distributed transactions, eventual consistency

## When to Use This Skill

Use this skill when:
- Decomposing monolith into microservices
- Designing service boundaries and APIs
- Implementing inter-service communication
- Managing distributed data consistency
- Building resilient service architectures
- Setting up service discovery and load balancing
- Implementing API gateways
- Handling distributed transactions

## Related Skills

- `@kubernetes-patterns` - Container orchestration for microservices
- `@docker-patterns` - Containerization
- `@api-rest-design` - API design for services
- `@observability-monitoring` - Distributed tracing and monitoring
- `@security-best-practices` - Service-to-service security
- `@performance-optimization` - Service performance
