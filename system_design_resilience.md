# Rate Limiting & Fault Tolerance Guide

## Table of Contents

1. [Rate Limiting Algorithms](#rate-limiting-algorithms)
2. [Circuit Breaker Pattern](#circuit-breaker-pattern)
3. [Retry Strategies](#retry-strategies)
4. [Bulkhead Pattern](#bulkhead-pattern)
5. [Backpressure](#backpressure)
6. [Timeout Patterns](#timeout-patterns)
7. [Graceful Degradation](#graceful-degradation)
8. [Chaos Engineering](#chaos-engineering)

---

## Rate Limiting Algorithms

### 1. Token Bucket

**Most popular: allows bursts**

```python
import time
import threading

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        """
        capacity: Maximum number of tokens
        refill_rate: Tokens added per second
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = capacity
        self.last_refill = time.time()
        self.lock = threading.Lock()
    
    def _refill(self):
        """Refill tokens based on time elapsed"""
        now = time.time()
        elapsed = now - self.last_refill
        tokens_to_add = elapsed * self.refill_rate
        
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now
    
    def allow_request(self, tokens=1):
        """Check if request is allowed"""
        with self.lock:
            self._refill()
            
            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            return False

# Usage
rate_limiter = TokenBucket(capacity=100, refill_rate=10)  # 10 requests/sec

for i in range(120):
    if rate_limiter.allow_request():
        print(f"Request {i} allowed")
    else:
        print(f"Request {i} rate limited")
    time.sleep(0.01)
```

### 2. Leaky Bucket

**Smooths traffic: constant output rate**

```python
from collections import deque
import time

class LeakyBucket:
    def __init__(self, capacity, leak_rate):
        """
        capacity: Maximum queue size
        leak_rate: Requests processed per second
        """
        self.capacity = capacity
        self.leak_rate = leak_rate
        self.queue = deque()
        self.last_leak = time.time()
    
    def _leak(self):
        """Process requests at constant rate"""
        now = time.time()
        elapsed = now - self.last_leak
        
        leaks = int(elapsed * self.leak_rate)
        for _ in range(min(leaks, len(self.queue))):
            self.queue.popleft()
        
        if leaks > 0:
            self.last_leak = now
    
    def allow_request(self):
        """Add request to queue if space available"""
        self._leak()
        
        if len(self.queue) < self.capacity:
            self.queue.append(time.time())
            return True
        return False

# Usage
rate_limiter = LeakyBucket(capacity=50, leak_rate=5)

for i in range(60):
    if rate_limiter.allow_request():
        print(f"Request {i} queued")
    else:
        print(f"Request {i} rejected - bucket full")
    time.sleep(0.05)
```

### 3. Fixed Window Counter

**Simple but allows bursts at window boundaries**

```python
import time
from collections import defaultdict

class FixedWindowCounter:
    def __init__(self, max_requests, window_seconds):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.windows = defaultdict(int)
    
    def allow_request(self, user_id):
        """Check if user is within rate limit"""
        current_window = int(time.time() / self.window_seconds)
        key = f"{user_id}:{current_window}"
        
        if self.windows[key] < self.max_requests:
            self.windows[key] += 1
            return True
        return False
    
    def cleanup_old_windows(self):
        """Remove expired windows"""
        current_window = int(time.time() / self.window_seconds)
        expired_keys = [k for k in self.windows if int(k.split(':')[1]) < current_window - 1]
        
        for key in expired_keys:
            del self.windows[key]

# Usage
rate_limiter = FixedWindowCounter(max_requests=100, window_seconds=60)

# Allow 100 requests per minute per user
if rate_limiter.allow_request(user_id='user123'):
    print("Request allowed")
else:
    print("Rate limit exceeded")
```

### 4. Sliding Window Log

**Most accurate: no burst issues**

```python
import time
from collections import defaultdict, deque

class SlidingWindowLog:
    def __init__(self, max_requests, window_seconds):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.request_logs = defaultdict(deque)
    
    def allow_request(self, user_id):
        """Check if request is allowed using sliding window"""
        now = time.time()
        window_start = now - self.window_seconds
        
        # Remove old requests outside window
        user_log = self.request_logs[user_id]
        while user_log and user_log[0] <= window_start:
            user_log.popleft()
        
        # Check if under limit
        if len(user_log) < self.max_requests:
            user_log.append(now)
            return True
        return False

# Usage
rate_limiter = SlidingWindowLog(max_requests=100, window_seconds=60)

for i in range(120):
    if rate_limiter.allow_request('user123'):
        print(f"Request {i} allowed")
    else:
        print(f"Request {i} rate limited")
    time.sleep(0.1)
```

### 5. Distributed Rate Limiting (Redis)

```python
import redis
import time

class DistributedRateLimiter:
    def __init__(self, redis_client, max_requests, window_seconds):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_seconds = window_seconds
    
    def allow_request(self, user_id):
        """Rate limit using Redis"""
        key = f"rate_limit:{user_id}"
        now = time.time()
        window_start = now - self.window_seconds
        
        # Use Redis sorted set
        pipe = self.redis.pipeline()
        
        # Remove old entries
        pipe.zremrangebyscore(key, 0, window_start)
        
        # Count requests in current window
        pipe.zcard(key)
        
        # Add current request
        pipe.zadd(key, {str(now): now})
        
        # Set expiry
        pipe.expire(key, self.window_seconds)
        
        results = pipe.execute()
        request_count = results[1]
        
        return request_count < self.max_requests

# Usage
redis_client = redis.Redis(host='localhost', port=6379)
rate_limiter = DistributedRateLimiter(redis_client, max_requests=100, window_seconds=60)

if rate_limiter.allow_request('user123'):
    print("Request allowed")
else:
    print("Rate limited")
```

---

## Circuit Breaker Pattern

**Prevents cascading failures**

```python
from enum import Enum
import time
import threading

class CircuitState(Enum):
    CLOSED = "CLOSED"      # Normal operation
    OPEN = "OPEN"          # Failing - reject requests
    HALF_OPEN = "HALF_OPEN"  # Testing if service recovered

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60, recovery_timeout=30):
        """
        failure_threshold: Failures before opening circuit
        timeout: Time to wait before trying half-open
        recovery_timeout: Time to wait in half-open before closing
        """
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.recovery_timeout = recovery_timeout
        
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = None
        self.success_count = 0
        self.lock = threading.Lock()
    
    def call(self, func, *args, **kwargs):
        """Execute function with circuit breaker"""
        with self.lock:
            if self.state == CircuitState.OPEN:
                # Check if timeout elapsed
                if time.time() - self.last_failure_time >= self.timeout:
                    self.state = CircuitState.HALF_OPEN
                    self.success_count = 0
                    print("Circuit HALF_OPEN - testing service")
                else:
                    raise CircuitBreakerOpenError("Circuit breaker is OPEN")
        
        try:
            # Execute function
            result = func(*args, **kwargs)
            
            with self.lock:
                if self.state == CircuitState.HALF_OPEN:
                    self.success_count += 1
                    
                    # If enough successes, close circuit
                    if self.success_count >= 3:
                        self.state = CircuitState.CLOSED
                        self.failure_count = 0
                        print("Circuit CLOSED - service recovered")
                elif self.state == CircuitState.CLOSED:
                    self.failure_count = 0  # Reset on success
            
            return result
            
        except Exception as e:
            with self.lock:
                self.failure_count += 1
                self.last_failure_time = time.time()
                
                if self.failure_count >= self.failure_threshold:
                    self.state = CircuitState.OPEN
                    print(f"Circuit OPEN - {self.failure_count} failures")
                
                if self.state == CircuitState.HALF_OPEN:
                    self.state = CircuitState.OPEN
                    print("Circuit OPEN again - service still failing")
            
            raise

class CircuitBreakerOpenError(Exception):
    pass

# Usage
def unreliable_api_call():
    """Simulates API that fails sometimes"""
    import random
    if random.random() < 0.7:  # 70% failure rate
        raise Exception("Service unavailable")
    return "Success"

circuit_breaker = CircuitBreaker(failure_threshold=3, timeout=5)

for i in range(20):
    try:
        result = circuit_breaker.call(unreliable_api_call)
        print(f"Request {i}: {result}")
    except CircuitBreakerOpenError:
        print(f"Request {i}: Circuit breaker is OPEN")
    except Exception as e:
        print(f"Request {i}: Failed - {e}")
    
    time.sleep(1)
```

---

## Retry Strategies

### 1. Exponential Backoff

```python
import time
import random

class ExponentialBackoff:
    def __init__(self, max_retries=5, base_delay=1, max_delay=60):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay
    
    def execute(self, func, *args, **kwargs):
        """Execute with exponential backoff"""
        for attempt in range(self.max_retries):
            try:
                return func(*args, **kwargs)
            except Exception as e:
                if attempt == self.max_retries - 1:
                    raise
                
                # Calculate delay with exponential backoff
                delay = min(self.base_delay * (2 ** attempt), self.max_delay)
                
                # Add jitter to prevent thundering herd
                jitter = random.uniform(0, delay * 0.1)
                total_delay = delay + jitter
                
                print(f"Attempt {attempt + 1} failed: {e}")
                print(f"Retrying in {total_delay:.2f} seconds...")
                time.sleep(total_delay)

# Usage
retry = ExponentialBackoff(max_retries=5)

def unstable_operation():
    import random
    if random.random() < 0.7:
        raise Exception("Temporary failure")
    return "Success"

result = retry.execute(unstable_operation)
print(f"Final result: {result}")
```

### 2. Retry with Circuit Breaker

```python
class RetryWithCircuitBreaker:
    def __init__(self, max_retries=3):
        self.max_retries = max_retries
        self.circuit_breaker = CircuitBreaker(failure_threshold=5, timeout=30)
    
    def execute(self, func, *args, **kwargs):
        """Retry with circuit breaker protection"""
        for attempt in range(self.max_retries):
            try:
                return self.circuit_breaker.call(func, *args, **kwargs)
            except CircuitBreakerOpenError:
                print("Circuit breaker is open - not retrying")
                raise
            except Exception as e:
                if attempt == self.max_retries - 1:
                    raise
                
                delay = 2 ** attempt
                print(f"Retry {attempt + 1}/{self.max_retries} after {delay}s")
                time.sleep(delay)

# Usage
retry_cb = RetryWithCircuitBreaker(max_retries=3)

try:
    result = retry_cb.execute(unreliable_api_call)
    print(f"Success: {result}")
except Exception as e:
    print(f"All retries failed: {e}")
```

---

## Bulkhead Pattern

**Isolate resources to prevent total system failure**

```python
from concurrent.futures import ThreadPoolExecutor
import time

class Bulkhead:
    def __init__(self, max_concurrent=10):
        """Limit concurrent executions"""
        self.executor = ThreadPoolExecutor(max_workers=max_concurrent)
    
    def execute(self, func, *args, **kwargs):
        """Execute with concurrency limit"""
        future = self.executor.submit(func, *args, **kwargs)
        return future

# Multiple bulkheads for different services
class ServiceBulkheads:
    def __init__(self):
        self.payment_bulkhead = Bulkhead(max_concurrent=5)
        self.inventory_bulkhead = Bulkhead(max_concurrent=10)
        self.notification_bulkhead = Bulkhead(max_concurrent=20)
    
    def process_payment(self, order):
        """Payment service with limited concurrency"""
        return self.payment_bulkhead.execute(self._payment_operation, order)
    
    def check_inventory(self, items):
        """Inventory service with its own limit"""
        return self.inventory_bulkhead.execute(self._inventory_operation, items)
    
    def send_notification(self, user_id, message):
        """Notification service - can handle more load"""
        return self.notification_bulkhead.execute(self._notification_operation, user_id, message)
    
    def _payment_operation(self, order):
        time.sleep(0.5)  # Slow operation
        return f"Payment processed for {order}"
    
    def _inventory_operation(self, items):
        time.sleep(0.1)
        return f"Inventory checked for {items}"
    
    def _notification_operation(self, user_id, message):
        time.sleep(0.05)
        return f"Notification sent to {user_id}"

# Usage
bulkheads = ServiceBulkheads()

# Payment service won't overwhelm the system
for i in range(100):
    future = bulkheads.process_payment(f"order-{i}")
    # Other services continue working even if payment is slow
```

---

## Backpressure

**Prevent system overload**

```python
from queue import Queue, Full
import threading
import time

class BackpressureQueue:
    def __init__(self, max_size=100, timeout=5):
        self.queue = Queue(maxsize=max_size)
        self.timeout = timeout
        self.dropped_requests = 0
    
    def push(self, item):
        """Add item with backpressure"""
        try:
            self.queue.put(item, timeout=self.timeout)
            return True
        except Full:
            self.dropped_requests += 1
            print(f"Queue full - dropping request (total dropped: {self.dropped_requests})")
            return False
    
    def pop(self):
        """Get item from queue"""
        try:
            return self.queue.get(timeout=1)
        except:
            return None
    
    def size(self):
        return self.queue.qsize()

# Producer-Consumer with backpressure
class BackpressureSystem:
    def __init__(self):
        self.queue = BackpressureQueue(max_size=50)
        self.running = True
    
    def producer(self, rate=100):
        """Produces items at given rate"""
        while self.running:
            if not self.queue.push(f"item-{time.time()}"):
                # Slow down if queue is full
                time.sleep(0.5)
            else:
                time.sleep(1 / rate)
    
    def consumer(self):
        """Consumes items (slower than producer)"""
        while self.running:
            item = self.queue.pop()
            if item:
                # Slow processing
                time.sleep(0.1)
                print(f"Processed: {item}")
    
    def start(self):
        """Start system"""
        producer_thread = threading.Thread(target=self.producer, args=(20,))
        consumer_thread = threading.Thread(target=self.consumer)
        
        producer_thread.start()
        consumer_thread.start()
        
        return producer_thread, consumer_thread

# Usage
system = BackpressureSystem()
threads = system.start()
# System will apply backpressure when queue fills up
```

---

## Timeout Patterns

```python
import signal
from functools import wraps

class TimeoutError(Exception):
    pass

def timeout(seconds):
    """Decorator to add timeout to function"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            def timeout_handler(signum, frame):
                raise TimeoutError(f"Function {func.__name__} timed out after {seconds}s")
            
            # Set alarm
            signal.signal(signal.SIGALRM, timeout_handler)
            signal.alarm(seconds)
            
            try:
                result = func(*args, **kwargs)
            finally:
                signal.alarm(0)  # Cancel alarm
            
            return result
        return wrapper
    return decorator

@timeout(5)
def slow_operation():
    """Operation that might take too long"""
    time.sleep(10)
    return "Done"

try:
    result = slow_operation()
except TimeoutError as e:
    print(f"Operation timed out: {e}")
```

---

## Graceful Degradation

**Provide reduced functionality when systems are overloaded**

```python
class GracefulDegradation:
    def __init__(self):
        self.system_load = 0  # 0-100
    
    def get_product_details(self, product_id):
        """Get product with degradation based on load"""
        if self.system_load < 50:
            # Full details with reviews, recommendations
            return self._get_full_details(product_id)
        
        elif self.system_load < 80:
            # Basic details without reviews
            return self._get_basic_details(product_id)
        
        else:
            # Cached data only
            return self._get_cached_details(product_id)
    
    def _get_full_details(self, product_id):
        """Full product details (slow)"""
        product = self._fetch_product(product_id)
        product['reviews'] = self._fetch_reviews(product_id)
        product['recommendations'] = self._get_recommendations(product_id)
        return product
    
    def _get_basic_details(self, product_id):
        """Basic product details (faster)"""
        return self._fetch_product(product_id)
    
    def _get_cached_details(self, product_id):
        """Cached data (fastest)"""
        return self._get_from_cache(product_id)
    
    def update_system_load(self, load):
        """Update current system load"""
        self.system_load = load

# Usage
service = GracefulDegradation()

# Normal load
service.update_system_load(30)
product = service.get_product_details('prod-123')  # Full details

# High load
service.update_system_load(85)
product = service.get_product_details('prod-123')  # Cached only
```

---

## Chaos Engineering

**Test system resilience**

```python
import random

class ChaosMonkey:
    """Inject failures to test resilience"""
    
    def __init__(self, failure_rate=0.1):
        self.failure_rate = failure_rate
    
    def maybe_fail(self, error_message="Chaos Monkey struck!"):
        """Randomly fail requests"""
        if random.random() < self.failure_rate:
            raise Exception(error_message)
    
    def add_latency(self, min_seconds=0.1, max_seconds=2):
        """Add random latency"""
        delay = random.uniform(min_seconds, max_seconds)
        time.sleep(delay)
    
    def kill_connection(self):
        """Simulate connection failure"""
        if random.random() < self.failure_rate:
            raise ConnectionError("Connection lost")

# Chaos testing
class ResilientService:
    def __init__(self, enable_chaos=False):
        self.chaos = ChaosMonkey(failure_rate=0.2) if enable_chaos else None
        self.circuit_breaker = CircuitBreaker()
        self.retry = ExponentialBackoff()
    
    def call_external_service(self):
        """Make resilient external call"""
        def operation():
            # Inject chaos if enabled
            if self.chaos:
                self.chaos.maybe_fail()
                self.chaos.add_latency()
            
            # Actual service call
            return "Success"
        
        # Use retry + circuit breaker
        return self.retry.execute(
            lambda: self.circuit_breaker.call(operation)
        )

# Test with chaos
service = ResilientService(enable_chaos=True)

for i in range(20):
    try:
        result = service.call_external_service()
        print(f"Request {i}: {result}")
    except Exception as e:
        print(f"Request {i}: Failed - {e}")
```

This comprehensive guide covers all essential fault tolerance and resilience patterns for building robust distributed systems!
