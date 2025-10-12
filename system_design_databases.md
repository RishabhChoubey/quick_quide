# Databases & Storage Systems Guide

## Table of Contents

1. [Database Fundamentals](#database-fundamentals)
2. [SQL vs NoSQL](#sql-vs-nossl)
3. [Sharding & Partitioning](#sharding--partitioning)
4. [Indexing Strategies](#indexing-strategies)
5. [Caching](#caching)
6. [Replication](#replication)
7. [How to Choose a Database](#how-to-choose-a-database)
8. [CAP Theorem in Practice](#cap-theorem-in-practice)

---

## Database Fundamentals

### ACID Properties (SQL Databases)

**Atomicity**: All or nothing - transactions complete fully or not at all
**Consistency**: Data remains valid according to rules
**Isolation**: Concurrent transactions don't interfere
**Durability**: Committed data persists even after system failure

```sql
-- Example: Bank transfer (ACID transaction)
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A123';
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B456';

-- If any statement fails, entire transaction rolls back
COMMIT;
```

### BASE Properties (NoSQL Databases)

**Basically Available**: System remains available
**Soft state**: State may change over time
**Eventually consistent**: System becomes consistent eventually

---

## SQL vs NoSQL

### Relational Databases (SQL)

**Best for**: Structured data, complex queries, transactions

**Popular Databases**:
- PostgreSQL
- MySQL
- Oracle
- SQL Server

```sql
-- Schema: Users and Orders
CREATE TABLE users (
    user_id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    total DECIMAL(10, 2),
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE TABLE order_items (
    item_id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT,
    product_id INT,
    quantity INT,
    price DECIMAL(10, 2),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- Complex JOIN query
SELECT 
    u.name,
    o.order_id,
    o.total,
    oi.product_id,
    oi.quantity
FROM users u
JOIN orders o ON u.user_id = o.user_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE u.user_id = 123
ORDER BY o.created_at DESC;
```

### Document Store (NoSQL)

**Best for**: Flexible schema, hierarchical data, rapid development

**Popular Databases**: MongoDB, CouchDB, Firebase

```javascript
// MongoDB - Denormalized document
db.orders.insertOne({
    _id: ObjectId("507f1f77bcf86cd799439011"),
    order_id: "ORD-123",
    user: {
        user_id: 123,
        email: "user@example.com",
        name: "John Doe"
    },
    items: [
        {
            product_id: "PROD-1",
            name: "Laptop",
            quantity: 1,
            price: 999.99
        },
        {
            product_id: "PROD-2",
            name: "Mouse",
            quantity: 2,
            price: 25.00
        }
    ],
    total: 1049.99,
    status: "completed",
    created_at: ISODate("2024-01-15T10:30:00Z")
});

// Query
db.orders.find({
    "user.user_id": 123
}).sort({ created_at: -1 });

// Aggregation pipeline
db.orders.aggregate([
    { $match: { "user.user_id": 123 } },
    { $unwind: "$items" },
    { $group: {
        _id: "$items.product_id",
        total_quantity: { $sum: "$items.quantity" },
        total_revenue: { $sum: { $multiply: ["$items.quantity", "$items.price"] } }
    }}
]);
```

### Key-Value Store

**Best for**: Caching, session management, simple lookups

**Popular Databases**: Redis, Memcached, DynamoDB

```python
import redis

# Redis examples
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Simple key-value
redis_client.set('user:123:name', 'John Doe')
name = redis_client.get('user:123:name')

# Hash (object)
redis_client.hset('user:123', mapping={
    'name': 'John Doe',
    'email': 'john@example.com',
    'age': 30
})
user_data = redis_client.hgetall('user:123')

# List (queue)
redis_client.lpush('task_queue', 'task1')
redis_client.lpush('task_queue', 'task2')
task = redis_client.rpop('task_queue')

# Set (unique values)
redis_client.sadd('user:123:following', 'user:456', 'user:789')
following = redis_client.smembers('user:123:following')

# Sorted Set (leaderboard)
redis_client.zadd('leaderboard', {'player1': 1000, 'player2': 1500})
top_players = redis_client.zrevrange('leaderboard', 0, 9, withscores=True)

# Expiration (TTL)
redis_client.setex('session:abc123', 3600, 'session_data')  # Expires in 1 hour
```

### Column-Family Store

**Best for**: Time-series data, analytics, wide tables

**Popular Databases**: Cassandra, HBase, ScyllaDB

```sql
-- Cassandra CQL
CREATE KEYSPACE ecommerce WITH replication = {
    'class': 'SimpleStrategy',
    'replication_factor': 3
};

CREATE TABLE orders_by_user (
    user_id UUID,
    order_time TIMESTAMP,
    order_id UUID,
    total DECIMAL,
    status TEXT,
    PRIMARY KEY (user_id, order_time)
) WITH CLUSTERING ORDER BY (order_time DESC);

-- Query by partition key
SELECT * FROM orders_by_user WHERE user_id = 123;

-- Time range query
SELECT * FROM orders_by_user 
WHERE user_id = 123 
AND order_time >= '2024-01-01' 
AND order_time < '2024-02-01';
```

### Graph Database

**Best for**: Relationships, social networks, recommendations

**Popular Databases**: Neo4j, Amazon Neptune

```cypher
// Neo4j Cypher queries

// Create nodes and relationships
CREATE (alice:Person {name: 'Alice', age: 30})
CREATE (bob:Person {name: 'Bob', age: 28})
CREATE (product:Product {name: 'Laptop', price: 999})
CREATE (alice)-[:FRIEND_OF]->(bob)
CREATE (alice)-[:PURCHASED]->(product)
CREATE (bob)-[:VIEWED]->(product)

// Find friends of friends
MATCH (user:Person {name: 'Alice'})-[:FRIEND_OF]->(friend)-[:FRIEND_OF]->(fof)
WHERE fof <> user
RETURN fof.name

// Product recommendations (friends also bought)
MATCH (user:Person {name: 'Alice'})-[:FRIEND_OF]->(friend)-[:PURCHASED]->(product)
WHERE NOT (user)-[:PURCHASED]->(product)
RETURN product.name, COUNT(*) as friend_count
ORDER BY friend_count DESC
LIMIT 5

// Shortest path
MATCH path = shortestPath(
    (alice:Person {name: 'Alice'})-[:FRIEND_OF*]-(target:Person {name: 'Target'})
)
RETURN path
```

---

## Sharding & Partitioning

### Horizontal Partitioning (Sharding)

**Split data across multiple servers based on a shard key**

```python
class ShardManager:
    def __init__(self, num_shards=4):
        self.num_shards = num_shards
        self.shards = [DatabaseConnection(f"shard_{i}") for i in range(num_shards)]
    
    def get_shard_key(self, user_id):
        """Hash-based sharding"""
        return hash(user_id) % self.num_shards
    
    def get_user(self, user_id):
        shard_id = self.get_shard_key(user_id)
        return self.shards[shard_id].query(f"SELECT * FROM users WHERE user_id = {user_id}")
    
    def insert_user(self, user_data):
        user_id = user_data['user_id']
        shard_id = self.get_shard_key(user_id)
        return self.shards[shard_id].insert('users', user_data)

# Range-based sharding
class RangeShardManager:
    def __init__(self):
        self.shard_ranges = [
            (0, 1000000, 'shard_0'),      # Users 0-1M
            (1000001, 2000000, 'shard_1'), # Users 1M-2M
            (2000001, 3000000, 'shard_2'), # Users 2M-3M
        ]
    
    def get_shard(self, user_id):
        for start, end, shard_name in self.shard_ranges:
            if start <= user_id <= end:
                return shard_name
        raise ValueError("User ID out of range")

# Geographic sharding
class GeoShardManager:
    def __init__(self):
        self.shard_map = {
            'US': 'shard_us',
            'EU': 'shard_eu',
            'ASIA': 'shard_asia'
        }
    
    def get_shard(self, region):
        return self.shard_map.get(region, 'shard_default')
```

### Vertical Partitioning

**Split tables by columns**

```sql
-- Split large user table
CREATE TABLE user_profile (
    user_id INT PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(100),
    created_at TIMESTAMP
);

CREATE TABLE user_details (
    user_id INT PRIMARY KEY,
    bio TEXT,
    profile_picture BLOB,
    preferences JSON,
    FOREIGN KEY (user_id) REFERENCES user_profile(user_id)
);

-- Frequently accessed data in one table, rarely accessed in another
```

### Consistent Hashing

**For dynamic sharding (adding/removing nodes)**

```python
import hashlib
import bisect

class ConsistentHashing:
    def __init__(self, nodes=None, virtual_nodes=150):
        self.virtual_nodes = virtual_nodes
        self.ring = {}
        self.sorted_keys = []
        
        if nodes:
            for node in nodes:
                self.add_node(node)
    
    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
    
    def add_node(self, node):
        """Add node with virtual nodes for better distribution"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_key = self._hash(virtual_key)
            self.ring[hash_key] = node
            bisect.insort(self.sorted_keys, hash_key)
    
    def remove_node(self, node):
        """Remove node"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_key = self._hash(virtual_key)
            del self.ring[hash_key]
            self.sorted_keys.remove(hash_key)
    
    def get_node(self, key):
        """Get node for given key"""
        if not self.ring:
            return None
        
        hash_key = self._hash(key)
        idx = bisect.bisect_right(self.sorted_keys, hash_key)
        
        if idx == len(self.sorted_keys):
            idx = 0
        
        return self.ring[self.sorted_keys[idx]]

# Usage
ch = ConsistentHashing(['server1', 'server2', 'server3'])

# Get server for user
server = ch.get_node('user:123')  # Returns 'server2'

# Add new server (minimal data movement)
ch.add_node('server4')

# Remove server (redistributes its data)
ch.remove_node('server1')
```

---

## Indexing Strategies

### B-Tree Index (Default in most SQL databases)

```sql
-- Create index
CREATE INDEX idx_users_email ON users(email);

-- Composite index
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- Partial index
CREATE INDEX idx_active_orders ON orders(user_id) WHERE status = 'active';

-- Covering index (includes all columns needed for query)
CREATE INDEX idx_user_order_summary ON orders(user_id, total, status, created_at);

-- Query uses index
EXPLAIN SELECT * FROM users WHERE email = 'john@example.com';
-- Uses idx_users_email

-- Index for sorting
SELECT * FROM orders WHERE user_id = 123 ORDER BY created_at DESC;
-- Uses idx_orders_user_date
```

### Full-Text Search Index

```sql
-- PostgreSQL full-text search
CREATE INDEX idx_products_search ON products 
USING GIN (to_tsvector('english', name || ' ' || description));

-- Search query
SELECT * FROM products
WHERE to_tsvector('english', name || ' ' || description) 
      @@ to_tsquery('english', 'laptop & wireless');

-- Elasticsearch (better for full-text search)
```

```python
from elasticsearch import Elasticsearch

es = Elasticsearch(['localhost:9200'])

# Index document
es.index(index='products', id=1, body={
    'name': 'Wireless Laptop Mouse',
    'description': 'Ergonomic wireless mouse for laptops',
    'price': 25.99,
    'category': 'Electronics'
})

# Full-text search
results = es.search(index='products', body={
    'query': {
        'multi_match': {
            'query': 'laptop wireless',
            'fields': ['name^2', 'description']  # Boost name field
        }
    }
})

# Fuzzy search (typo tolerance)
results = es.search(index='products', body={
    'query': {
        'fuzzy': {
            'name': {
                'value': 'laptp',  # Typo
                'fuzziness': 'AUTO'
            }
        }
    }
})
```

### Geospatial Index

```sql
-- PostgreSQL with PostGIS
CREATE EXTENSION postgis;

CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    location GEOGRAPHY(POINT, 4326)
);

-- Create spatial index
CREATE INDEX idx_locations_gist ON locations USING GIST(location);

-- Find locations within 10km
SELECT name, ST_Distance(location, ST_MakePoint(-73.935242, 40.730610)::geography) as distance
FROM locations
WHERE ST_DWithin(
    location,
    ST_MakePoint(-73.935242, 40.730610)::geography,
    10000  -- 10km in meters
)
ORDER BY distance;
```

---

## Caching

### Cache-Aside (Lazy Loading)

```python
import redis
import json

class CacheAside:
    def __init__(self, redis_client, db_client):
        self.cache = redis_client
        self.db = db_client
    
    def get_user(self, user_id):
        cache_key = f"user:{user_id}"
        
        # Try cache first
        cached_data = self.cache.get(cache_key)
        if cached_data:
            print("Cache hit!")
            return json.loads(cached_data)
        
        # Cache miss - fetch from database
        print("Cache miss - fetching from DB")
        user_data = self.db.query(f"SELECT * FROM users WHERE user_id = {user_id}")
        
        if user_data:
            # Store in cache with TTL
            self.cache.setex(cache_key, 3600, json.dumps(user_data))
        
        return user_data
    
    def update_user(self, user_id, user_data):
        # Update database
        self.db.update('users', user_id, user_data)
        
        # Invalidate cache
        cache_key = f"user:{user_id}"
        self.cache.delete(cache_key)
```

### Write-Through Cache

```python
class WriteThroughCache:
    def update_user(self, user_id, user_data):
        # Write to database first
        self.db.update('users', user_id, user_data)
        
        # Update cache
        cache_key = f"user:{user_id}"
        self.cache.setex(cache_key, 3600, json.dumps(user_data))
```

### Write-Behind (Write-Back) Cache

```python
from queue import Queue
import threading

class WriteBehindCache:
    def __init__(self, redis_client, db_client):
        self.cache = redis_client
        self.db = db_client
        self.write_queue = Queue()
        self._start_background_writer()
    
    def update_user(self, user_id, user_data):
        # Write to cache immediately
        cache_key = f"user:{user_id}"
        self.cache.setex(cache_key, 3600, json.dumps(user_data))
        
        # Queue database write (async)
        self.write_queue.put((user_id, user_data))
    
    def _start_background_writer(self):
        def writer():
            while True:
                user_id, user_data = self.write_queue.get()
                try:
                    self.db.update('users', user_id, user_data)
                except Exception as e:
                    print(f"DB write failed: {e}")
                    # Retry logic or dead letter queue
        
        thread = threading.Thread(target=writer, daemon=True)
        thread.start()
```

### Cache Eviction Policies

```python
# LRU (Least Recently Used) - Redis supports this
redis_client.config_set('maxmemory-policy', 'allkeys-lru')

# LFU (Least Frequently Used)
redis_client.config_set('maxmemory-policy', 'allkeys-lfu')

# TTL-based expiration
redis_client.setex('key', 3600, 'value')  # Expires in 1 hour
```

---

## Replication

### Master-Slave Replication

```python
class MasterSlaveDB:
    def __init__(self, master, slaves):
        self.master = master
        self.slaves = slaves
        self.current_slave = 0
    
    def write(self, query):
        """All writes go to master"""
        return self.master.execute(query)
    
    def read(self, query):
        """Distribute reads across slaves (round-robin)"""
        slave = self.slaves[self.current_slave]
        self.current_slave = (self.current_slave + 1) % len(self.slaves)
        return slave.execute(query)

# Usage
db = MasterSlaveDB(
    master=DatabaseConnection('master-db'),
    slaves=[
        DatabaseConnection('slave-1'),
        DatabaseConnection('slave-2'),
        DatabaseConnection('slave-3')
    ]
)

# Write
db.write("INSERT INTO users VALUES (...)")

# Read (load balanced across slaves)
user = db.read("SELECT * FROM users WHERE user_id = 123")
```

### Multi-Master Replication

```python
class MultiMasterDB:
    def __init__(self, masters):
        self.masters = masters
        self.current_master = 0
    
    def write(self, query):
        """Distribute writes across masters"""
        master = self.masters[self.current_master]
        self.current_master = (self.current_master + 1) % len(self.masters)
        return master.execute(query)
    
    def read(self, query):
        """Read from any master"""
        master = self.masters[self.current_master]
        return master.execute(query)
```

### Conflict Resolution

```python
# Last-Write-Wins (LWW)
def resolve_conflict_lww(version1, version2):
    return version1 if version1['timestamp'] > version2['timestamp'] else version2

# Vector Clocks (Cassandra, Riak)
class VectorClock:
    def __init__(self):
        self.clock = {}
    
    def increment(self, node_id):
        self.clock[node_id] = self.clock.get(node_id, 0) + 1
    
    def merge(self, other):
        for node_id, version in other.clock.items():
            self.clock[node_id] = max(self.clock.get(node_id, 0), version)
    
    def happens_before(self, other):
        """Check if this clock happened before other"""
        return all(
            self.clock.get(node, 0) <= other.clock.get(node, 0)
            for node in set(self.clock.keys()) | set(other.clock.keys())
        ) and self.clock != other.clock
```

---

## How to Choose a Database

### Decision Tree

```
Start
│
├─ Need ACID transactions + complex queries?
│  └─ YES → PostgreSQL / MySQL
│  
├─ Need flexible schema + scalability?
│  └─ YES → MongoDB / DynamoDB
│
├─ Need high-speed caching?
│  └─ YES → Redis / Memcached
│
├─ Need time-series data?
│  └─ YES → InfluxDB / TimescaleDB
│
├─ Need full-text search?
│  └─ YES → Elasticsearch / Solr
│
├─ Need graph relationships?
│  └─ YES → Neo4j / Amazon Neptune
│
└─ Need wide-column analytics?
   └─ YES → Cassandra / HBase
```

### Use Case Examples

| **Use Case** | **Recommended Database** | **Reason** |
|--------------|-------------------------|------------|
| E-commerce transactions | PostgreSQL | ACID, complex queries, joins |
| User sessions | Redis | Fast, in-memory, TTL support |
| Product catalog | MongoDB | Flexible schema, nested data |
| Real-time analytics | Cassandra | Write-heavy, time-series |
| Social network | Neo4j | Relationship-focused queries |
| Logging | Elasticsearch | Full-text search, aggregations |
| IoT sensor data | InfluxDB | Time-series optimization |

This comprehensive database guide covers all essential concepts for designing scalable storage systems!
