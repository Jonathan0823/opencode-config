# Data Management in Microservices

## Database Per Service

### Pattern Overview

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Order Service │     │ Payment Service │     │ Inventory Svc   │
│  ┌───────────┐  │     │  ┌───────────┐  │     │  ┌───────────┐  │
│  │ Orders DB │  │     │  │ PaymentDB │  │     │  │InventoryDB│  │
│  └───────────┘  │     │  └───────────┘  │     │  └───────────┘  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

### Implementation

```yaml
# docker-compose.yml for local development
version: '3.8'
services:
  order-service:
    build: ./order-service
    environment:
      - DATABASE_URL=postgresql://orders_user:pass@orders-db/orders
  
  orders-db:
    image: postgres:15
    environment:
      - POSTGRES_USER=orders_user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=orders
    volumes:
      - orders_data:/var/lib/postgresql/data
  
  payment-service:
    build: ./payment-service
    environment:
      - DATABASE_URL=postgresql://payment_user:pass@payment-db/payments
  
  payment-db:
    image: postgres:15
    environment:
      - POSTGRES_USER=payment_user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=payments
    volumes:
      - payment_data:/var/lib/postgresql/data

volumes:
  orders_data:
  payment_data:
```

## Data Consistency Patterns

### SAGA Pattern (Orchestration)

```python
class OrderSaga:
    """
    Distributed transaction using Saga pattern.
    """
    def __init__(self, saga_id: str, event_store):
        self.saga_id = saga_id
        self.event_store = event_store
        self.steps = []
        self.current_step = 0
    
    def add_step(self, action: Callable, compensation: Callable):
        self.steps.append({'action': action, 'compensation': compensation})
    
    def execute(self):
        """Execute saga steps."""
        for i, step in enumerate(self.steps):
            try:
                step['action']()
                self.current_step = i + 1
                self._persist_state()
            except Exception as e:
                self._compensate()
                raise SagaFailedException(f"Saga failed at step {i}: {e}")
    
    def _compensate(self):
        """Run compensation for completed steps in reverse order."""
        for i in range(self.current_step - 1, -1, -1):
            try:
                self.steps[i]['compensation']()
            except Exception as e:
                # Log compensation failure for manual intervention
                print(f"Compensation failed for step {i}: {e}")
    
    def _persist_state(self):
        """Persist saga state for recovery."""
        self.event_store.append({
            'saga_id': self.saga_id,
            'current_step': self.current_step,
            'status': 'in_progress'
        })

# Usage
class OrderService:
    def create_order(self, order_data):
        saga = OrderSaga(str(uuid4()), self.event_store)
        
        # Step 1: Create order
        saga.add_step(
            action=lambda: self._create_order_local(order_data),
            compensation=lambda: self._cancel_order(order_data['order_id'])
        )
        
        # Step 2: Reserve inventory
        saga.add_step(
            action=lambda: self.inventory_client.reserve(order_data['items']),
            compensation=lambda: self.inventory_client.release(order_data['items'])
        )
        
        # Step 3: Process payment
        saga.add_step(
            action=lambda: self.payment_client.charge(order_data['payment']),
            compensation=lambda: self.payment_client.refund(order_data['payment'])
        )
        
        saga.execute()
```

### Eventual Consistency

```python
class EventualConsistencyManager:
    """
    Manages data synchronization between services.
    """
    def __init__(self, event_bus, retry_policy):
        self.event_bus = event_bus
        self.retry_policy = retry_policy
        self.pending_syncs = {}
    
    def sync_data(self, entity_type: str, entity_id: str, target_services: list):
        """
        Ensure data is synchronized to all target services.
        """
        sync_id = str(uuid4())
        self.pending_syncs[sync_id] = {
            'entity_type': entity_type,
            'entity_id': entity_id,
            'pending_services': set(target_services),
            'confirmed_services': set(),
            'attempts': 0
        }
        
        # Publish sync event
        self.event_bus.publish('DataSyncRequested', {
            'sync_id': sync_id,
            'entity_type': entity_type,
            'entity_id': entity_id,
            'target_services': target_services
        })
        
        return sync_id
    
    def confirm_sync(self, sync_id: str, service_name: str):
        """
        Called by services when they receive and process the data.
        """
        if sync_id in self.pending_syncs:
            sync = self.pending_syncs[sync_id]
            sync['pending_services'].discard(service_name)
            sync['confirmed_services'].add(service_name)
            
            if not sync['pending_services']:
                # All services synced
                del self.pending_syncs[sync_id]
                return True
        
        return False
    
    def check_sync_status(self, sync_id: str):
        """Check if all services have synced."""
        if sync_id not in self.pending_syncs:
            return {'status': 'completed'}
        
        sync = self.pending_syncs[sync_id]
        return {
            'status': 'in_progress',
            'pending': list(sync['pending_services']),
            'confirmed': list(sync['confirmed_services'])
        }
```

## API Composition

```python
class OrderDetailsComposer:
    """
    Compose order details from multiple services.
    """
    def __init__(self, order_service, user_service, inventory_service):
        self.order_service = order_service
        self.user_service = user_service
        self.inventory_service = inventory_service
    
    async def get_order_details(self, order_id: str) -> dict:
        """
        Fetch and compose order details from multiple services.
        """
        # Fetch order
        order = await self.order_service.get_order(order_id)
        
        # Fetch related data in parallel
        user_task = self.user_service.get_user(order['user_id'])
        items_tasks = [
            self.inventory_service.get_product(item['product_id'])
            for item in order['items']
        ]
        
        user, *products = await asyncio.gather(user_task, *items_tasks)
        
        # Compose response
        return {
            'order_id': order_id,
            'status': order['status'],
            'created_at': order['created_at'],
            'user': {
                'id': user['id'],
                'name': user['name'],
                'email': user['email']
            },
            'items': [
                {
                    'product_id': item['product_id'],
                    'name': products[i]['name'],
                    'quantity': item['quantity'],
                    'price': item['price']
                }
                for i, item in enumerate(order['items'])
            ],
            'total': order['total']
        }
```

## Materialized View Pattern

```python
class OrderMaterializedViewBuilder:
    """
    Build materialized views from events.
    """
    def __init__(self, read_db, event_store):
        self.read_db = read_db
        self.event_store = event_store
    
    def handle_order_created(self, event):
        """Update view when order is created."""
        self.read_db.execute(
            """
            INSERT INTO order_summary (order_id, user_id, status, total, created_at)
            VALUES (%s, %s, %s, %s, %s)
            """,
            (event['order_id'], event['user_id'], 'pending', 0, event['created_at'])
        )
        self.read_db.commit()
    
    def handle_item_added(self, event):
        """Update view when item is added."""
        self.read_db.execute(
            """
            UPDATE order_summary 
            SET total = total + %s
            WHERE order_id = %s
            """,
            (event['quantity'] * event['price'], event['order_id'])
        )
        self.read_db.commit()
    
    def handle_order_completed(self, event):
        """Update view when order is completed."""
        self.read_db.execute(
            """
            UPDATE order_summary 
            SET status = 'completed', completed_at = %s
            WHERE order_id = %s
            """,
            (event['completed_at'], event['order_id'])
        )
        self.read_db.commit()
    
    def rebuild_view(self):
        """Rebuild materialized view from scratch."""
        # Clear existing data
        self.read_db.execute("TRUNCATE TABLE order_summary")
        
        # Replay all events
        events = self.event_store.get_all_events('order')
        for event in events:
            handler = getattr(self, f"handle_{event['event_type']}", None)
            if handler:
                handler(event)
```

## Shared Database Anti-Pattern Prevention

```python
class DatabaseIsolationValidator:
    """
    Prevent accidental direct database access between services.
    """
    def __init__(self, service_name: str, allowed_databases: list):
        self.service_name = service_name
        self.allowed_databases = allowed_databases
    
    def validate_connection_string(self, connection_string: str):
        """
        Validate that service only connects to allowed databases.
        """
        parsed = urlparse(connection_string)
        database = parsed.path.strip('/')
        
        if database not in self.allowed_databases:
            raise SecurityException(
                f"Service {self.service_name} cannot access database {database}. "
                f"Allowed databases: {self.allowed_databases}"
            )
    
    def intercept_sql(self, sql: str):
        """
        Intercept and validate SQL to prevent cross-service queries.
        """
        # Check for cross-schema/table access
        forbidden_patterns = [
            r'FROM\s+\w+\.orders',      # Other service's orders
            r'FROM\s+\w+\.payments',    # Other service's payments
            r'JOIN\s+\w+\.\w+',         # Cross-service joins
        ]
        
        for pattern in forbidden_patterns:
            if re.search(pattern, sql, re.IGNORECASE):
                raise SecurityException(
                    f"Cross-service database access detected in query: {sql}"
                )

# Service configuration validation
def validate_service_config(service_name: str, config: dict):
    """
    Validate service configuration enforces database per service.
    """
    validator = DatabaseIsolationValidator(
        service_name=service_name,
        allowed_databases=[f"{service_name}_db"]
    )
    
    # Validate all connection strings
    for key, value in config.items():
        if 'DATABASE_URL' in key or 'DB_CONNECTION' in key:
            validator.validate_connection_string(value)
```
