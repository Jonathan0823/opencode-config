# Event-Driven Architecture

## Event Sourcing

### Basic Concept

```python
from datetime import datetime
from typing import List, Dict
from uuid import uuid4
import json

class Event:
    def __init__(self, aggregate_id: str, event_type: str, data: dict):
        self.id = str(uuid4())
        self.aggregate_id = aggregate_id
        self.event_type = event_type
        self.data = data
        self.timestamp = datetime.utcnow()
        self.version = None
    
    def to_dict(self) -> dict:
        return {
            'id': self.id,
            'aggregate_id': self.aggregate_id,
            'event_type': self.event_type,
            'data': self.data,
            'timestamp': self.timestamp.isoformat(),
            'version': self.version
        }

class EventStore:
    """
    Stores events as the source of truth.
    """
    def __init__(self, db_connection):
        self.db = db_connection
    
    def append(self, aggregate_id: str, event: Event, expected_version: int) -> bool:
        """
        Append event to event store with optimistic concurrency control.
        """
        try:
            # Get current version
            result = self.db.execute(
                "SELECT MAX(version) FROM events WHERE aggregate_id = %s",
                (aggregate_id,)
            ).fetchone()
            
            current_version = result[0] or 0
            
            if current_version != expected_version:
                raise ConcurrencyException(
                    f"Expected version {expected_version}, found {current_version}"
                )
            
            event.version = current_version + 1
            
            # Store event
            self.db.execute(
                """
                INSERT INTO events (id, aggregate_id, event_type, data, timestamp, version)
                VALUES (%s, %s, %s, %s, %s, %s)
                """,
                (event.id, event.aggregate_id, event.event_type,
                 json.dumps(event.data), event.timestamp, event.version)
            )
            
            self.db.commit()
            return True
            
        except Exception as e:
            self.db.rollback()
            raise
    
    def get_events(self, aggregate_id: str, since_version: int = 0) -> List[Event]:
        """
        Get all events for an aggregate.
        """
        rows = self.db.execute(
            """
            SELECT * FROM events 
            WHERE aggregate_id = %s AND version > %s
            ORDER BY version ASC
            """,
            (aggregate_id, since_version)
        ).fetchall()
        
        events = []
        for row in rows:
            event = Event(
                aggregate_id=row['aggregate_id'],
                event_type=row['event_type'],
                data=json.loads(row['data'])
            )
            event.id = row['id']
            event.timestamp = row['timestamp']
            event.version = row['version']
            events.append(event)
        
        return events

class Aggregate:
    """
    Base class for event-sourced aggregates.
    """
    def __init__(self, aggregate_id: str):
        self.id = aggregate_id
        self.version = 0
        self.uncommitted_events: List[Event] = []
    
    def apply(self, event: Event):
        """Apply event to update aggregate state."""
        handler = getattr(self, f'_on_{event.event_type}', None)
        if handler:
            handler(event.data)
        self.version = event.version
    
    def apply_event(self, event_type: str, data: dict):
        """Create and apply new event."""
        event = Event(self.id, event_type, data)
        self.apply(event)
        self.uncommitted_events.append(event)
    
    def load_from_history(self, events: List[Event]):
        """Rehydrate aggregate from event history."""
        for event in events:
            self.apply(event)
        self.uncommitted_events = []

# Example: Order Aggregate
class Order(Aggregate):
    def __init__(self, order_id: str, user_id: str = None):
        super().__init__(order_id)
        self.user_id = user_id
        self.items = []
        self.status = 'pending'
        self.total = 0.0
    
    @staticmethod
    def create(order_id: str, user_id: str) -> 'Order':
        order = Order(order_id)
        order.apply_event('OrderCreated', {
            'order_id': order_id,
            'user_id': user_id,
            'created_at': datetime.utcnow().isoformat()
        })
        return order
    
    def add_item(self, product_id: str, quantity: int, price: float):
        self.apply_event('ItemAdded', {
            'product_id': product_id,
            'quantity': quantity,
            'price': price
        })
    
    def complete(self):
        if self.status != 'pending':
            raise ValueError("Order must be pending to complete")
        self.apply_event('OrderCompleted', {
            'completed_at': datetime.utcnow().isoformat()
        })
    
    # Event handlers
    def _on_OrderCreated(self, data):
        self.user_id = data['user_id']
    
    def _on_ItemAdded(self, data):
        self.items.append(data)
        self.total += data['quantity'] * data['price']
    
    def _on_OrderCompleted(self, data):
        self.status = 'completed'

# Repository
class OrderRepository:
    def __init__(self, event_store: EventStore):
        self.event_store = event_store
    
    def save(self, order: Order):
        """Save uncommitted events."""
        for event in order.uncommitted_events:
            self.event_store.append(order.id, event, order.version - 1)
        order.uncommitted_events = []
    
    def get_by_id(self, order_id: str) -> Order:
        """Rehydrate order from events."""
        events = self.event_store.get_events(order_id)
        if not events:
            raise NotFoundException(f"Order {order_id} not found")
        
        order = Order(order_id)
        order.load_from_history(events)
        return order
```

## CQRS (Command Query Responsibility Segregation)

### Architecture

```python
from abc import ABC, abstractmethod
from typing import TypeVar, Generic

T = TypeVar('T')

# Commands
class Command:
    pass

class CreateOrderCommand(Command):
    def __init__(self, order_id: str, user_id: str, items: list):
        self.order_id = order_id
        self.user_id = user_id
        self.items = items

class CommandHandler(ABC, Generic[T]):
    @abstractmethod
    def handle(self, command: T):
        pass

class CreateOrderHandler(CommandHandler[CreateOrderCommand]):
    def __init__(self, order_repository: OrderRepository, event_bus):
        self.order_repository = order_repository
        self.event_bus = event_bus
    
    def handle(self, command: CreateOrderCommand):
        # Create aggregate
        order = Order.create(command.order_id, command.user_id)
        
        # Add items
        for item in command.items:
            order.add_item(item['product_id'], item['quantity'], item['price'])
        
        # Save
        self.order_repository.save(order)
        
        # Publish events
        for event in order.uncommitted_events:
            self.event_bus.publish(event)

# Queries
class Query:
    pass

class GetOrderQuery(Query):
    def __init__(self, order_id: str):
        self.order_id = order_id

class OrderDTO:
    def __init__(self, order_id: str, user_id: str, status: str, total: float):
        self.order_id = order_id
        self.user_id = user_id
        self.status = status
        self.total = total

class QueryHandler(ABC, Generic[T]):
    @abstractmethod
    def handle(self, query: T):
        pass

class GetOrderHandler(QueryHandler[GetOrderQuery]):
    def __init__(self, read_db):
        self.read_db = read_db
    
    def handle(self, query: GetOrderQuery) -> OrderDTO:
        row = self.read_db.execute(
            "SELECT * FROM order_view WHERE order_id = %s",
            (query.order_id,)
        ).fetchone()
        
        if not row:
            raise NotFoundException()
        
        return OrderDTO(
            order_id=row['order_id'],
            user_id=row['user_id'],
            status=row['status'],
            total=row['total']
        )

# Projection (Event Handler)
class OrderProjection:
    def __init__(self, read_db):
        self.read_db = read_db
    
    def handle_OrderCreated(self, event):
        self.read_db.execute(
            """
            INSERT INTO order_view (order_id, user_id, status, total, created_at)
            VALUES (%s, %s, %s, %s, %s)
            """,
            (event.aggregate_id, event.data['user_id'], 'pending', 0, event.data['created_at'])
        )
        self.read_db.commit()
    
    def handle_ItemAdded(self, event):
        self.read_db.execute(
            """
            UPDATE order_view 
            SET total = total + %s
            WHERE order_id = %s
            """,
            (event.data['quantity'] * event.data['price'], event.aggregate_id)
        )
        self.read_db.commit()
    
    def handle_OrderCompleted(self, event):
        self.read_db.execute(
            """
            UPDATE order_view 
            SET status = 'completed', completed_at = %s
            WHERE order_id = %s
            """,
            (event.data['completed_at'], event.aggregate_id)
        )
        self.read_db.commit()
```

## Saga Pattern

### Orchestration-Based Saga

```python
class SagaOrchestrator:
    def __init__(self, event_bus, command_bus):
        self.event_bus = event_bus
        self.command_bus = command_bus
        self.compensation_handlers = {}
    
    def execute_order_saga(self, order_id: str, user_id: str, items: list):
        saga = OrderSaga(order_id, self.command_bus)
        
        try:
            # Step 1: Reserve inventory
            saga.reserve_inventory(items)
            
            # Step 2: Process payment
            total = sum(item['price'] * item['quantity'] for item in items)
            saga.process_payment(user_id, total)
            
            # Step 3: Ship order
            saga.ship_order(order_id)
            
            # Step 4: Complete order
            saga.complete_order(order_id)
            
        except SagaException as e:
            # Compensate (rollback)
            saga.compensate()
            raise

class OrderSaga:
    def __init__(self, order_id: str, command_bus):
        self.order_id = order_id
        self.command_bus = command_bus
        self.completed_steps = []
    
    def reserve_inventory(self, items):
        try:
            self.command_bus.send(ReserveInventoryCommand(items))
            self.completed_steps.append('inventory')
        except Exception as e:
            raise SagaException(f"Failed to reserve inventory: {e}")
    
    def process_payment(self, user_id: str, amount: float):
        try:
            self.command_bus.send(ProcessPaymentCommand(user_id, amount))
            self.completed_steps.append('payment')
        except Exception as e:
            raise SagaException(f"Failed to process payment: {e}")
    
    def ship_order(self, order_id: str):
        try:
            self.command_bus.send(ShipOrderCommand(order_id))
            self.completed_steps.append('shipping')
        except Exception as e:
            raise SagaException(f"Failed to ship order: {e}")
    
    def complete_order(self, order_id: str):
        try:
            self.command_bus.send(CompleteOrderCommand(order_id))
            self.completed_steps.append('complete')
        except Exception as e:
            raise SagaException(f"Failed to complete order: {e}")
    
    def compensate(self):
        """Rollback completed steps in reverse order."""
        for step in reversed(self.completed_steps):
            if step == 'complete':
                self.command_bus.send(RevertOrderCompletionCommand(self.order_id))
            elif step == 'shipping':
                self.command_bus.send(CancelShipmentCommand(self.order_id))
            elif step == 'payment':
                self.command_bus.send(RefundPaymentCommand(self.order_id))
            elif step == 'inventory':
                self.command_bus.send(ReleaseInventoryCommand(self.order_id))
```

## Outbox Pattern

```python
from sqlalchemy import create_engine, Column, String, JSON, DateTime
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from datetime import datetime
import json

Base = declarative_base()

class OutboxEvent(Base):
    __tablename__ = 'outbox'
    
    id = Column(String(36), primary_key=True)
    aggregate_type = Column(String(50))
    aggregate_id = Column(String(36))
    event_type = Column(String(50))
    payload = Column(JSON)
    created_at = Column(DateTime, default=datetime.utcnow)
    processed = Column(String(1), default='N')

class OutboxPublisher:
    """
    Ensures events are published reliably.
    """
    def __init__(self, session, message_broker):
        self.session = session
        self.message_broker = message_broker
    
    def publish(self, aggregate_type: str, aggregate_id: str, 
                event_type: str, payload: dict):
        """
        Store event in outbox table (same transaction as business logic).
        """
        outbox_event = OutboxEvent(
            id=str(uuid4()),
            aggregate_type=aggregate_type,
            aggregate_id=aggregate_id,
            event_type=event_type,
            payload=payload
        )
        self.session.add(outbox_event)
    
    def process_outbox(self):
        """
        Poll outbox table and publish events.
        Called by background worker.
        """
        events = self.session.query(OutboxEvent).filter_by(processed='N').all()
        
        for event in events:
            try:
                # Publish to message broker
                self.message_broker.publish(
                    exchange=event.aggregate_type,
                    routing_key=event.event_type,
                    body=json.dumps({
                        'aggregate_id': event.aggregate_id,
                        'event_type': event.event_type,
                        'payload': event.payload,
                        'timestamp': event.created_at.isoformat()
                    })
                )
                
                # Mark as processed
                event.processed = 'Y'
                self.session.commit()
                
            except Exception as e:
                self.session.rollback()
                # Log error, will retry on next poll
                print(f"Failed to publish event {event.id}: {e}")

# Usage in service
class OrderService:
    def __init__(self, session, outbox_publisher):
        self.session = session
        self.outbox_publisher = outbox_publisher
    
    def create_order(self, user_id: str, items: list):
        # Business logic
        order = Order(user_id=user_id)
        for item in items:
            order.add_item(item)
        
        self.session.add(order)
        
        # Publish event via outbox (same transaction)
        self.outbox_publisher.publish(
            aggregate_type='order',
            aggregate_id=order.id,
            event_type='OrderCreated',
            payload={
                'user_id': user_id,
                'items': items,
                'total': order.total
            }
        )
        
        self.session.commit()
```
