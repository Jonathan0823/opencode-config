# Caching Strategies

## Cache Layers

### 1. Browser Cache

```http
Cache-Control: public, max-age=31536000, immutable
ETag: "abc123"
Last-Modified: Wed, 21 Oct 2023 07:28:00 GMT
Vary: Accept-Encoding
```

### 2. CDN Cache

```yaml
# CloudFront settings
DefaultCacheBehavior:
  ViewerProtocolPolicy: redirect-to-https
  CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6  # Managed-CachingOptimized
  Compress: true
  
# Custom cache behavior
CacheBehaviors:
  - PathPattern: /api/*
    CachePolicyId: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad  # Managed-CachingDisabled
```

### 3. Application Cache

```python
# Redis caching implementation
import redis
import json
import hashlib
from functools import wraps
from typing import Optional

class CacheManager:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
    
    def generate_key(self, prefix: str, *args, **kwargs) -> str:
        """Generate cache key from function arguments."""
        key_data = f"{prefix}:{args}:{kwargs}"
        return hashlib.md5(key_data.encode()).hexdigest()
    
    def get(self, key: str) -> Optional[dict]:
        """Get value from cache."""
        data = self.redis.get(key)
        return json.loads(data) if data else None
    
    def set(self, key: str, value: dict, ttl: int = 300):
        """Set value in cache with TTL."""
        self.redis.setex(key, ttl, json.dumps(value))
    
    def delete(self, key: str):
        """Delete key from cache."""
        self.redis.delete(key)
    
    def invalidate_pattern(self, pattern: str):
        """Invalidate all keys matching pattern."""
        for key in self.redis.scan_iter(match=pattern):
            self.redis.delete(key)

def cached(prefix: str, ttl: int = 300):
    """Decorator for caching function results."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            cache = CacheManager(redis.Redis())
            key = cache.generate_key(prefix, *args, **kwargs)
            
            # Try to get from cache
            cached_value = cache.get(key)
            if cached_value is not None:
                return cached_value
            
            # Execute function and cache result
            result = func(*args, **kwargs)
            cache.set(key, result, ttl)
            return result
        return wrapper
    return decorator

# Usage
@cached(prefix="user", ttl=600)
def get_user(user_id: int) -> dict:
    return db.query(User).get(user_id).to_dict()

@cached(prefix="products", ttl=300)
def get_products(category: str, page: int = 1) -> list:
    return db.query(Product).filter_by(category=category).paginate(page, 20).items
```

### 4. Database Query Cache

```python
# SQLAlchemy query caching with dogpile.cache
from dogpile.cache import make_region

region = make_region().configure(
    'dogpile.cache.redis',
    arguments={
        'host': 'localhost',
        'port': 6379,
        'db': 0,
        'redis_expiration_time': 3600,
    }
)

@region.cache_on_arguments()
def get_user_orders(user_id: int) -> list:
    return db.query(Order).filter_by(user_id=user_id).all()
```

## Cache Invalidation Strategies

### 1. Time-Based (TTL)

```python
# Automatic expiration
redis.setex("key", 300, "value")  # Expires in 5 minutes

# Dynamic TTL based on data freshness
def calculate_ttl(data_type: str) -> int:
    ttls = {
        "user_profile": 3600,      # 1 hour
        "product_catalog": 86400,  # 1 day
        "realtime_data": 60,      # 1 minute
    }
    return ttls.get(data_type, 300)
```

### 2. Event-Based

```python
class CacheInvalidator:
    def __init__(self, cache: CacheManager):
        self.cache = cache
    
    def invalidate_user_cache(self, user_id: int):
        """Invalidate all caches related to user."""
        patterns = [
            f"user:{user_id}:*",
            f"user_orders:{user_id}:*",
            f"user_profile:{user_id}",
        ]
        for pattern in patterns:
            self.cache.invalidate_pattern(pattern)
    
    def invalidate_product_cache(self, product_id: int):
        """Invalidate product-related caches."""
        self.cache.delete(f"product:{product_id}")
        self.cache.invalidate_pattern("products:*")
        self.cache.invalidate_pattern("product_search:*")

# Usage in service layer
def update_user(user_id: int, data: dict):
    user = db.query(User).get(user_id)
    user.update(data)
    db.commit()
    
    # Invalidate caches
    invalidator = CacheInvalidator(cache)
    invalidator.invalidate_user_cache(user_id)
```

### 3. Version-Based (Cache Busting)

```python
class VersionedCache:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.version_key = "cache:version"
    
    def get_version(self) -> int:
        version = self.redis.get(self.version_key)
        return int(version) if version else 1
    
    def bump_version(self):
        """Increment version to invalidate all versioned caches."""
        self.redis.incr(self.version_key)
    
    def make_key(self, base_key: str) -> str:
        """Create versioned cache key."""
        version = self.get_version()
        return f"{base_key}:v{version}"

# Usage
cache = VersionedCache(redis.Redis())

# Store
key = cache.make_key("config")
redis.set(key, config_data)

# Retrieve
key = cache.make_key("config")
config = redis.get(key)

# Invalidate all versioned caches
cache.bump_version()
```

## Cache Patterns

### 1. Cache-Aside (Lazy Loading)

```python
def get_data(cache_key: str, db_query_func) -> dict:
    # Try cache first
    data = cache.get(cache_key)
    if data:
        return data
    
    # Cache miss - query database
    data = db_query_func()
    
    # Store in cache
    cache.set(cache_key, data, ttl=300)
    return data
```

### 2. Write-Through

```python
class WriteThroughCache:
    def __init__(self, cache: CacheManager, db_session):
        self.cache = cache
        self.db = db_session
    
    def set(self, key: str, value: dict):
        # Write to both cache and DB
        self.cache.set(key, value)
        self.db.execute("UPDATE table SET data = %s WHERE key = %s", (value, key))
        self.db.commit()
```

### 3. Write-Behind (Write-Back)

```python
class WriteBehindCache:
    def __init__(self, cache: CacheManager):
        self.cache = cache
        self.write_queue = []
    
    def set(self, key: str, value: dict):
        # Write to cache immediately
        self.cache.set(key, value)
        
        # Queue for async DB write
        self.write_queue.append((key, value))
        
        # Flush queue periodically
        if len(self.write_queue) >= 100:
            self.flush_writes()
    
    def flush_writes(self):
        # Batch write to database
        for key, value in self.write_queue:
            # DB write logic
            pass
        self.write_queue.clear()
```

### 4. Read-Through

```python
class ReadThroughCache:
    def __init__(self, cache: CacheManager, data_source):
        self.cache = cache
        self.data_source = data_source
    
    def get(self, key: str) -> dict:
        data = self.cache.get(key)
        if data is None:
            # Cache miss - load from source
            data = self.data_source.load(key)
            self.cache.set(key, data)
        return data
```

## Redis Best Practices

### Connection Pooling

```python
# Single connection pool
import redis

pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    db=0,
    max_connections=50,
    socket_connect_timeout=5,
    socket_timeout=5,
    health_check_interval=30
)

redis_client = redis.Redis(connection_pool=pool)
```

### Pipelining

```python
# Batch operations
pipe = redis_client.pipeline()

for user_id in user_ids:
    pipe.get(f"user:{user_id}")

results = pipe.execute()
```

### Distributed Locking

```python
import redis
import uuid
import time

class DistributedLock:
    def __init__(self, redis_client: redis.Redis, lock_key: str, timeout: int = 10):
        self.redis = redis_client
        self.lock_key = f"lock:{lock_key}"
        self.timeout = timeout
        self.identifier = str(uuid.uuid4())
    
    def acquire(self) -> bool:
        """Try to acquire lock."""
        return self.redis.set(
            self.lock_key,
            self.identifier,
            nx=True,  # Only set if not exists
            ex=self.timeout
        )
    
    def release(self):
        """Release lock if we own it."""
        # Lua script for atomic check-and-delete
        script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        self.redis.eval(script, 1, self.lock_key, self.identifier)
    
    def __enter__(self):
        while not self.acquire():
            time.sleep(0.1)
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.release()

# Usage
with DistributedLock(redis_client, "order_processing"):
    process_order(order_id)
```

## Cache Warming

```python
async def warm_cache():
    """Pre-populate cache with frequently accessed data."""
    # Warm user profiles
    users = await db.fetch("SELECT id FROM users WHERE active = true LIMIT 1000")
    for user in users:
        await cache_user_profile(user['id'])
    
    # Warm product catalog
    products = await db.fetch("SELECT id FROM products WHERE in_stock = true")
    for product in products:
        await cache_product(product['id'])
    
    # Warm configuration
    configs = await db.fetch("SELECT * FROM config")
    for config in configs:
        await cache.set(f"config:{config['key']}", config['value'])

# Schedule cache warming
# - On application startup
# - Periodically via cron job
# - After deployments
```
