# Concurrency & Multithreading Guide

## Table of Contents

1. [Threading Fundamentals](#threading-fundamentals)
2. [Thread Synchronization](#thread-synchronization)
3. [Deadlocks](#deadlocks)
4. [Locks vs Non-blocking](#locks-vs-non-blocking)
5. [Producer-Consumer Pattern](#producer-consumer-pattern)
6. [Thread Pools](#thread-pools)
7. [Concurrent Data Structures](#concurrent-data-structures)
8. [Best Practices](#best-practices)

---

## Threading Fundamentals

### Creating and Managing Threads

```python
import threading
import time
from concurrent.futures import ThreadPoolExecutor

# Method 1: Using Thread class
class WorkerThread(threading.Thread):
    def __init__(self, name, task_id):
        super().__init__()
        self.name = name
        self.task_id = task_id
    
    def run(self):
        print(f"Thread {self.name} starting task {self.task_id}")
        time.sleep(2)
        print(f"Thread {self.name} completed task {self.task_id}")

# Method 2: Using thread function
def worker_function(name, task_id):
    print(f"Thread {name} starting task {task_id}")
    time.sleep(2)
    print(f"Thread {name} completed task {task_id}")

# Creating threads
thread1 = WorkerThread("Worker-1", 1)
thread2 = threading.Thread(target=worker_function, args=("Worker-2", 2))

# Starting threads
thread1.start()
thread2.start()

# Waiting for completion
thread1.join()
thread2.join()

print("All threads completed")
```

### Java Threading

```java
// Method 1: Extending Thread class
class WorkerThread extends Thread {
    private int taskId;
    
    public WorkerThread(int taskId) {
        this.taskId = taskId;
    }
    
    @Override
    public void run() {
        System.out.println("Thread " + Thread.currentThread().getName() + 
                          " starting task " + taskId);
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Thread " + Thread.currentThread().getName() + 
                          " completed task " + taskId);
    }
}

// Method 2: Implementing Runnable
class WorkerRunnable implements Runnable {
    private int taskId;
    
    public WorkerRunnable(int taskId) {
        this.taskId = taskId;
    }
    
    @Override
    public void run() {
        System.out.println("Thread " + Thread.currentThread().getName() + 
                          " processing task " + taskId);
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

// Method 3: Using Lambda (Java 8+)
public class ThreadingExample {
    public static void main(String[] args) {
        // Using Thread class
        Thread thread1 = new WorkerThread(1);
        thread1.start();
        
        // Using Runnable
        Thread thread2 = new Thread(new WorkerRunnable(2));
        thread2.start();
        
        // Using Lambda
        Thread thread3 = new Thread(() -> {
            System.out.println("Lambda thread executing");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread3.start();
        
        // Wait for all threads
        try {
            thread1.join();
            thread2.join();
            thread3.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

---

## Thread Synchronization

### Race Conditions

```python
import threading

# PROBLEM: Race condition
class BankAccount:
    def __init__(self, balance):
        self.balance = balance
    
    def withdraw(self, amount):
        if self.balance >= amount:
            # Context switch can happen here!
            time.sleep(0.001)  # Simulate processing
            self.balance -= amount
            return True
        return False

account = BankAccount(1000)

def withdraw_money(amount):
    if account.withdraw(amount):
        print(f"Withdrew ${amount}, Balance: ${account.balance}")

# Create multiple threads trying to withdraw
threads = []
for i in range(10):
    thread = threading.Thread(target=withdraw_money, args=(200,))
    threads.append(thread)
    thread.start()

for thread in threads:
    thread.join()

print(f"Final balance: ${account.balance}")
# PROBLEM: Balance might go negative!
```

### Solution 1: Using Locks

```python
import threading

class BankAccountWithLock:
    def __init__(self, balance):
        self.balance = balance
        self.lock = threading.Lock()
    
    def withdraw(self, amount):
        with self.lock:  # Acquire lock
            if self.balance >= amount:
                time.sleep(0.001)
                self.balance -= amount
                return True
            return False
    
    def deposit(self, amount):
        with self.lock:
            self.balance += amount

account = BankAccountWithLock(1000)

def withdraw_money(amount):
    if account.withdraw(amount):
        print(f"Withdrew ${amount}, Balance: ${account.balance}")

threads = []
for i in range(10):
    thread = threading.Thread(target=withdraw_money, args=(200,))
    threads.append(thread)
    thread.start()

for thread in threads:
    thread.join()

print(f"Final balance: ${account.balance}")
# Now balance is correct!
```

### Solution 2: Using RLock (Reentrant Lock)

```python
class BankAccountWithRLock:
    def __init__(self, balance):
        self.balance = balance
        self.lock = threading.RLock()  # Reentrant lock
    
    def withdraw(self, amount):
        with self.lock:
            if self.balance >= amount:
                self.balance -= amount
                return True
            return False
    
    def transfer(self, other_account, amount):
        with self.lock:  # Can acquire lock again (reentrant)
            if self.withdraw(amount):
                other_account.deposit(amount)
                return True
            return False
    
    def deposit(self, amount):
        with self.lock:
            self.balance += amount
```

### Java Synchronization

```java
// Method 1: Synchronized methods
public class BankAccount {
    private double balance;
    
    public BankAccount(double balance) {
        this.balance = balance;
    }
    
    public synchronized boolean withdraw(double amount) {
        if (balance >= amount) {
            try {
                Thread.sleep(10);  // Simulate processing
            } catch (InterruptedException e) {}
            balance -= amount;
            return true;
        }
        return false;
    }
    
    public synchronized void deposit(double amount) {
        balance += amount;
    }
    
    public synchronized double getBalance() {
        return balance;
    }
}

// Method 2: Synchronized blocks
public class BankAccount {
    private double balance;
    private final Object lock = new Object();
    
    public boolean withdraw(double amount) {
        synchronized(lock) {
            if (balance >= amount) {
                balance -= amount;
                return true;
            }
            return false;
        }
    }
    
    public void deposit(double amount) {
        synchronized(lock) {
            balance += amount;
        }
    }
}

// Method 3: Using ReentrantLock
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class BankAccount {
    private double balance;
    private final Lock lock = new ReentrantLock();
    
    public boolean withdraw(double amount) {
        lock.lock();
        try {
            if (balance >= amount) {
                balance -= amount;
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }
    
    public void deposit(double amount) {
        lock.lock();
        try {
            balance += amount;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## Deadlocks

### What is Deadlock?

**Four conditions for deadlock:**
1. **Mutual Exclusion**: Resources cannot be shared
2. **Hold and Wait**: Thread holds resources while waiting for more
3. **No Preemption**: Resources cannot be forcibly taken
4. **Circular Wait**: Circular chain of threads waiting for resources

### Deadlock Example

```python
import threading
import time

lock1 = threading.Lock()
lock2 = threading.Lock()

def thread1_work():
    print("Thread 1: Trying to acquire lock1")
    with lock1:
        print("Thread 1: Acquired lock1")
        time.sleep(0.1)  # Simulate work
        
        print("Thread 1: Trying to acquire lock2")
        with lock2:
            print("Thread 1: Acquired lock2")
            print("Thread 1: Work completed")

def thread2_work():
    print("Thread 2: Trying to acquire lock2")
    with lock2:
        print("Thread 2: Acquired lock2")
        time.sleep(0.1)  # Simulate work
        
        print("Thread 2: Trying to acquire lock1")
        with lock1:
            print("Thread 2: Acquired lock1")
            print("Thread 2: Work completed")

# This will cause deadlock!
t1 = threading.Thread(target=thread1_work)
t2 = threading.Thread(target=thread2_work)

t1.start()
t2.start()

t1.join()
t2.join()
```

### Deadlock Prevention

**Solution 1: Lock Ordering**

```python
import threading

lock1 = threading.Lock()
lock2 = threading.Lock()

def thread1_work():
    # Always acquire locks in the same order
    with lock1:
        print("Thread 1: Acquired lock1")
        time.sleep(0.1)
        with lock2:
            print("Thread 1: Acquired lock2")
            print("Thread 1: Work completed")

def thread2_work():
    # Same order as thread1
    with lock1:  # Changed from lock2
        print("Thread 2: Acquired lock1")
        time.sleep(0.1)
        with lock2:
            print("Thread 2: Acquired lock2")
            print("Thread 2: Work completed")

# No deadlock!
t1 = threading.Thread(target=thread1_work)
t2 = threading.Thread(target=thread2_work)
t1.start()
t2.start()
t1.join()
t2.join()
```

**Solution 2: Timeout**

```python
def thread_work_with_timeout():
    acquired1 = lock1.acquire(timeout=1)
    if not acquired1:
        print("Could not acquire lock1, aborting")
        return
    
    try:
        acquired2 = lock2.acquire(timeout=1)
        if not acquired2:
            print("Could not acquire lock2, releasing lock1")
            return
        
        try:
            print("Both locks acquired, doing work")
        finally:
            lock2.release()
    finally:
        lock1.release()
```

**Solution 3: Banker's Algorithm (Resource Allocation)**

```python
class ResourceAllocator:
    def __init__(self, resources):
        self.available = resources.copy()
        self.allocated = {}
        self.max_needed = {}
        self.lock = threading.Lock()
    
    def is_safe_state(self, request, thread_id):
        """Check if granting request leads to safe state"""
        # Simulate allocation
        temp_available = self.available.copy()
        
        for resource, amount in request.items():
            temp_available[resource] -= amount
            if temp_available[resource] < 0:
                return False
        
        # Check if remaining threads can complete
        return self._can_complete(temp_available)
    
    def request_resources(self, thread_id, request):
        with self.lock:
            if self.is_safe_state(request, thread_id):
                # Allocate resources
                for resource, amount in request.items():
                    self.available[resource] -= amount
                    if thread_id not in self.allocated:
                        self.allocated[thread_id] = {}
                    self.allocated[thread_id][resource] = \
                        self.allocated[thread_id].get(resource, 0) + amount
                return True
            return False
    
    def release_resources(self, thread_id):
        with self.lock:
            if thread_id in self.allocated:
                for resource, amount in self.allocated[thread_id].items():
                    self.available[resource] += amount
                del self.allocated[thread_id]
```

---

## Locks vs Non-blocking

### Traditional Locking

```java
// Pessimistic locking - blocks other threads
public class Counter {
    private int count = 0;
    private final Lock lock = new ReentrantLock();
    
    public void increment() {
        lock.lock();
        try {
            count++;  // Only one thread at a time
        } finally {
            lock.unlock();
        }
    }
    
    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

### Non-blocking (Lock-free) Approach

```java
import java.util.concurrent.atomic.AtomicInteger;

// Optimistic approach - uses Compare-and-Swap (CAS)
public class LockFreeCounter {
    private AtomicInteger count = new AtomicInteger(0);
    
    public void increment() {
        int current;
        int next;
        do {
            current = count.get();
            next = current + 1;
            // Try to update if value hasn't changed
        } while (!count.compareAndSet(current, next));
    }
    
    public int getCount() {
        return count.get();  // No locking needed
    }
}

// Performance comparison
public class PerformanceTest {
    public static void main(String[] args) throws InterruptedException {
        final int THREADS = 10;
        final int ITERATIONS = 100000;
        
        // Test with locks
        Counter lockedCounter = new Counter();
        long start = System.currentTimeMillis();
        
        Thread[] threads = new Thread[THREADS];
        for (int i = 0; i < THREADS; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < ITERATIONS; j++) {
                    lockedCounter.increment();
                }
            });
            threads[i].start();
        }
        
        for (Thread t : threads) {
            t.join();
        }
        
        long lockTime = System.currentTimeMillis() - start;
        System.out.println("Locked counter time: " + lockTime + "ms");
        
        // Test lock-free
        LockFreeCounter lockFreeCounter = new LockFreeCounter();
        start = System.currentTimeMillis();
        
        for (int i = 0; i < THREADS; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < ITERATIONS; j++) {
                    lockFreeCounter.increment();
                }
            });
            threads[i].start();
        }
        
        for (Thread t : threads) {
            t.join();
        }
        
        long lockFreeTime = System.currentTimeMillis() - start;
        System.out.println("Lock-free counter time: " + lockFreeTime + "ms");
    }
}
```

### Python Atomic Operations

```python
import threading
from queue import Queue

# Using Queue (thread-safe)
class LockFreeCounter:
    def __init__(self):
        self.queue = Queue()
        self.count = 0
        
        # Background thread to process increments
        self.worker = threading.Thread(target=self._process_increments, daemon=True)
        self.worker.start()
    
    def increment(self):
        self.queue.put(1)
    
    def _process_increments(self):
        while True:
            increment = self.queue.get()
            self.count += increment
    
    def get_count(self):
        return self.count
```

---

## Producer-Consumer Pattern

### Using Queue (Thread-safe)

```python
import threading
import time
from queue import Queue
import random

class ProducerConsumer:
    def __init__(self, buffer_size=10):
        self.queue = Queue(maxsize=buffer_size)
        self.stop_flag = threading.Event()
    
    def producer(self, producer_id):
        """Produces items and adds to queue"""
        while not self.stop_flag.is_set():
            item = random.randint(1, 100)
            self.queue.put(item)  # Blocks if queue is full
            print(f"Producer {producer_id} produced: {item} "
                  f"(Queue size: {self.queue.qsize()})")
            time.sleep(random.uniform(0.1, 0.5))
    
    def consumer(self, consumer_id):
        """Consumes items from queue"""
        while not self.stop_flag.is_set() or not self.queue.empty():
            try:
                item = self.queue.get(timeout=1)  # Blocks if queue is empty
                print(f"Consumer {consumer_id} consumed: {item} "
                      f"(Queue size: {self.queue.qsize()})")
                time.sleep(random.uniform(0.5, 1.0))  # Processing time
                self.queue.task_done()
            except:
                pass
    
    def run(self, num_producers=2, num_consumers=3, duration=5):
        threads = []
        
        # Start producers
        for i in range(num_producers):
            t = threading.Thread(target=self.producer, args=(i,))
            t.start()
            threads.append(t)
        
        # Start consumers
        for i in range(num_consumers):
            t = threading.Thread(target=self.consumer, args=(i,))
            t.start()
            threads.append(t)
        
        # Run for specified duration
        time.sleep(duration)
        self.stop_flag.set()
        
        # Wait for all threads
        for t in threads:
            t.join()
        
        print("All producers and consumers stopped")

# Usage
pc = ProducerConsumer(buffer_size=5)
pc.run(num_producers=2, num_consumers=3, duration=10)
```

### Using Condition Variables

```python
import threading

class BoundedBuffer:
    def __init__(self, capacity):
        self.capacity = capacity
        self.buffer = []
        self.lock = threading.Lock()
        self.not_full = threading.Condition(self.lock)
        self.not_empty = threading.Condition(self.lock)
    
    def produce(self, item):
        with self.not_full:
            while len(self.buffer) >= self.capacity:
                print("Buffer full, producer waiting...")
                self.not_full.wait()
            
            self.buffer.append(item)
            print(f"Produced {item}, buffer size: {len(self.buffer)}")
            self.not_empty.notify()
    
    def consume(self):
        with self.not_empty:
            while len(self.buffer) == 0:
                print("Buffer empty, consumer waiting...")
                self.not_empty.wait()
            
            item = self.buffer.pop(0)
            print(f"Consumed {item}, buffer size: {len(self.buffer)}")
            self.not_full.notify()
            return item

# Usage
buffer = BoundedBuffer(5)

def producer():
    for i in range(10):
        buffer.produce(i)
        time.sleep(0.1)

def consumer():
    for i in range(10):
        item = buffer.consume()
        time.sleep(0.5)

t1 = threading.Thread(target=producer)
t2 = threading.Thread(target=consumer)

t1.start()
t2.start()
t1.join()
t2.join()
```

### Java BlockingQueue

```java
import java.util.concurrent.*;

public class ProducerConsumerExample {
    public static void main(String[] args) {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);
        
        // Producer
        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 20; i++) {
                    queue.put(i);  // Blocks if queue is full
                    System.out.println("Produced: " + i);
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        // Consumer
        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    Integer item = queue.take();  // Blocks if queue is empty
                    System.out.println("Consumed: " + item);
                    Thread.sleep(500);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        producer.start();
        consumer.start();
    }
}
```

---

## Thread Pools

### Python ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def task(task_id):
    print(f"Task {task_id} starting")
    time.sleep(2)
    print(f"Task {task_id} completed")
    return f"Result from task {task_id}"

# Create thread pool
with ThreadPoolExecutor(max_workers=5) as executor:
    # Submit tasks
    futures = [executor.submit(task, i) for i in range(10)]
    
    # Get results as they complete
    for future in as_completed(futures):
        result = future.result()
        print(result)

# Alternative: map
with ThreadPoolExecutor(max_workers=5) as executor:
    results = executor.map(task, range(10))
    for result in results:
        print(result)
```

### Java ExecutorService

```java
import java.util.concurrent.*;

public class ThreadPoolExample {
    public static void main(String[] args) throws InterruptedException {
        // Fixed thread pool
        ExecutorService executor = Executors.newFixedThreadPool(5);
        
        // Submit tasks
        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            executor.submit(() -> {
                System.out.println("Task " + taskId + " executing on " + 
                                 Thread.currentThread().getName());
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                return "Result " + taskId;
            });
        }
        
        // Shutdown
        executor.shutdown();
        executor.awaitTermination(1, TimeUnit.MINUTES);
    }
}

// Custom thread pool
public class CustomThreadPool {
    public static void main(String[] args) {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            5,                      // Core pool size
            10,                     // Max pool size
            60L,                    // Keep alive time
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(100),  // Work queue
            new ThreadPoolExecutor.CallerRunsPolicy()  // Rejection policy
        );
        
        // Submit tasks
        for (int i = 0; i < 100; i++) {
            executor.execute(() -> {
                // Task logic
            });
        }
        
        executor.shutdown();
    }
}
```

---

## Concurrent Data Structures

### Java ConcurrentHashMap

```java
import java.util.concurrent.ConcurrentHashMap;

public class ConcurrentMapExample {
    private ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
    
    public void incrementCounter(String key) {
        map.compute(key, (k, v) -> v == null ? 1 : v + 1);
    }
    
    public void putIfAbsentDemo() {
        // Atomic operation
        map.putIfAbsent("key1", 100);
    }
    
    public void removeDemo() {
        // Remove only if value matches
        map.remove("key1", 100);
    }
    
    public void replaceDemo() {
        // Replace only if value matches
        map.replace("key1", 100, 200);
    }
}
```

### Python Thread-safe Collections

```python
from collections import defaultdict
from threading import Lock

class ThreadSafeCounter:
    def __init__(self):
        self.counts = defaultdict(int)
        self.lock = Lock()
    
    def increment(self, key):
        with self.lock:
            self.counts[key] += 1
    
    def get(self, key):
        with self.lock:
            return self.counts[key]

# Using queue for thread-safe operations
from queue import Queue

class ThreadSafeCache:
    def __init__(self):
        self.cache = {}
        self.lock = Lock()
    
    def get(self, key):
        with self.lock:
            return self.cache.get(key)
    
    def set(self, key, value):
        with self.lock:
            self.cache[key] = value
```

---

## Best Practices

### 1. Minimize Lock Scope

```python
# BAD: Lock held for too long
def bad_example(self):
    with self.lock:
        data = self.fetch_data()  # Network call - SLOW!
        processed = self.process(data)  # CPU intensive - SLOW!
        self.update(processed)

# GOOD: Lock only when necessary
def good_example(self):
    data = self.fetch_data()  # No lock needed
    processed = self.process(data)  # No lock needed
    
    with self.lock:
        self.update(processed)  # Only critical section locked
```

### 2. Avoid Nested Locks

```python
# BAD: Nested locks increase deadlock risk
with lock1:
    with lock2:
        with lock3:
            # Complex nested locking

# GOOD: Single lock or lock-free design
with single_lock:
    # Single critical section
```

### 3. Use Higher-level Abstractions

```python
# GOOD: Use thread-safe queues instead of manual locking
from queue import Queue

queue = Queue()

def producer():
    queue.put(item)

def consumer():
    item = queue.get()
```

### 4. Thread-local Storage

```python
import threading

# Thread-local storage
thread_local = threading.local()

def worker():
    # Each thread has its own copy
    thread_local.request_id = generate_id()
    process_request(thread_local.request_id)
```

This comprehensive guide covers all essential aspects of concurrency and multithreading, from basics to advanced patterns!

