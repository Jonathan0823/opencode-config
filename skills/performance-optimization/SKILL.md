---
name: performance-optimization
description: Performance optimization strategies including caching, profiling, database query optimization, frontend optimization, and load testing. Use when optimizing application performance, reducing latency, improving throughput, or diagnosing performance bottlenecks.
license: MIT
compatibility: opencode
---

# Performance Optimization Skill

## Overview

This skill provides comprehensive performance optimization strategies covering application profiling, caching strategies, database optimization, frontend performance, and load testing methodologies.

## Quick Start

### Performance Checklist

- [ ] Profile to identify bottlenecks before optimizing
- [ ] Optimize database queries and add indexes
- [ ] Implement caching at appropriate layers
- [ ] Use CDN for static assets
- [ ] Enable compression and minification
- [ ] Implement lazy loading
- [ ] Use connection pooling
- [ ] Load test before production

### Common Bottlenecks

1. **N+1 Query Problem** - Missing eager loading
2. **Missing Indexes** - Full table scans
3. **Synchronous I/O** - Blocking operations
4. **Large Payloads** - Unoptimized responses
5. **Memory Leaks** - Unreleased resources
6. **Cold Starts** - Unoptimized initialization

## Caching Strategies

### Application-Level Caching

```python
# Python with functools.lru_cache
from functools import lru_cache

@lru_cache(maxsize=128)
def get_user_by_id(user_id: int) -> dict:
    return db.query(User).get(user_id)

# Redis caching
import redis
from functools import wraps

r = redis.Redis(host='localhost', port=6379, db=0)

def cache_with_ttl(seconds=300):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            key = f"{func.__name__}:{args}:{kwargs}"
            cached = r.get(key)
            if cached:
                return json.loads(cached)
            
            result = func(*args, **kwargs)
            r.setex(key, seconds, json.dumps(result))
            return result
        return wrapper
    return decorator

@cache_with_ttl(seconds=600)
def get_expensive_data(param: str):
    # Expensive operation
    return result
```

### CDN Caching

```yaml
# CloudFront/CloudFlare headers
Cache-Control: public, max-age=31536000, immutable  # Static assets
Cache-Control: public, max-age=3600                 # API responses
Cache-Control: private, no-cache                    # User-specific data
```

### Database Query Caching

```python
# SQLAlchemy query caching
from sqlalchemy.orm import joinedload

# ❌ N+1 Problem
users = db.query(User).all()
for user in users:
    print(user.profile.bio)  # N additional queries

# ✅ Eager Loading
users = db.query(User).options(joinedload(User.profile)).all()

# ✅ Selective Loading
users = db.query(User).options(
    joinedload(User.profile),
    joinedload(User.posts)
).all()
```

## Database Optimization

### Query Optimization

```sql
-- ❌ Full table scan
SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- ✅ Index-friendly query
SELECT * FROM orders 
WHERE created_at >= '2024-01-01' 
AND created_at < '2025-01-01';

-- ✅ Covering index
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at) 
INCLUDE (total_amount, status);

-- ✅ Composite index for range queries
CREATE INDEX idx_products_category_price ON products(category, price);
```

### Connection Pooling

```python
# SQLAlchemy connection pooling
from sqlalchemy import create_engine

engine = create_engine(
    'postgresql://user:pass@localhost/db',
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,
    pool_recycle=3600
)

# Node.js with pg-pool
const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'myapp',
  max: 20,        // Maximum pool size
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});
```

## Frontend Optimization

### Code Splitting

```javascript
// React lazy loading
const Dashboard = lazy(() => import('./Dashboard'));
const Analytics = lazy(() => import('./Analytics'));

// Route-based splitting
<Route path="/dashboard" component={Dashboard} />
<Route path="/analytics" component={Analytics} />
```

### Image Optimization

```html
<!-- Responsive images -->
<picture>
  <source srcset="image.webp" type="image/webp">
  <source srcset="image.jpg" type="image/jpeg">
  <img src="image.jpg" alt="Description" loading="lazy">
</picture>

<!-- Next.js Image component -->
import Image from 'next/image';

<Image
  src="/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  placeholder="blur"
  priority={false}
/>
```

## Profiling

### Python Profiling

```python
# cProfile
import cProfile
import pstats

profiler = cProfile.Profile()
profiler.enable()

# Your code here
result = expensive_function()

profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)

# Line profiler
from line_profiler import LineProfiler

profiler = LineProfiler()

@profiler  # Add decorator
def my_function():
    for i in range(1000):
        x = [i ** 2 for i in range(1000)]
    return x

my_function()
profiler.print_stats()
```

### Database Query Analysis

```sql
-- PostgreSQL EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending';

-- MySQL EXPLAIN
EXPLAIN ANALYZE
SELECT * FROM orders 
WHERE user_id = 123 
ORDER BY created_at DESC 
LIMIT 10;
```

## Detailed References

See comprehensive guides in references/:

- **[Caching Strategies](references/caching.md)** - Redis, CDN, application caching, cache invalidation
- **[Database Optimization](references/database-optimization.md)** - Query optimization, indexing, connection pooling
- **[Frontend Performance](references/frontend-performance.md)** - Code splitting, lazy loading, image optimization, Core Web Vitals
- **[Load Testing](references/load-testing.md)** - k6, Artillery, JMeter, performance testing strategies

## When to Use This Skill

Use this skill when:
- Application is slow or unresponsive
- Database queries are taking too long
- API response times need improvement
- Implementing caching strategies
- Optimizing frontend performance
- Preparing for high traffic events
- Conducting performance audits
- Setting up monitoring and alerting

## Related Skills

- `@kubernetes-patterns` - Scaling and resource optimization
- `@docker-patterns` - Container optimization
- `@postgresql-patterns` - Database-specific optimization
- `@mongodb-patterns` - NoSQL optimization
- `@observability-monitoring` - Performance monitoring
