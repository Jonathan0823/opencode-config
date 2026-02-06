# Database Optimization

## Query Optimization

### Indexing Strategies

```sql
-- Single column index
CREATE INDEX idx_users_email ON users(email);

-- Composite index (order matters!)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- Partial index
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- Expression index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- Covering index (PostgreSQL)
CREATE INDEX idx_orders_covering ON orders(user_id, status) 
INCLUDE (total_amount, created_at);

-- GIN index for JSON/array
CREATE INDEX idx_products_tags ON products USING GIN(tags);
```

### Query Rewriting

```sql
-- ❌ Full table scan with function
SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- ✅ Index-friendly range query
SELECT * FROM orders 
WHERE created_at >= '2024-01-01' 
AND created_at < '2025-01-01';

-- ❌ SELECT *
SELECT * FROM users WHERE id = 123;

-- ✅ Select only needed columns
SELECT id, name, email FROM users WHERE id = 123;

-- ❌ Not using index (wildcard at start)
SELECT * FROM products WHERE name LIKE '%phone%';

-- ✅ Using full-text search
SELECT * FROM products 
WHERE search_vector @@ plainto_tsquery('phone');
```

### N+1 Problem Solutions

```python
# ❌ N+1 Problem
users = db.query(User).all()
for user in users:
    orders = db.query(Order).filter_by(user_id=user.id).all()  # N queries
    print(f"{user.name}: {len(orders)} orders")

# ✅ Eager Loading (SQLAlchemy)
from sqlalchemy.orm import joinedload, selectinload

users = db.query(User).options(
    selectinload(User.orders)  # 2 queries total
).all()

# ✅ Bulk Loading
from sqlalchemy.orm import load_only

users = db.query(User).options(
    load_only('id', 'name'),
    selectinload(User.orders).load_only('id', 'total')
).all()

# Django ORM equivalent
from django.db.models import Prefetch

users = User.objects.prefetch_related(
    Prefetch('orders', queryset=Order.objects.filter(status='completed'))
)
```

## Connection Pooling

### PostgreSQL

```python
# SQLAlchemy with connection pooling
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    'postgresql://user:pass@localhost/db',
    poolclass=QueuePool,
    pool_size=10,              # Default connections
    max_overflow=20,           # Extra connections when needed
    pool_timeout=30,           # Wait time for available connection
    pool_recycle=3600,         # Recycle connections after 1 hour
    pool_pre_ping=True,        # Verify connection before using
)
```

### MySQL

```python
from sqlalchemy import create_engine

engine = create_engine(
    'mysql+pymysql://user:pass@localhost/db',
    pool_size=10,
    max_overflow=20,
    pool_recycle=3600,
    pool_pre_ping=True,
    connect_args={
        'connect_timeout': 10,
        'read_timeout': 30,
        'write_timeout': 30,
    }
)
```

### Node.js

```javascript
// pg-pool
const { Pool } = require('pg');

const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'myapp',
  user: 'user',
  password: 'pass',
  max: 20,                    // Maximum pool size
  idleTimeoutMillis: 30000,   // Close idle clients after 30s
  connectionTimeoutMillis: 2000,
  statement_timeout: 30000,   // Query timeout
});
```

## Partitioning

### PostgreSQL Table Partitioning

```sql
-- Create partitioned table
CREATE TABLE events (
    id BIGSERIAL,
    user_id INTEGER NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    data JSONB
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE events_2024_q1 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE events_2024_q2 PARTITION OF events
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Create indexes on each partition
CREATE INDEX idx_events_user_2024_q1 ON events_2024_q1(user_id);
CREATE INDEX idx_events_type_2024_q1 ON events_2024_q1(event_type);

-- Automated partition creation (using pg_partman)
SELECT partman.create_parent('public.events', 'created_at', 'native', 'monthly');
```

## Query Analysis

### PostgreSQL EXPLAIN

```sql
-- Basic explain
EXPLAIN ANALYZE
SELECT * FROM orders 
WHERE user_id = 123 
ORDER BY created_at DESC 
LIMIT 10;

-- Detailed analysis
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id, u.name
HAVING COUNT(o.id) > 5;
```

### Slow Query Log

```ini
# postgresql.conf
log_min_duration_statement = 1000  # Log queries > 1 second
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
temp_file_size = 0  # Log all temp file usage
```

## Database Monitoring

### pg_stat_statements

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top queries by total time
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

## Read Replicas

### Connection Routing

```python
class DatabaseRouter:
    def __init__(self, primary_url: str, replica_urls: list):
        self.primary = create_engine(primary_url)
        self.replicas = [create_engine(url) for url in replica_urls]
        self.replica_index = 0
    
    def get_read_engine(self):
        """Round-robin read replica selection."""
        engine = self.replicas[self.replica_index]
        self.replica_index = (self.replica_index + 1) % len(self.replicas)
        return engine
    
    def get_write_engine(self):
        """Always use primary for writes."""
        return self.primary

# Usage
router = DatabaseRouter(
    primary_url="postgresql://primary/db",
    replica_urls=[
        "postgresql://replica1/db",
        "postgresql://replica2/db",
    ]
)

# Read from replica
with router.get_read_engine().connect() as conn:
    result = conn.execute("SELECT * FROM users WHERE id = %s", (user_id,))

# Write to primary
with router.get_write_engine().connect() as conn:
    conn.execute("UPDATE users SET name = %s WHERE id = %s", (name, user_id))
```
