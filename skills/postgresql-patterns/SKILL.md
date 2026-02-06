---
name: postgresql-patterns
description: PostgreSQL patterns, optimization, and best practices for production
license: MIT
compatibility: opencode
---

# PostgreSQL Patterns Skill

## Overview

This skill provides guidelines for designing, querying, and optimizing PostgreSQL databases for production applications.

## Schema Design

### 1. Table Design

```sql
-- DO: Use appropriate data types
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    username VARCHAR(50) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(100),
    age INTEGER CHECK (age >= 0 AND age <= 150),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB -- Flexible schema for unstructured data
);

-- DO: Add constraints for data integrity
ALTER TABLE users ADD CONSTRAINT valid_email 
    CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

ALTER TABLE users ADD CONSTRAINT username_format 
    CHECK (username ~* '^[a-zA-Z0-9_]{3,50}$');
```

### 2. Relationships

```sql
-- One-to-Many
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    total_amount DECIMAL(10, 2) NOT NULL CHECK (total_amount >= 0),
    status VARCHAR(20) DEFAULT 'pending' 
        CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Many-to-Many with junction table
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10, 2) NOT NULL CHECK (price >= 0)
);

CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price_at_time DECIMAL(10, 2) NOT NULL,
    UNIQUE(order_id, product_id) -- Prevent duplicates
);

-- Self-referential (hierarchical)
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    parent_id UUID REFERENCES categories(id) ON DELETE CASCADE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### 3. Indexes

```sql
-- DO: Index frequently queried columns
CREATE INDEX idx_users_email ON users(email);

-- DO: Composite indexes for multi-column queries
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- DO: Partial indexes for filtered queries
CREATE INDEX idx_orders_pending ON orders(created_at) 
    WHERE status = 'pending';

-- DO: Expression indexes
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- DO: GIN index for JSONB/array
CREATE INDEX idx_users_metadata ON users USING GIN(metadata);

-- DO: Index for text search
CREATE INDEX idx_products_name_trgm ON products 
    USING gin(name gin_trgm_ops);

-- DO: BRIN index for large, naturally ordered tables
CREATE INDEX idx_events_created_brin ON events 
    USING BRIN(created_at) WITH (pages_per_range = 128);

-- DON'T: Over-index
-- Every index slows down writes
-- Monitor with: SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;
```

## Query Patterns

### 1. Select Queries

```sql
-- DO: Select only needed columns
SELECT id, email, full_name FROM users WHERE id = 'uuid';

-- DON'T: SELECT * in production
SELECT * FROM users; -- Avoid this

-- DO: Use LIMIT with large datasets
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20;

-- DO: Use OFFSET for pagination (consider keyset for large datasets)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 40;

-- DO: Use keyset pagination for better performance
SELECT * FROM orders 
WHERE created_at < '2024-01-01' 
ORDER BY created_at DESC 
LIMIT 20;
```

### 2. Joins

```sql
-- DO: Use explicit JOIN syntax
SELECT 
    u.id,
    u.email,
    COUNT(o.id) as order_count,
    COALESCE(SUM(o.total_amount), 0) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.status != 'cancelled'
WHERE u.is_active = true
GROUP BY u.id, u.email
HAVING COUNT(o.id) > 0
ORDER BY total_spent DESC;

-- DO: Use CTEs for complex queries
WITH user_stats AS (
    SELECT 
        user_id,
        COUNT(*) as order_count,
        SUM(total_amount) as total_revenue
    FROM orders
    WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY user_id
)
SELECT 
    u.email,
    us.order_count,
    us.total_revenue
FROM users u
JOIN user_stats us ON u.id = us.user_id
WHERE us.total_revenue > 1000;
```

### 3. Insert/Update/Delete

```sql
-- DO: Use INSERT with ON CONFLICT (upsert)
INSERT INTO users (email, username, password_hash)
VALUES ('test@example.com', 'testuser', 'hash')
ON CONFLICT (email) DO UPDATE SET
    password_hash = EXCLUDED.password_hash,
    updated_at = CURRENT_TIMESTAMP
RETURNING id;

-- DO: Batch inserts
INSERT INTO order_items (order_id, product_id, quantity, price_at_time)
VALUES 
    ('order-1', 'product-1', 2, 29.99),
    ('order-1', 'product-2', 1, 49.99),
    ('order-1', 'product-3', 3, 9.99);

-- DO: Use UPDATE with RETURNING
UPDATE users 
SET last_login = CURRENT_TIMESTAMP
WHERE id = 'user-id'
RETURNING *;

-- DO: Use DELETE with LIMIT and RETURNING
DELETE FROM temp_logs 
WHERE created_at < CURRENT_DATE - INTERVAL '7 days'
RETURNING id;
```

## Advanced Features

### 1. Full-Text Search

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Create search vector column
ALTER TABLE products ADD COLUMN search_vector tsvector;

-- Update search vector
UPDATE products SET search_vector = 
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'B');

-- Create index
CREATE INDEX idx_products_search ON products USING gin(search_vector);

-- Search query
SELECT id, name, ts_rank(search_vector, query) as rank
FROM products, plainto_tsquery('english', 'search term') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

### 2. JSONB Operations

```sql
-- DO: Use JSONB for flexible schemas
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type VARCHAR(50) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Index specific JSONB paths
CREATE INDEX idx_events_user_id ON events((payload->>'user_id'));

-- Query JSONB
SELECT * FROM events 
WHERE payload @> '{"event": "login"}';

-- Extract values
SELECT 
    event_type,
    payload->>'user_id' as user_id,
    payload->'details'->>'ip' as ip_address
FROM events
WHERE created_at > CURRENT_DATE - INTERVAL '1 day';

-- Update JSONB
UPDATE events 
SET payload = payload || '{"processed": true}'::jsonb
WHERE id = 'event-id';
```

### 3. Window Functions

```sql
-- Running totals
SELECT 
    user_id,
    order_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY user_id 
        ORDER BY order_date
    ) as running_total
FROM orders;

-- Ranking
SELECT 
    product_id,
    sales_amount,
    RANK() OVER (ORDER BY sales_amount DESC) as rank,
    DENSE_RANK() OVER (ORDER BY sales_amount DESC) as dense_rank
FROM product_sales;

-- Lag/Lead for comparisons
SELECT 
    user_id,
    login_date,
    LAG(login_date) OVER (PARTITION BY user_id ORDER BY login_date) as prev_login,
    login_date - LAG(login_date) OVER (PARTITION BY user_id ORDER BY login_date) as days_between
FROM user_logins;
```

## Optimization

### 1. Query Analysis

```sql
-- Analyze query execution
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- Check for seq scans
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders WHERE user_id = 'uuid';

-- Look for high-cost operations
-- - Seq Scan on large tables (should be index scan)
-- - Nested Loop with high row counts (consider hash join)
-- - Sort operations (consider index for ordering)
```

### 2. Performance Tips

```sql
-- DO: Use prepared statements (automatic in most ORMs)
PREPARE get_user (UUID) AS
    SELECT * FROM users WHERE id = $1;

EXECUTE get_user('uuid');

-- DO: Vacuum and analyze regularly
VACUUM ANALYZE users;

-- DO: Partition large tables
CREATE TABLE events_2024 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- DO: Use connection pooling (PgBouncer)
-- DO: Set appropriate work_mem for complex queries
SET work_mem = '256MB';
```

### 3. Monitoring Queries

```sql
-- Find slow queries
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

-- Check index usage
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

## Migration Patterns

### 1. Zero-Downtime Migrations

```sql
-- Phase 1: Add column (nullable)
ALTER TABLE users ADD COLUMN full_name VARCHAR(200);

-- Phase 2: Dual-write in application code
-- Write to both first_name/last_name AND full_name

-- Phase 3: Backfill data
UPDATE users 
SET full_name = CONCAT(first_name, ' ', last_name)
WHERE full_name IS NULL;

-- Phase 4: Make NOT NULL after backfill
ALTER TABLE users ALTER COLUMN full_name SET NOT NULL;

-- Phase 5: Update application to read from full_name only

-- Phase 6: Remove old columns
ALTER TABLE users DROP COLUMN first_name;
ALTER TABLE users DROP COLUMN last_name;
```

### 2. Transaction Safety

```sql
-- DO: Use transactions for related changes
BEGIN;

INSERT INTO orders (user_id, total_amount) 
VALUES ('user-uuid', 99.99)
RETURNING id;

INSERT INTO order_items (order_id, product_id, quantity)
VALUES ('order-uuid', 'product-uuid', 1);

UPDATE inventory 
SET quantity = quantity - 1 
WHERE product_id = 'product-uuid';

COMMIT;

-- DO: Use savepoints for partial rollback
BEGIN;

SAVEPOINT before_payment;

-- Try payment processing
-- If fails:
ROLLBACK TO SAVEPOINT before_payment;

-- Continue with other operations

COMMIT;
```

## When to Use

Use this skill when:
- Designing PostgreSQL schemas
- Writing complex SQL queries
- Optimizing query performance
- Implementing migrations
- Working with JSONB data
- Setting up full-text search
- Analyzing query performance
