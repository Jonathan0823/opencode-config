# Service Communication

## REST API

### Best Practices

```yaml
# OpenAPI Specification
openapi: 3.0.0
info:
  title: Order Service API
  version: 1.0.0
paths:
  /orders:
    post:
      summary: Create order
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Order created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '400':
          description: Invalid request
        '500':
          description: Server error

components:
  schemas:
    CreateOrderRequest:
      type: object
      required:
        - userId
        - items
      properties:
        userId:
          type: string
          format: uuid
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
    
    Order:
      type: object
      properties:
        id:
          type: string
          format: uuid
        userId:
          type: string
        status:
          type: string
          enum: [pending, processing, completed, cancelled]
        total:
          type: number
          format: decimal
```

### Client Implementation

```python
import requests
from tenacity import retry, stop_after_attempt, wait_exponential

class OrderServiceClient:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({
            'Content-Type': 'application/json',
            'Accept': 'application/json'
        })
    
    @retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
    def create_order(self, user_id: str, items: list) -> dict:
        response = self.session.post(
            f"{self.base_url}/orders",
            json={'userId': user_id, 'items': items},
            timeout=30
        )
        response.raise_for_status()
        return response.json()
    
    def get_order(self, order_id: str) -> dict:
        response = self.session.get(
            f"{self.base_url}/orders/{order_id}",
            timeout=10
        )
        response.raise_for_status()
        return response.json()
```

## gRPC

### Protocol Definition

```protobuf
// order.proto
syntax = "proto3";

package orders;

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (Order);
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc StreamOrderUpdates(StreamOrderRequest) returns (stream OrderUpdate);
}

message CreateOrderRequest {
  string user_id = 1;
  repeated OrderItem items = 2;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  double price = 3;
}

message Order {
  string id = 1;
  string user_id = 2;
  OrderStatus status = 3;
  repeated OrderItem items = 4;
  double total = 5;
}

enum OrderStatus {
  PENDING = 0;
  PROCESSING = 1;
  COMPLETED = 2;
  CANCELLED = 3;
}
```

### Server Implementation (Python)

```python
from concurrent import futures
import grpc
from order_pb2 import Order, CreateOrderRequest
from order_pb2_grpc import OrderServiceServicer, add_OrderServiceServicer_to_server

class OrderService(OrderServiceServicer):
    def CreateOrder(self, request, context):
        # Create order logic
        order = Order(
            id=str(uuid.uuid4()),
            user_id=request.user_id,
            status=Order.PENDING,
            items=request.items,
            total=sum(item.price * item.quantity for item in request.items)
        )
        return order
    
    def GetOrder(self, request, context):
        # Get order logic
        pass
    
    def StreamOrderUpdates(self, request, context):
        # Stream updates
        pass

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    add_OrderServiceServicer_to_server(OrderService(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()
```

## GraphQL

### Schema Definition

```graphql
# schema.graphql
type Order {
  id: ID!
  user: User!
  items: [OrderItem!]!
  status: OrderStatus!
  total: Float!
  createdAt: String!
}

type OrderItem {
  product: Product!
  quantity: Int!
  price: Float!
}

enum OrderStatus {
  PENDING
  PROCESSING
  COMPLETED
  CANCELLED
}

type Query {
  order(id: ID!): Order
  orders(userId: ID, status: OrderStatus): [Order!]!
}

type Mutation {
  createOrder(input: CreateOrderInput!): Order!
  updateOrderStatus(id: ID!, status: OrderStatus!): Order!
}

input CreateOrderInput {
  userId: ID!
  items: [OrderItemInput!]!
}

input OrderItemInput {
  productId: ID!
  quantity: Int!
}
```

### Resolver Implementation

```typescript
// resolvers.ts
export const resolvers = {
  Query: {
    order: async (_, { id }, { dataSources }) => {
      return dataSources.orderService.getOrder(id);
    },
    orders: async (_, { userId, status }, { dataSources }) => {
      return dataSources.orderService.getOrders({ userId, status });
    },
  },
  
  Order: {
    user: async (parent, _, { dataSources }) => {
      return dataSources.userService.getUser(parent.userId);
    },
    items: async (parent, _, { dataSources }) => {
      return Promise.all(
        parent.itemIds.map(id => dataSources.productService.getProduct(id))
      );
    },
  },
  
  Mutation: {
    createOrder: async (_, { input }, { dataSources }) => {
      return dataSources.orderService.createOrder(input);
    },
  },
};
```

## Message Queue (RabbitMQ)

### Producer

```python
import pika
import json

class EventPublisher:
    def __init__(self, amqp_url: str):
        self.connection = pika.BlockingConnection(pika.URLParameters(amqp_url))
        self.channel = self.connection.channel()
        
        # Declare exchange
        self.channel.exchange_declare(
            exchange='orders',
            exchange_type='topic',
            durable=True
        )
    
    def publish_order_created(self, order_id: str, user_id: str, total: float):
        message = {
            'event': 'order.created',
            'orderId': order_id,
            'userId': user_id,
            'total': total,
            'timestamp': datetime.utcnow().isoformat()
        }
        
        self.channel.basic_publish(
            exchange='orders',
            routing_key='order.created',
            body=json.dumps(message),
            properties=pika.BasicProperties(
                delivery_mode=2,  # Persistent
                content_type='application/json'
            )
        )
    
    def close(self):
        self.connection.close()
```

### Consumer

```python
import pika
import json

class OrderEventConsumer:
    def __init__(self, amqp_url: str):
        self.connection = pika.BlockingConnection(pika.URLParameters(amqp_url))
        self.channel = self.connection.channel()
        
        self.channel.exchange_declare(
            exchange='orders',
            exchange_type='topic',
            durable=True
        )
        
        # Declare queue
        result = self.channel.queue_declare(queue='email-service', durable=True)
        self.queue_name = result.method.queue
        
        # Bind queue
        self.channel.queue_bind(
            exchange='orders',
            queue=self.queue_name,
            routing_key='order.*'
        )
    
    def callback(self, ch, method, properties, body):
        event = json.loads(body)
        
        try:
            if event['event'] == 'order.created':
                self.handle_order_created(event)
            elif event['event'] == 'order.completed':
                self.handle_order_completed(event)
            
            ch.basic_ack(delivery_tag=method.delivery_tag)
        except Exception as e:
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
    
    def handle_order_created(self, event):
        # Send confirmation email
        print(f"Sending confirmation email for order {event['orderId']}")
    
    def start_consuming(self):
        self.channel.basic_qos(prefetch_count=10)
        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=self.callback
        )
        self.channel.start_consuming()
```

## Apache Kafka

### Producer

```python
from kafka import KafkaProducer
import json

class KafkaEventPublisher:
    def __init__(self, bootstrap_servers: list):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda v: v.encode('utf-8') if v else None,
            acks='all',
            retries=3,
            max_in_flight_requests_per_connection=5
        )
    
    def publish_order_event(self, order_id: str, event_type: str, data: dict):
        key = order_id.encode('utf-8')
        message = {
            'eventType': event_type,
            'orderId': order_id,
            'data': data,
            'timestamp': datetime.utcnow().isoformat()
        }
        
        future = self.producer.send(
            'orders',
            key=key,
            value=message
        )
        
        # Wait for confirmation
        record_metadata = future.get(timeout=10)
        return record_metadata
    
    def close(self):
        self.producer.flush()
        self.producer.close()
```

### Consumer with Consumer Group

```python
from kafka import KafkaConsumer
import json

class OrderEventConsumer:
    def __init__(self, bootstrap_servers: list, group_id: str):
        self.consumer = KafkaConsumer(
            'orders',
            bootstrap_servers=bootstrap_servers,
            group_id=group_id,
            auto_offset_reset='earliest',
            enable_auto_commit=False,
            max_poll_records=100,
            value_deserializer=lambda v: json.loads(v.decode('utf-8'))
        )
    
    def process_events(self):
        try:
            while True:
                messages = self.consumer.poll(timeout_ms=1000)
                
                for topic_partition, records in messages.items():
                    for record in records:
                        try:
                            self.handle_event(record.value)
                        except Exception as e:
                            print(f"Error processing record: {e}")
                            # Send to DLQ
                            continue
                
                # Commit offsets
                self.consumer.commit()
        
        finally:
            self.consumer.close()
    
    def handle_event(self, event):
        event_type = event['eventType']
        
        handlers = {
            'OrderCreated': self.handle_order_created,
            'OrderCompleted': self.handle_order_completed,
        }
        
        handler = handlers.get(event_type)
        if handler:
            handler(event['data'])
```
