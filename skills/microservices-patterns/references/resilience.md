# Resilience Patterns

## Circuit Breaker

### Implementation

```python
from enum import Enum
import time
from typing import Callable, Optional

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"         # Failing, rejecting requests
    HALF_OPEN = "half_open"  # Testing if service recovered

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 30,
        half_open_max_calls: int = 3
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max_calls = half_open_max_calls
        
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self.half_open_calls = 0
    
    def call(self, func: Callable, fallback: Optional[Callable] = None, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
                self.half_open_calls = 0
                self.success_count = 0
            else:
                if fallback:
                    return fallback(*args, **kwargs)
                raise CircuitBreakerOpenException("Circuit breaker is open")
        
        if self.state == CircuitState.HALF_OPEN:
            if self.half_open_calls >= self.half_open_max_calls:
                if fallback:
                    return fallback(*args, **kwargs)
                raise CircuitBreakerOpenException("Circuit breaker is open")
            self.half_open_calls += 1
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            if fallback:
                return fallback(*args, **kwargs)
            raise
    
    def _on_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.half_open_max_calls:
                self._reset()
        else:
            self.failure_count = 0
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.OPEN
        elif self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
    
    def _should_attempt_reset(self) -> bool:
        if not self.last_failure_time:
            return True
        return time.time() - self.last_failure_time >= self.recovery_timeout
    
    def _reset(self):
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.half_open_calls = 0
        self.last_failure_time = None

# Usage with decorator
from functools import wraps

def circuit_breaker(
    failure_threshold: int = 5,
    recovery_timeout: int = 30,
    fallback: Optional[Callable] = None
):
    breaker = CircuitBreaker(failure_threshold, recovery_timeout)
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            return breaker.call(func, fallback, *args, **kwargs)
        return wrapper
    return decorator

# Example usage
@circuit_breaker(failure_threshold=5, recovery_timeout=30)
def call_payment_service(data):
    response = requests.post('http://payment-service/charge', json=data)
    response.raise_for_status()
    return response.json()

def payment_fallback(data):
    return {'status': 'queued', 'message': 'Payment will be processed later'}

@circuit_breaker(fallback=payment_fallback)
def process_payment(data):
    return call_payment_service(data)
```

## Retry Pattern

### Implementation

```python
import time
import random
from functools import wraps
from typing import Callable, Tuple, Type

def retry(
    max_attempts: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    exceptions: Tuple[Type[Exception], ...] = (Exception,),
    on_retry: Callable = None
):
    """
    Retry decorator with exponential backoff.
    
    Args:
        max_attempts: Maximum number of attempts
        delay: Initial delay between retries
        backoff: Multiplier for delay after each retry
        exceptions: Tuple of exceptions to catch
        on_retry: Callback function called on each retry
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            attempt = 1
            current_delay = delay
            
            while attempt <= max_attempts:
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts:
                        raise
                    
                    if on_retry:
                        on_retry(attempt, e)
                    
                    # Add jitter to prevent thundering herd
                    jitter = random.uniform(0, current_delay * 0.1)
                    time.sleep(current_delay + jitter)
                    
                    current_delay *= backoff
                    attempt += 1
            
            return None  # Should not reach here
        
        return wrapper
    return decorator

# Usage
@retry(
    max_attempts=5,
    delay=1.0,
    backoff=2.0,
    exceptions=(requests.RequestException,),
    on_retry=lambda attempt, error: print(f"Retry {attempt}: {error}")
)
def fetch_data():
    response = requests.get('https://api.example.com/data', timeout=10)
    response.raise_for_status()
    return response.json()
```

## Timeout Pattern

```python
import signal
from contextlib import contextmanager
from functools import wraps

class TimeoutException(Exception):
    pass

@contextmanager
def timeout(seconds: int):
    """Context manager for timeout."""
    def timeout_handler(signum, frame):
        raise TimeoutException(f"Operation timed out after {seconds} seconds")
    
    # Set the alarm
    old_handler = signal.signal(signal.SIGALRM, timeout_handler)
    signal.alarm(seconds)
    
    try:
        yield
    finally:
        signal.alarm(0)
        signal.signal(signal.SIGALRM, old_handler)

# Usage with decorator
def with_timeout(seconds: int):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            with timeout(seconds):
                return func(*args, **kwargs)
        return wrapper
    return decorator

# Usage
@with_timeout(30)
def slow_operation():
    # Operation that might hang
    pass
```

## Bulkhead Pattern

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from threading import Semaphore
import queue

class Bulkhead:
    """
    Isolate failures in one part of the system from others.
    """
    def __init__(self, max_concurrent: int, max_queue: int):
        self.semaphore = Semaphore(max_concurrent)
        self.queue = queue.Queue(maxsize=max_queue)
        self.executor = ThreadPoolExecutor(max_workers=max_concurrent)
    
    def execute(self, func, *args, **kwargs):
        """Execute function with bulkhead protection."""
        if not self.semaphore.acquire(blocking=False):
            raise BulkheadFullException("Bulkhead capacity exceeded")
        
        try:
            future = self.executor.submit(func, *args, **kwargs)
            return future.result()
        finally:
            self.semaphore.release()

# Usage
class OrderService:
    def __init__(self):
        # Separate bulkheads for different operations
        self.payment_bulkhead = Bulkhead(max_concurrent=10, max_queue=20)
        self.inventory_bulkhead = Bulkhead(max_concurrent=20, max_queue=50)
    
    def process_order(self, order):
        try:
            # Process payment
            payment_result = self.payment_bulkhead.execute(
                self._process_payment,
                order
            )
            
            # Reserve inventory
            inventory_result = self.inventory_bulkhead.execute(
                self._reserve_inventory,
                order
            )
            
            return {'payment': payment_result, 'inventory': inventory_result}
        
        except BulkheadFullException:
            return {'error': 'Service temporarily unavailable'}
```

## Rate Limiting

```python
import time
from threading import Lock
from collections import deque

class TokenBucket:
    """
    Token bucket rate limiter.
    """
    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate
        self.last_refill = time.time()
        self.lock = Lock()
    
    def acquire(self, tokens: int = 1) -> bool:
        with self.lock:
            now = time.time()
            elapsed = now - self.last_refill
            
            # Refill tokens
            self.tokens = min(
                self.capacity,
                self.tokens + elapsed * self.refill_rate
            )
            self.last_refill = now
            
            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            return False

class SlidingWindow:
    """
    Sliding window rate limiter.
    """
    def __init__(self, window_size: int, max_requests: int):
        self.window_size = window_size
        self.max_requests = max_requests
        self.requests = deque()
        self.lock = Lock()
    
    def allow_request(self) -> bool:
        with self.lock:
            now = time.time()
            window_start = now - self.window_size
            
            # Remove old requests
            while self.requests and self.requests[0] < window_start:
                self.requests.popleft()
            
            if len(self.requests) < self.max_requests:
                self.requests.append(now)
                return True
            return False

# Distributed rate limiting with Redis
import redis

class RedisRateLimiter:
    def __init__(self, redis_client: redis.Redis, key_prefix: str = "rate_limit"):
        self.redis = redis_client
        self.key_prefix = key_prefix
    
    def is_allowed(self, key: str, max_requests: int, window: int) -> bool:
        """
        Check if request is allowed using sliding window.
        """
        pipe = self.redis.pipeline()
        now = time.time()
        window_start = now - window
        
        redis_key = f"{self.key_prefix}:{key}"
        
        # Remove old entries
        pipe.zremrangebyscore(redis_key, 0, window_start)
        
        # Count current requests
        pipe.zcard(redis_key)
        
        # Add current request
        pipe.zadd(redis_key, {str(now): now})
        
        # Set expiry
        pipe.expire(redis_key, window)
        
        results = pipe.execute()
        current_requests = results[1]
        
        return current_requests < max_requests
```
