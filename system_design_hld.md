# High-Level Design (HLD) Guide: Building Scalable Distributed Systems

## Table of Contents

1. [Introduction to System Design](#introduction-to-system-design)
2. [Core Concepts](#core-concepts)
3. [Scalability Patterns](#scalability-patterns)
4. [Reliability Patterns](#reliability-patterns)
5. [Real-World System Designs](#real-world-system-designs)
6. [Architecture Patterns](#architecture-patterns)
7. [Best Practices](#best-practices)

---

## Introduction to System Design

### What is High-Level Design?

High-Level Design (HLD) focuses on the architecture and design of large-scale distributed systems. It deals with:
- System architecture and component interaction
- Data flow and storage strategies
- Scalability and performance optimization
- Reliability and fault tolerance
- Trade-offs and design decisions

### Key Principles

1. **Scalability**: System can handle increasing load
2. **Reliability**: System works correctly even with failures
3. **Availability**: System is operational when needed
4. **Maintainability**: System is easy to update and debug
5. **Performance**: System responds quickly to requests

---

## Core Concepts

### CAP Theorem

**Consistency, Availability, Partition Tolerance - Pick Two**

```
┌─────────────────────────────────────┐
│         CAP Theorem                 │
├─────────────────────────────────────┤
│                                     │
│     Consistency (C)                 │
│     ↓                               │
│     All nodes see same data         │
│                                     │
│     Availability (A)                │
│     ↓                               │
│     Every request gets response     │
│                                     │
│     Partition Tolerance (P)         │
│     ↓                               │
│     System works despite failures   │
│                                     │
│  You can only choose 2 of 3:        │
│  - CP: Consistency + Partition      │
│  - AP: Availability + Partition     │
│  - CA: Consistency + Availability   │
│        (Not realistic in dist. sys) │
└─────────────────────────────────────┘
```

**Examples:**
- **CP Systems**: MongoDB, HBase, Redis (with persistence)
- **AP Systems**: Cassandra, DynamoDB, CouchDB
- **CA Systems**: Traditional RDBMS (single node)

---

### How to Achieve Each CAP Property

#### 1. Achieving Consistency (C)

**Strong Consistency**: All nodes see the same data at the same time.

**Techniques:**

**a) Consensus Algorithms (Raft/Paxos)**
```python
class RaftNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers
        self.state = "FOLLOWER"  # FOLLOWER, CANDIDATE, LEADER
        self.current_term = 0
        self.voted_for = None
        self.log = []
        self.commit_index = 0
    
    def replicate_write(self, data):
        """Leader replicates to majority before committing"""
        if self.state != "LEADER":
            raise Exception("Not the leader")
        
        # Append to local log
        log_entry = {
            "term": self.current_term,
            "data": data,
            "index": len(self.log)
        }
        self.log.append(log_entry)
        
        # Replicate to followers
        acks = 1  # Leader's vote
        for peer in self.peers:
            if peer.append_entry(log_entry):
                acks += 1
        
        # Wait for majority
        if acks > len(self.peers) // 2:
            self.commit_index = log_entry["index"]
            return True
        else:
            # Rollback if no majority
            self.log.pop()
            return False
```

**b) Two-Phase Commit (2PC)**
```python
class TransactionCoordinator:
    def execute_distributed_transaction(self, participants, operations):
        transaction_id = generate_id()
        
        # Phase 1: Prepare
        prepare_responses = []
        for participant in participants:
            response = participant.prepare(transaction_id, operations)
            prepare_responses.append(response)
        
        # Check if all agreed to commit
        if all(r.vote == "YES" for r in prepare_responses):
            # Phase 2: Commit
            for participant in participants:
                participant.commit(transaction_id)
            return "COMMITTED"
        else:
            # Phase 2: Abort
            for participant in participants:
                participant.abort(transaction_id)
            return "ABORTED"

class DatabaseParticipant:
    def prepare(self, txn_id, operations):
        try:
            # Execute operations in memory/temp
            self.temp_changes[txn_id] = operations
            self.lock_resources(operations)
            return Response(vote="YES")
        except Exception:
            return Response(vote="NO")
    
    def commit(self, txn_id):
        # Apply changes permanently
        self.apply_changes(self.temp_changes[txn_id])
        self.unlock_resources()
        del self.temp_changes[txn_id]
    
    def abort(self, txn_id):
        # Discard changes
        del self.temp_changes[txn_id]
        self.unlock_resources()
```

**c) Quorum Reads/Writes**
```python
class QuorumBasedStorage:
    def __init__(self, nodes, replication_factor=3):
        self.nodes = nodes
        self.replication_factor = replication_factor
    
    def write_with_quorum(self, key, value):
        """Write to W nodes, where W > N/2"""
        W = (self.replication_factor // 2) + 1  # Write quorum
        
        # Get responsible nodes for this key
        target_nodes = self.get_nodes_for_key(key)[:self.replication_factor]
        
        # Write to all nodes
        success_count = 0
        version = self.get_next_version(key)
        
        for node in target_nodes:
            try:
                node.write(key, value, version)
                success_count += 1
            except Exception:
                continue
        
        # Check if quorum achieved
        if success_count >= W:
            return True
        else:
            # Rollback
            self.rollback_write(key, target_nodes)
            return False
    
    def read_with_quorum(self, key):
        """Read from R nodes, where R > N/2"""
        R = (self.replication_factor // 2) + 1  # Read quorum
        
        target_nodes = self.get_nodes_for_key(key)[:self.replication_factor]
        
        # Read from multiple nodes
        responses = []
        for node in target_nodes:
            try:
                data = node.read(key)
                responses.append(data)
            except Exception:
                continue
        
        if len(responses) >= R:
            # Return most recent version (highest version number)
            return max(responses, key=lambda x: x.version)
        else:
            raise Exception("Could not achieve read quorum")
```

**d) Read-After-Write Consistency**
```python
class ConsistentCache:
    def write(self, key, value):
        # Write to master database
        master_db.write(key, value)
        
        # Invalidate all caches
        cache_cluster.invalidate(key)
        
        # Optional: Wait for replicas to sync
        self.wait_for_replication(key, value)
    
    def read(self, key):
        # Always read from master for strong consistency
        return master_db.read(key)
        
        # OR read from replica with version check
        cached = cache.get(key)
        master_version = master_db.get_version(key)
        
        if cached and cached.version == master_version:
            return cached.value
        else:
            # Stale, read from master
            return master_db.read(key)
```

---

#### 2. Achieving Availability (A)

**Availability**: Every request receives a response (success or failure).

**Techniques:**

**a) Replication**
```python
class HighAvailabilityCluster:
    def __init__(self):
        self.primary = DatabaseNode("primary")
        self.replicas = [
            DatabaseNode("replica1"),
            DatabaseNode("replica2"),
            DatabaseNode("replica3")
        ]
    
    def write(self, key, value):
        # Write to primary
        try:
            self.primary.write(key, value)
        except Exception:
            # Promote a replica to primary
            self.failover()
            self.primary.write(key, value)
        
        # Async replication to replicas
        for replica in self.replicas:
            background_task.run(replica.write, key, value)
    
    def read(self, key):
        # Try primary first
        try:
            return self.primary.read(key)
        except Exception:
            # Fallback to replicas
            for replica in self.replicas:
                try:
                    return replica.read(key)
                except Exception:
                    continue
            raise Exception("All nodes unavailable")
    
    def failover(self):
        """Automatic failover to promote replica"""
        # Elect new primary from replicas
        new_primary = self.replicas[0]
        self.replicas.remove(new_primary)
        self.replicas.append(self.primary)
        self.primary = new_primary
```

**b) Eventually Consistent Writes**
```python
class EventuallyConsistentStore:
    def write(self, key, value):
        """Accept writes immediately without waiting"""
        # Write locally first
        local_store.write(key, value)
        
        # Queue for async replication
        replication_queue.enqueue({
            "operation": "write",
            "key": key,
            "value": value,
            "timestamp": time.time(),
            "node_id": self.node_id
        })
        
        # Return immediately (don't wait for replication)
        return {"status": "accepted", "key": key}
    
    def sync_replicas(self):
        """Background process to sync replicas"""
        while True:
            item = replication_queue.dequeue()
            
            for replica in self.replicas:
                try:
                    replica.write(item["key"], item["value"])
                except Exception:
                    # Retry later
                    replication_queue.enqueue(item)
                    break
```

**c) Multi-Master Replication**
```python
class MultiMasterNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers
        self.data = {}
        self.version_vectors = {}
    
    def write(self, key, value):
        """Accept write at any node"""
        # Increment version vector
        if key not in self.version_vectors:
            self.version_vectors[key] = {self.node_id: 0}
        self.version_vectors[key][self.node_id] += 1
        
        # Write locally
        self.data[key] = {
            "value": value,
            "version": self.version_vectors[key].copy(),
            "timestamp": time.time()
        }
        
        # Replicate to other masters (async)
        for peer in self.peers:
            background_task.run(
                peer.replicate,
                key,
                value,
                self.version_vectors[key]
            )
        
        return {"status": "success"}
    
    def replicate(self, key, value, version):
        """Handle replication from another master"""
        if key not in self.data:
            self.data[key] = {"value": value, "version": version}
        else:
            # Conflict resolution using version vectors
            if self.is_newer(version, self.data[key]["version"]):
                self.data[key] = {"value": value, "version": version}
            elif self.is_concurrent(version, self.data[key]["version"]):
                # Conflict! Keep both versions
                self.resolve_conflict(key, value, version)
```

**d) Health Checks and Auto-Healing**
```python
class HealthMonitor:
    def __init__(self, nodes, check_interval=5):
        self.nodes = nodes
        self.check_interval = check_interval
        self.unhealthy_nodes = set()
    
    def start_monitoring(self):
        while True:
            for node in self.nodes:
                if self.check_health(node):
                    if node in self.unhealthy_nodes:
                        # Node recovered
                        self.unhealthy_nodes.remove(node)
                        self.reintegrate_node(node)
                else:
                    if node not in self.unhealthy_nodes:
                        # Node failed
                        self.unhealthy_nodes.add(node)
                        self.handle_failure(node)
            
            time.sleep(self.check_interval)
    
    def check_health(self, node):
        try:
            response = requests.get(
                f"http://{node.address}/health",
                timeout=2
            )
            return response.status_code == 200
        except Exception:
            return False
    
    def handle_failure(self, node):
        # Remove from load balancer
        load_balancer.remove_node(node)
        
        # Spin up replacement
        new_node = orchestrator.create_node()
        self.nodes.append(new_node)
        
        # Replicate data to new node
        self.replicate_to_new_node(new_node)
```

---

#### 3. Achieving Partition Tolerance (P)

**Partition Tolerance**: System continues to operate despite network partitions.

**Techniques:**

**a) Gossip Protocol**
```python
class GossipNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers
        self.data = {}
        self.metadata = {}  # version info
    
    def gossip_round(self):
        """Periodically sync with random peers"""
        # Select random peer
        peer = random.choice(self.peers)
        
        try:
            # Exchange metadata
            my_metadata = self.get_metadata()
            peer_metadata = peer.get_metadata()
            
            # Identify differences
            my_missing = peer_metadata.keys() - my_metadata.keys()
            peer_missing = my_metadata.keys() - peer_metadata.keys()
            
            # Request missing data from peer
            for key in my_missing:
                self.data[key] = peer.get(key)
            
            # Send missing data to peer
            for key in peer_missing:
                peer.put(key, self.data[key])
            
            # Resolve conflicts for common keys
            common_keys = my_metadata.keys() & peer_metadata.keys()
            for key in common_keys:
                if peer_metadata[key]["version"] > my_metadata[key]["version"]:
                    self.data[key] = peer.get(key)
                elif peer_metadata[key]["version"] < my_metadata[key]["version"]:
                    peer.put(key, self.data[key])
        
        except NetworkException:
            # Partition detected, but continue operation
            pass
    
    def start_gossip(self, interval=1):
        while True:
            self.gossip_round()
            time.sleep(interval)
```

**b) Hinted Handoff**
```python
class HintedHandoffNode:
    def __init__(self, node_id):
        self.node_id = node_id
        self.data = {}
        self.hints = {}  # Temporary storage for unreachable nodes
    
    def write(self, key, value, target_nodes):
        successful_writes = []
        
        for node in target_nodes:
            try:
                node.write(key, value)
                successful_writes.append(node)
            except NetworkException:
                # Store hint locally
                self.store_hint(node.id, key, value)
        
        return len(successful_writes) > 0
    
    def store_hint(self, target_node_id, key, value):
        """Store write temporarily for unreachable node"""
        if target_node_id not in self.hints:
            self.hints[target_node_id] = []
        
        self.hints[target_node_id].append({
            "key": key,
            "value": value,
            "timestamp": time.time()
        })
    
    def replay_hints(self):
        """Background process to replay hints when nodes recover"""
        while True:
            for target_node_id, hints in list(self.hints.items()):
                node = self.get_node(target_node_id)
                
                try:
                    # Try to reach the node
                    for hint in hints:
                        node.write(hint["key"], hint["value"])
                    
                    # Success! Remove hints
                    del self.hints[target_node_id]
                
                except NetworkException:
                    # Still unreachable, keep hints
                    pass
            
            time.sleep(10)
```

**c) Vector Clocks for Conflict Resolution**
```python
class VectorClock:
    def __init__(self, node_id):
        self.node_id = node_id
        self.clock = {}
    
    def increment(self):
        if self.node_id not in self.clock:
            self.clock[self.node_id] = 0
        self.clock[self.node_id] += 1
    
    def update(self, other_clock):
        for node, value in other_clock.items():
            self.clock[node] = max(self.clock.get(node, 0), value)
        self.increment()
    
    def happens_before(self, other):
        """Check if this clock happened before other"""
        return (all(self.clock.get(k, 0) <= other.clock.get(k, 0) 
                   for k in set(self.clock.keys()) | set(other.clock.keys())) and
                self.clock != other.clock)
    
    def is_concurrent(self, other):
        """Check if events are concurrent (conflict)"""
        return not (self.happens_before(other) or 
                   other.happens_before(self))

class PartitionTolerantStore:
    def write(self, key, value):
        # Create version with vector clock
        clock = VectorClock(self.node_id)
        
        if key in self.data:
            # Update existing clock
            clock.update(self.data[key]["clock"].clock)
        else:
            clock.increment()
        
        self.data[key] = {
            "value": value,
            "clock": clock,
            "timestamp": time.time()
        }
    
    def merge(self, key, remote_data):
        """Merge data from another partition"""
        if key not in self.data:
            self.data[key] = remote_data
        else:
            local = self.data[key]
            remote = remote_data
            
            if local["clock"].happens_before(remote["clock"]):
                # Remote is newer
                self.data[key] = remote
            elif remote["clock"].happens_before(local["clock"]):
                # Local is newer, keep it
                pass
            else:
                # Concurrent writes (conflict!)
                self.resolve_conflict(key, local, remote)
    
    def resolve_conflict(self, key, local, remote):
        """Application-specific conflict resolution"""
        # Strategy 1: Last-write-wins
        if remote["timestamp"] > local["timestamp"]:
            self.data[key] = remote
        
        # Strategy 2: Keep both versions
        self.data[key] = {
            "versions": [local, remote],
            "status": "conflict"
        }
        
        # Strategy 3: Merge values (for counters, sets, etc.)
        if isinstance(local["value"], int):
            self.data[key] = {
                "value": local["value"] + remote["value"],
                "clock": local["clock"]
            }
```

**d) Anti-Entropy Repair**
```python
class AntiEntropyService:
    def __init__(self, nodes):
        self.nodes = nodes
    
    def merkle_tree_sync(self, node1, node2):
        """Efficient sync using Merkle trees"""
        # Get Merkle tree from both nodes
        tree1 = node1.get_merkle_tree()
        tree2 = node2.get_merkle_tree()
        
        # Compare roots
        if tree1.root == tree2.root:
            return  # Already in sync
        
        # Find differences recursively
        differences = self.find_differences(tree1, tree2)
        
        # Sync only different keys
        for key in differences:
            data1 = node1.get(key)
            data2 = node2.get(key)
            
            if data1 is None:
                node1.put(key, data2)
            elif data2 is None:
                node2.put(key, data1)
            else:
                # Resolve conflict
                self.resolve_and_sync(node1, node2, key, data1, data2)
    
    def periodic_repair(self, interval=3600):
        """Run anti-entropy process periodically"""
        while True:
            # Compare all pairs of nodes
            for i in range(len(self.nodes)):
                for j in range(i + 1, len(self.nodes)):
                    try:
                        self.merkle_tree_sync(
                            self.nodes[i],
                            self.nodes[j]
                        )
                    except Exception as e:
                        logging.error(f"Sync failed: {e}")
            
            time.sleep(interval)
```

---

### CAP Trade-offs Summary

| Property | Techniques | Trade-offs |
|----------|-----------|------------|
| **Consistency** | Consensus algorithms (Raft/Paxos), Two-Phase Commit, Quorum reads/writes, Synchronous replication | ❌ Higher latency<br>❌ Lower availability during partitions<br>✅ Strong guarantees |
| **Availability** | Async replication, Multi-master, Eventually consistent writes, Health checks, Auto-healing | ❌ Stale reads possible<br>❌ Conflict resolution complexity<br>✅ Always responsive |
| **Partition Tolerance** | Gossip protocol, Hinted handoff, Vector clocks, Anti-entropy repair, Merkle trees | ❌ Increased complexity<br>❌ Storage overhead<br>✅ Works during network issues |

---

### Load Balancing

**Horizontal Scaling with Load Balancers**

```
                    Internet
                       ↓
              [Load Balancer]
               /      |      \
              /       |       \
         [Server1] [Server2] [Server3]
              \       |       /
               \      |      /
              [Shared Database]
```

**Load Balancing Algorithms:**

1. **Round Robin**: Sequential distribution
2. **Least Connections**: Route to server with fewest connections
3. **IP Hash**: Based on client IP address
4. **Weighted Round Robin**: Servers have different capacities
5. **Least Response Time**: Route to fastest server

### Caching Strategies

**Cache Hierarchy**

```
[Client] → [CDN Cache] → [Application Cache] 
          → [Database Cache] → [Database]
```

**Caching Patterns:**

1. **Cache-Aside (Lazy Loading)**
```python
def get_user(user_id):
    # Try cache first
    user = cache.get(f"user:{user_id}")
    if user:
        return user
    
    # Cache miss - fetch from DB
    user = db.get_user(user_id)
    cache.set(f"user:{user_id}", user, ttl=3600)
    return user
```

2. **Write-Through Cache**
```python
def update_user(user_id, data):
    # Update database
    db.update_user(user_id, data)
    # Update cache
    cache.set(f"user:{user_id}", data, ttl=3600)
```

3. **Write-Behind (Write-Back)**
```python
def update_user(user_id, data):
    # Update cache immediately
    cache.set(f"user:{user_id}", data)
    # Queue for async DB update
    queue.push({"action": "update", "user_id": user_id, "data": data})
```

4. **Refresh-Ahead**
```python
def get_user(user_id):
    user = cache.get(f"user:{user_id}")
    if cache.ttl(f"user:{user_id}") < 300:  # 5 minutes
        # Proactively refresh
        background_task.refresh_cache(user_id)
    return user
```

**Cache Eviction Policies:**
- **LRU (Least Recently Used)**: Remove least recently accessed
- **LFU (Least Frequently Used)**: Remove least frequently accessed
- **FIFO**: First In First Out
- **TTL (Time To Live)**: Expire after time period

### Database Sharding

**Horizontal Partitioning of Data**

```
User Data Distribution by User ID:

Shard 1 (0-999):      Shard 2 (1000-1999):
[Users 0-999]         [Users 1000-1999]

Shard 3 (2000-2999):  Shard 4 (3000-3999):
[Users 2000-2999]     [Users 3000-3999]
```

**Sharding Strategies:**

1. **Hash-Based Sharding**
```python
def get_shard(user_id, num_shards=4):
    shard_id = hash(user_id) % num_shards
    return f"shard_{shard_id}"
```

2. **Range-Based Sharding**
```python
def get_shard(user_id):
    if user_id < 1000:
        return "shard_1"
    elif user_id < 2000:
        return "shard_2"
    # ... etc
```

3. **Geography-Based Sharding**
```python
def get_shard(user_location):
    region_map = {
        "US-EAST": "shard_us_east",
        "US-WEST": "shard_us_west",
        "EU": "shard_eu",
        "ASIA": "shard_asia"
    }
    return region_map.get(user_location)
```

### Database Replication

**Master-Slave Replication**

```
        [Master DB]
       (Writes Only)
            ↓
    ┌───────┼───────┐
    ↓       ↓       ↓
[Slave1] [Slave2] [Slave3]
(Reads)  (Reads)  (Reads)
```

**Multi-Master Replication**

```
[Master 1] ⟷ [Master 2]
    ↓             ↓
[Slave 1]    [Slave 2]
```

---

## Scalability Patterns

### Horizontal vs Vertical Scaling

**Vertical Scaling (Scale Up)**
```
Before:              After:
[4GB RAM]    →      [16GB RAM]
[2 CPUs]            [8 CPUs]
```

**Horizontal Scaling (Scale Out)**
```
Before:              After:
[Server 1]    →     [Server 1] [Server 2] [Server 3]
```

### Microservices Architecture

```
                [API Gateway]
                      ↓
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
   [User Service] [Order Service] [Payment Service]
        ↓             ↓             ↓
     [User DB]     [Order DB]    [Payment DB]
```

**Benefits:**
- Independent deployment
- Technology flexibility
- Fault isolation
- Easier scaling

**Example Microservice Decomposition:**

```yaml
# User Service
Responsibilities:
  - User registration
  - Authentication
  - Profile management
Database: PostgreSQL
API Endpoints:
  - POST /api/users/register
  - POST /api/users/login
  - GET /api/users/{id}
  - PUT /api/users/{id}

# Order Service
Responsibilities:
  - Order creation
  - Order tracking
  - Order history
Database: MongoDB
API Endpoints:
  - POST /api/orders
  - GET /api/orders/{id}
  - GET /api/orders/user/{userId}
  
# Payment Service
Responsibilities:
  - Payment processing
  - Refunds
  - Payment history
Database: PostgreSQL
External APIs: Stripe, PayPal
API Endpoints:
  - POST /api/payments
  - POST /api/payments/refund
  - GET /api/payments/{id}
```

### Message Queues for Async Processing

```
[Web Server] → [Message Queue] → [Worker Processes]
                   (RabbitMQ/Kafka)
                        ↓
                   [Database]
```

**Example: Order Processing**

```python
# Producer (Web Server)
def create_order(order_data):
    order_id = generate_order_id()
    db.save_order(order_id, order_data, status="PENDING")
    
    # Send to queue for processing
    queue.publish("order.created", {
        "order_id": order_id,
        "data": order_data
    })
    
    return {"order_id": order_id, "status": "PENDING"}

# Consumer (Worker)
def process_order(message):
    order_id = message["order_id"]
    
    # Validate inventory
    if inventory_service.check_availability(order_id):
        # Process payment
        payment_result = payment_service.process(order_id)
        
        if payment_result.success:
            db.update_order(order_id, status="CONFIRMED")
            # Send confirmation email
            email_service.send_confirmation(order_id)
        else:
            db.update_order(order_id, status="FAILED")
    else:
        db.update_order(order_id, status="OUT_OF_STOCK")
```

### Content Delivery Network (CDN)

```
                    [Origin Server]
                          ↓
              [CDN Edge Servers Worldwide]
                 /        |        \
            [US-EAST]  [EU-WEST]  [ASIA-PACIFIC]
                ↓          ↓          ↓
            [Users]    [Users]    [Users]
```

**Benefits:**
- Reduced latency
- Lower bandwidth costs
- DDoS protection
- Better user experience

---

## Reliability Patterns

### Circuit Breaker Pattern

**What is a Circuit Breaker?**

A Circuit Breaker is a design pattern that prevents an application from repeatedly trying to execute an operation that's likely to fail. It acts like an electrical circuit breaker that "trips" when things go wrong.

**Why Use Circuit Breaker?**
- Prevents cascading failures in distributed systems
- Stops wasting resources on operations likely to fail
- Allows failing services time to recover
- Provides fallback behavior when services are down
- Improves system resilience and user experience

**How It Works:**

```
┌─────────────────────────────────────────────────┐
│            Circuit Breaker States               │
├─────────────────────────────────────────────────┤
│                                                 │
│   CLOSED (Normal Operation)                     │
│   ↓                                             │
│   - All requests pass through                   │
│   - Monitor for failures                        │
│   - Count consecutive failures                  │
│                                                 │
│   IF failures >= threshold                      │
│   ↓                                             │
│                                                 │
│   OPEN (Circuit Tripped)                        │
│   ↓                                             │
│   - Reject requests immediately                 │
│   - Return error or fallback response           │
│   - Don't call the failing service              │
│   - Wait for timeout period                     │
│                                                 │
│   AFTER timeout expires                         │
│   ↓                                             │
│                                                 │
│   HALF_OPEN (Testing)                           │
│   ↓                                             │
│   - Allow limited requests through              │
│   - Test if service recovered                   │
│                                                 │
│   IF success → CLOSED                           │
│   IF failure → OPEN                             │
│                                                 │
└─────────────────────────────────────────────────┘
```

**State Transitions:**

```
    CLOSED ──(failures >= threshold)──→ OPEN
       ↑                                  │
       │                                  │
       │                            (timeout)
       │                                  ↓
       └──(success)──── HALF_OPEN ←──────┘
                            │
                       (failure)
                            │
                            ↓
                          OPEN
```

**Real-World Analogy:**
Think of calling a restaurant:
- **CLOSED**: Phone line works, you can call anytime
- **OPEN**: Line is always busy (5 failed attempts), so you stop calling for 10 minutes
- **HALF_OPEN**: After 10 minutes, try calling once to see if it's fixed
- If it works → back to **CLOSED**, if busy → back to **OPEN** for another 10 minutes

**Implementation:**

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e
    
    def on_success(self):
        self.failures = 0
        self.state = "CLOSED"
    
    def on_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()
        if self.failures >= self.failure_threshold:
            self.state = "OPEN"

# Usage
payment_circuit = CircuitBreaker(failure_threshold=5)

def make_payment(amount):
    return payment_circuit.call(payment_service.charge, amount)
```

**Detailed Explanation:**

**1. Configuration Parameters:**
```python
failure_threshold = 5  # Number of failures before opening circuit
timeout = 60           # Seconds to wait before trying again (OPEN → HALF_OPEN)
```

**2. How Each State Works:**

**CLOSED State (Normal):**
```python
# Circuit is closed - requests go through
result = circuit_breaker.call(external_service.api_call)
# If it fails, increment failure counter
# If failures >= threshold (e.g., 5), switch to OPEN
```

**OPEN State (Tripped):**
```python
# Circuit is open - reject immediately without calling service
try:
    result = circuit_breaker.call(external_service.api_call)
except Exception as e:
    print("Circuit breaker is OPEN - service unavailable")
    # Return cached data or default response
    return get_fallback_response()
```

**HALF_OPEN State (Testing):**
```python
# After timeout, allow one test request
result = circuit_breaker.call(external_service.api_call)
# If success → reset counter, go to CLOSED
# If failure → go back to OPEN, wait another timeout period
```

**3. Complete Example with Fallback:**

```python
class SmartCircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60, success_threshold=2):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.success_threshold = success_threshold  # Successes needed in HALF_OPEN
        self.failures = 0
        self.successes = 0
        self.last_failure_time = None
        self.state = "CLOSED"
    
    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
                print("Circuit breaker: OPEN → HALF_OPEN (testing service)")
            else:
                raise CircuitOpenException("Service unavailable - circuit is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e
    
    def on_success(self):
        if self.state == "HALF_OPEN":
            self.successes += 1
            if self.successes >= self.success_threshold:
                print("Circuit breaker: HALF_OPEN → CLOSED (service recovered)")
                self.state = "CLOSED"
                self.failures = 0
                self.successes = 0
        else:
            self.failures = 0
    
    def on_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()
        self.successes = 0
        
        if self.failures >= self.failure_threshold:
            old_state = self.state
            self.state = "OPEN"
            print(f"Circuit breaker: {old_state} → OPEN (too many failures)")

# Real-world usage with fallback
class PaymentService:
    def __init__(self):
        self.circuit = SmartCircuitBreaker(
            failure_threshold=3,
            timeout=30,
            success_threshold=2
        )
        self.cache = {}
    
    def charge_customer(self, user_id, amount):
        try:
            # Try to call external payment API
            result = self.circuit.call(
                stripe_api.charge,
                user_id,
                amount
            )
            # Cache successful result
            self.cache[user_id] = result
            return result
            
        except CircuitOpenException:
            # Circuit is open - return fallback
            print(f"Payment service down, using fallback for user {user_id}")
            return {
                "status": "queued",
                "message": "Payment will be processed when service recovers",
                "user_id": user_id,
                "amount": amount
            }
        except Exception as e:
            # Other errors
            print(f"Payment failed: {e}")
            raise
```

**4. Example Scenario Timeline:**

```
Time    State       Failures    Action
────────────────────────────────────────────────────────
0s      CLOSED      0          Request 1 → Success ✓
2s      CLOSED      0          Request 2 → Success ✓
5s      CLOSED      0          Request 3 → Failure ✗ (count=1)
7s      CLOSED      1          Request 4 → Failure ✗ (count=2)
9s      CLOSED      2          Request 5 → Failure ✗ (count=3)
10s     OPEN        3          Request 6 → REJECTED (circuit tripped!)
12s     OPEN        3          Request 7 → REJECTED (no call made)
15s     OPEN        3          Request 8 → REJECTED (no call made)
40s     HALF_OPEN   3          Request 9 → Trying... Success ✓ (count=1)
42s     HALF_OPEN   3          Request 10 → Trying... Success ✓ (count=2)
43s     CLOSED      0          Circuit closed! Service recovered
45s     CLOSED      0          Request 11 → Success ✓ (normal operation)
```

**5. Benefits in Distributed Systems:**

```python
# Example: Microservices with Circuit Breakers
class OrderService:
    def __init__(self):
        # Each external service has its own circuit breaker
        self.inventory_circuit = CircuitBreaker(failure_threshold=5)
        self.payment_circuit = CircuitBreaker(failure_threshold=3)
        self.notification_circuit = CircuitBreaker(failure_threshold=10)
    
    def create_order(self, order_data):
        # Check inventory (critical - fail fast)
        try:
            inventory = self.inventory_circuit.call(
                inventory_service.check_stock,
                order_data.items
            )
        except CircuitOpenException:
            return {"error": "Inventory service unavailable"}
        
        # Process payment (critical - fail fast)
        try:
            payment = self.payment_circuit.call(
                payment_service.charge,
                order_data.user_id,
                order_data.total
            )
        except CircuitOpenException:
            return {"error": "Payment service unavailable"}
        
        # Send notification (non-critical - degrade gracefully)
        try:
            self.notification_circuit.call(
                notification_service.send_email,
                order_data.user_email
            )
        except CircuitOpenException:
            # Queue for later, don't fail the order
            notification_queue.enqueue(order_data)
        
        return {"status": "success", "order_id": order_id}
```

**6. Monitoring Circuit Breakers:**

```python
class MonitoredCircuitBreaker(CircuitBreaker):
    def __init__(self, name, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.name = name
        self.metrics = {
            "total_requests": 0,
            "successful_requests": 0,
            "failed_requests": 0,
            "rejected_requests": 0,
            "state_changes": []
        }
    
    def call(self, func, *args, **kwargs):
        self.metrics["total_requests"] += 1
        
        if self.state == "OPEN":
            self.metrics["rejected_requests"] += 1
        
        try:
            result = super().call(func, *args, **kwargs)
            self.metrics["successful_requests"] += 1
            return result
        except Exception as e:
            self.metrics["failed_requests"] += 1
            raise e
    
    def on_state_change(self, old_state, new_state):
        self.metrics["state_changes"].append({
            "from": old_state,
            "to": new_state,
            "timestamp": time.time()
        })
        # Send alert
        monitoring.alert(f"Circuit breaker '{self.name}': {old_state} → {new_state}")
    
    def get_health_metrics(self):
        success_rate = (
            self.metrics["successful_requests"] / 
            max(1, self.metrics["total_requests"])
        ) * 100
        
        return {
            "name": self.name,
            "state": self.state,
            "success_rate": f"{success_rate:.2f}%",
            "failures": self.failures,
            "total_requests": self.metrics["total_requests"],
            "rejected_requests": self.metrics["rejected_requests"]
        }
```

**Key Takeaways:**

| Aspect | Description |
|--------|-------------|
| **Purpose** | Prevent cascading failures by failing fast |
| **When to Use** | External API calls, database queries, microservice communication |
| **Failure Threshold** | Typically 3-10 failures (tune based on service reliability) |
| **Timeout** | 30-120 seconds (time for service to recover) |
| **Fallback** | Cached data, default values, or queued operations |
| **Monitoring** | Track state changes, success rates, rejection counts |

---

### Rate Limiting

**Token Bucket Algorithm**

```python
import time
from collections import defaultdict

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.tokens = defaultdict(lambda: capacity)
        self.last_refill = defaultdict(lambda: time.time())
    
    def allow_request(self, user_id):
        current_time = time.time()
        
        # Refill tokens
        time_passed = current_time - self.last_refill[user_id]
        tokens_to_add = time_passed * self.refill_rate
        self.tokens[user_id] = min(
            self.capacity,
            self.tokens[user_id] + tokens_to_add
        )
        self.last_refill[user_id] = current_time
        
        # Check if request allowed
        if self.tokens[user_id] >= 1:
            self.tokens[user_id] -= 1
            return True
        return False

# Usage: 100 requests per minute per user
rate_limiter = TokenBucket(capacity=100, refill_rate=100/60)

def api_endpoint(user_id):
    if not rate_limiter.allow_request(user_id):
        return {"error": "Rate limit exceeded"}, 429
    
    # Process request
    return handle_request()
```

### Retry with Exponential Backoff

```python
import time
import random

def retry_with_backoff(func, max_retries=5, base_delay=1):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise e
            
            # Exponential backoff with jitter
            delay = base_delay * (2 ** attempt)
            jitter = random.uniform(0, delay * 0.1)
            time.sleep(delay + jitter)
            
            print(f"Retry attempt {attempt + 1} after {delay:.2f}s")

# Usage
def call_external_api():
    return retry_with_backoff(lambda: requests.get("https://api.example.com"))
```

### Health Checks and Heartbeats

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/health', methods=['GET'])
def health_check():
    checks = {
        "database": check_database(),
        "cache": check_cache(),
        "queue": check_message_queue(),
        "external_api": check_external_services()
    }
    
    all_healthy = all(checks.values())
    status_code = 200 if all_healthy else 503
    
    return jsonify({
        "status": "healthy" if all_healthy else "unhealthy",
        "checks": checks,
        "timestamp": time.time()
    }), status_code

def check_database():
    try:
        db.execute("SELECT 1")
        return True
    except:
        return False

def check_cache():
    try:
        cache.ping()
        return True
    except:
        return False
```

---

## Real-World System Designs

### 1. WhatsApp Messaging System

**Requirements:**
- 2 billion users
- 100 billion messages/day
- Real-time delivery
- Message persistence
- End-to-end encryption

**High-Level Architecture:**

```
[Mobile Clients]
      ↓
[Load Balancer]
      ↓
[WebSocket Servers] ← [Presence Service]
      ↓                      ↓
[Message Queue]          [Redis Cache]
      ↓                      ↓
[Message Processor]    [User Status DB]
      ↓
[Message Storage]
(Cassandra/HBase)
```

**Key Components:**

```yaml
WebSocket Servers:
  Purpose: Maintain persistent connections
  Technology: Node.js, WebSockets
  Scaling: Horizontal with load balancing
  
Message Queue:
  Purpose: Decouple message sending/receiving
  Technology: Kafka
  Partitions: By user_id
  
Message Storage:
  Purpose: Store message history
  Technology: Cassandra
  Schema:
    - conversation_id (partition key)
    - timestamp (clustering key)
    - message_id
    - sender_id
    - content (encrypted)
    - status (sent/delivered/read)

Presence Service:
  Purpose: Track online/offline status
  Technology: Redis
  Data Structure: 
    - key: user:{user_id}:status
    - value: {status: "online", last_seen: timestamp}
    - TTL: 5 minutes

Media Storage:
  Purpose: Store images, videos, files
  Technology: S3/Azure Blob Storage
  CDN: CloudFront/Akamai
```

**Message Flow:**

```python
# Sending a message
def send_message(sender_id, receiver_id, content):
    message = {
        "message_id": generate_uuid(),
        "sender_id": sender_id,
        "receiver_id": receiver_id,
        "content": encrypt(content),
        "timestamp": current_timestamp(),
        "status": "sent"
    }
    
    # Store in database
    cassandra.insert("messages", message)
    
    # Check if receiver is online
    receiver_connection = connection_manager.get(receiver_id)
    
    if receiver_connection:
        # Send directly via WebSocket
        receiver_connection.send(message)
        update_status(message_id, "delivered")
    else:
        # Queue for later delivery
        push_notification_service.send(receiver_id, "New message")
        
    return message_id

# Receiving acknowledgment
def message_delivered(message_id, receiver_id):
    update_status(message_id, "delivered")
    notify_sender(message_id, "delivered")

def message_read(message_id, receiver_id):
    update_status(message_id, "read")
    notify_sender(message_id, "read")
```

### 2. Uber Ride Dispatch System

**Requirements:**
- Match riders with nearby drivers
- Real-time location tracking
- ETA calculation
- Pricing algorithm
- High availability

**High-Level Architecture:**

```
[Mobile Apps (Riders/Drivers)]
            ↓
      [API Gateway]
            ↓
    ┌───────┼───────┐
    ↓       ↓       ↓
[Location] [Matching] [Payment]
[Service]  [Service]  [Service]
    ↓       ↓       ↓
[Geo DB] [Redis]  [Payment DB]
```

**Key Components:**

```yaml
Location Service:
  Purpose: Track real-time locations
  Technology: 
    - Redis with Geospatial indexes
    - GEOADD to add locations
    - GEORADIUS to find nearby drivers
  Update Frequency: Every 4 seconds
  
  Commands:
    # Add driver location
    GEOADD drivers:active {longitude} {latitude} {driver_id}
    
    # Find drivers within 5km
    GEORADIUS drivers:active {long} {lat} 5 km WITHDIST

Matching Service:
  Algorithm:
    1. Find drivers within radius (start 1km, expand)
    2. Filter by driver preferences
    3. Calculate ETA for each driver
    4. Score based on: distance, ETA, rating
    5. Send request to top 3 drivers
    6. First to accept gets the ride
    
Pricing Service:
  Factors:
    - Base fare
    - Distance
    - Time
    - Demand (surge pricing)
    - Traffic conditions
    
  Formula:
    price = base_fare + (distance * per_km) + 
            (time * per_minute) * surge_multiplier

Surge Pricing:
  Purpose: Balance supply and demand
  Algorithm:
    supply = active_drivers_in_area
    demand = active_ride_requests_in_area
    if demand / supply > threshold:
        surge_multiplier = calculate_surge(demand, supply)
```

**Ride Matching Algorithm:**

```python
class RideMatchingService:
    def find_drivers(self, rider_location, radius_km=5):
        # Find nearby drivers using geospatial query
        nearby_drivers = redis.georadius(
            "drivers:active",
            latitude=rider_location.lat,
            longitude=rider_location.lon,
            radius=radius_km,
            unit="km",
            withdist=True
        )
        
        # Filter available drivers
        available = []
        for driver_id, distance in nearby_drivers:
            driver_status = redis.get(f"driver:{driver_id}:status")
            if driver_status == "available":
                available.append({
                    "driver_id": driver_id,
                    "distance": distance,
                    "rating": get_driver_rating(driver_id),
                    "eta": calculate_eta(driver_location, rider_location)
                })
        
        # Sort by score
        return sorted(available, key=lambda d: self.score_driver(d))
    
    def score_driver(self, driver):
        # Lower is better
        distance_score = driver["distance"] * 2
        eta_score = driver["eta"]
        rating_score = (5 - driver["rating"]) * 10
        return distance_score + eta_score + rating_score
    
    def request_ride(self, rider_id, pickup_location):
        drivers = self.find_drivers(pickup_location)
        
        # Send request to top 3 drivers
        for driver in drivers[:3]:
            notification_service.send_ride_request(
                driver["driver_id"],
                {
                    "ride_id": generate_ride_id(),
                    "pickup": pickup_location,
                    "rider_rating": get_rider_rating(rider_id),
                    "estimated_price": calculate_price(pickup, destination)
                }
            )
            
        # Wait for acceptance (with timeout)
        return wait_for_acceptance(ride_id, timeout=30)
```

### 3. Netflix Streaming Service

**Requirements:**
- 200+ million subscribers
- Stream video content globally
- Multiple quality levels (360p to 4K)
- Resume playback
- Personalized recommendations

**High-Level Architecture:**

```
[Users]
   ↓
[CDN (Content Delivery Network)]
   ↓
[API Gateway]
   ↓
┌──────┼──────────┼──────────┐
↓      ↓          ↓          ↓
[User] [Video]  [Recommendation] [Playback]
[Service] [Service] [Service]    [Service]
   ↓      ↓          ↓          ↓
[User DB] [S3]  [ML Models]  [Redis]
```

**Key Components:**

```yaml
Video Encoding:
  Purpose: Convert source to multiple formats
  Process:
    1. Upload source video
    2. Create multiple bitrates:
       - 4K (3840x2160) - 25 Mbps
       - 1080p - 8 Mbps
       - 720p - 5 Mbps
       - 480p - 2.5 Mbps
       - 360p - 1 Mbps
    3. Segment into chunks (10 seconds each)
    4. Store in S3
    5. Distribute to CDN

Adaptive Bitrate Streaming:
  Technology: HLS (HTTP Live Streaming) or DASH
  Manifest File (m3u8):
    #EXTM3U
    #EXT-X-STREAM-INF:BANDWIDTH=8000000,RESOLUTION=1920x1080
    1080p/playlist.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1280x720
    720p/playlist.m3u8

CDN Strategy:
  - Use multiple CDN providers (multi-CDN)
  - Open Connect (Netflix's own CDN)
  - Cache popular content at edge
  - Predict what to pre-cache based on:
    * Geographic popularity
    * Time of day
    * New releases

Recommendation Engine:
  Algorithms:
    - Collaborative Filtering
    - Content-Based Filtering
    - Deep Learning Models
  Data Points:
    - Watch history
    - Ratings
    - Search queries
    - Time of day
    - Device type
    - Completion rate

Playback Service:
  Responsibilities:
    - Track watch position
    - Resume playback
    - Skip intro/credits
    - Next episode auto-play
  
  Storage: Redis
  Schema:
    key: playback:{user_id}:{video_id}
    value: {
      position: seconds,
      timestamp: datetime,
      quality: "1080p",
      device: "smart_tv"
    }
```

**Video Streaming Flow:**

```python
class VideoStreamingService:
    def start_playback(self, user_id, video_id):
        # Check subscription
        if not subscription_service.is_active(user_id):
            return {"error": "Subscription required"}
        
        # Get resume position
        position = redis.get(f"playback:{user_id}:{video_id}")
        
        # Get video manifest
        manifest = self.get_video_manifest(video_id)
        
        # Log playback event
        analytics.log_event({
            "event": "playback_started",
            "user_id": user_id,
            "video_id": video_id,
            "timestamp": current_time(),
            "device": request.device
        })
        
        return {
            "manifest_url": manifest.url,
            "resume_position": position or 0,
            "subtitles": get_subtitles(video_id, user_language),
            "cdn_url": get_nearest_cdn(user_location)
        }
    
    def update_playback_position(self, user_id, video_id, position):
        # Update every 10 seconds
        redis.setex(
            f"playback:{user_id}:{video_id}",
            2592000,  # 30 days TTL
            position
        )
        
        # Check for completion
        if position > video_duration * 0.9:
            self.mark_as_watched(user_id, video_id)
            recommendation_service.update_preferences(user_id, video_id)
    
    def get_video_manifest(self, video_id):
        # Generate adaptive bitrate manifest
        video_metadata = db.get_video(video_id)
        
        manifest = {
            "video_id": video_id,
            "variants": []
        }
        
        for quality in ["4k", "1080p", "720p", "480p", "360p"]:
            manifest["variants"].append({
                "quality": quality,
                "bandwidth": quality_to_bandwidth[quality],
                "url": f"https://cdn.netflix.com/{video_id}/{quality}/playlist.m3u8"
            })
        
        return manifest
```

### 4. Instagram News Feed

**Requirements:**
- Generate personalized feed
- Mix of following + recommended content
- Low latency (<200ms)
- Handle millions of concurrent users
- Real-time updates

**High-Level Architecture:**

```
[Mobile Apps]
      ↓
[API Gateway]
      ↓
[Feed Generation Service]
      ↓
┌─────┴─────┬─────────┐
↓           ↓         ↓
[Post]   [Graph]  [Ranking]
[Service] [Service] [Service]
↓           ↓         ↓
[Post DB] [Graph DB] [ML Models]
```

**Feed Generation Strategy:**

```yaml
Approach: Hybrid (Pull + Push)

Push Model (Fanout on Write):
  When user posts:
    1. Get follower list
    2. Insert post into each follower's feed cache
  Pros: Fast reads
  Cons: Slow writes for celebrities
  
  Use for: Regular users (<10k followers)

Pull Model (Fanout on Read):
  When user requests feed:
    1. Get list of following
    2. Query recent posts from each
    3. Merge and rank
  Pros: No fanout overhead
  Cons: Slower reads
  
  Use for: Celebrities (>10k followers)

Hybrid Approach:
  - Push for regular users
  - Pull for celebrities
  - Cache results
  - Pre-compute for active users
```

**Implementation:**

```python
class FeedGenerationService:
    def generate_feed(self, user_id, page=1, page_size=20):
        # Check cache first
        cached_feed = redis.get(f"feed:{user_id}:{page}")
        if cached_feed:
            return json.loads(cached_feed)
        
        # Get following list
        following = graph_service.get_following(user_id)
        
        # Separate celebrities from regular users
        celebrities = [u for u in following if u.follower_count > 10000]
        regular_users = [u for u in following if u.follower_count <= 10000]
        
        # Pull from cache for regular users (already fanned out)
        regular_posts = self.get_cached_posts(regular_users)
        
        # Pull in real-time for celebrities
        celebrity_posts = self.fetch_recent_posts(celebrities, limit=50)
        
        # Add recommended posts
        recommended = recommendation_service.get_posts(user_id, limit=10)
        
        # Combine all posts
        all_posts = regular_posts + celebrity_posts + recommended
        
        # Rank posts
        ranked_feed = self.rank_posts(user_id, all_posts)
        
        # Paginate
        result = ranked_feed[(page-1)*page_size : page*page_size]
        
        # Cache result
        redis.setex(f"feed:{user_id}:{page}", 300, json.dumps(result))
        
        return result
    
    def rank_posts(self, user_id, posts):
        # Ranking factors
        for post in posts:
            score = 0
            
            # Recency (decay over time)
            age_hours = (now() - post.created_at).hours
            recency_score = 1 / (1 + age_hours/24)
            
            # Engagement
            engagement_score = (
                post.likes * 1 +
                post.comments * 2 +
                post.shares * 3
            ) / (age_hours + 1)
            
            # User affinity (interaction history)
            affinity = self.get_user_affinity(user_id, post.author_id)
            
            # Content type preference
            content_pref = self.get_content_preference(user_id, post.type)
            
            # Combined score
            post.score = (
                recency_score * 0.3 +
                engagement_score * 0.4 +
                affinity * 0.2 +
                content_pref * 0.1
            )
        
        return sorted(posts, key=lambda p: p.score, reverse=True)
    
    def fanout_post(self, post_id, author_id):
        # Get followers
        followers = graph_service.get_followers(author_id)
        
        # For users with <10k followers, fanout immediately
        if len(followers) < 10000:
            for follower_id in followers:
                # Add to follower's feed cache
                redis.lpush(f"feed:cache:{follower_id}", post_id)
                redis.ltrim(f"feed:cache:{follower_id}", 0, 499)  # Keep last 500
                
                # Send push notification if enabled
                if user_preferences.push_enabled(follower_id):
                    push_service.send(follower_id, f"New post from {author_id}")
        else:
            # For celebrities, just cache the post
            redis.setex(f"post:{post_id}", 86400, json.dumps(post_data))
```

**Graph Service (Followers/Following):**

```python
class GraphService:
    def follow(self, follower_id, followee_id):
        # Add to graph database
        graph_db.create_relationship(
            follower_id,
            "FOLLOWS",
            followee_id,
            {"created_at": current_time()}
        )
        
        # Update counters
        redis.incr(f"user:{follower_id}:following_count")
        redis.incr(f"user:{followee_id}:follower_count")
        
        # Invalidate feed cache
        redis.delete(f"feed:{follower_id}:*")
    
    def get_followers(self, user_id):
        # Check cache
        cached = redis.get(f"followers:{user_id}")
        if cached:
            return json.loads(cached)
        
        # Query graph
        followers = graph_db.query(f"""
            MATCH (follower)-[:FOLLOWS]->(user:{user_id})
            RETURN follower.id
        """)
        
        # Cache result
        redis.setex(f"followers:{user_id}", 3600, json.dumps(followers))
        return followers
```

---

## Architecture Patterns

### Event-Driven Architecture

```
[Service A] → [Event Bus] → [Service B]
                  ↓
              [Service C]
```

**Example: E-commerce Order Flow**

```python
# Order Service publishes event
def create_order(order_data):
    order = db.save_order(order_data)
    
    event_bus.publish("order.created", {
        "order_id": order.id,
        "user_id": order.user_id,
        "items": order.items,
        "total": order.total
    })
    
    return order

# Inventory Service listens
@event_bus.subscribe("order.created")
def reserve_inventory(event):
    order_id = event["order_id"]
    items = event["items"]
    
    for item in items:
        inventory.reserve(item.id, item.quantity)
    
    event_bus.publish("inventory.reserved", {
        "order_id": order_id,
        "status": "success"
    })

# Payment Service listens
@event_bus.subscribe("inventory.reserved")
def process_payment(event):
    order_id = event["order_id"]
    payment_result = payment_gateway.charge(order_id)
    
    if payment_result.success:
        event_bus.publish("payment.completed", {
            "order_id": order_id
        })
    else:
        event_bus.publish("payment.failed", {
            "order_id": order_id
        })

# Notification Service listens
@event_bus.subscribe("payment.completed")
def send_confirmation(event):
    order = db.get_order(event["order_id"])
    email_service.send_confirmation(order.user_email, order)
    sms_service.send_notification(order.user_phone)
```

### CQRS (Command Query Responsibility Segregation)

```
Commands (Write):        Queries (Read):
[Write API]             [Read API]
    ↓                       ↑
[Write DB]  → [Events] → [Read DB]
(PostgreSQL)             (MongoDB/Elasticsearch)
```

**Example Implementation:**

```python
# Write Model (Commands)
class OrderCommandService:
    def create_order(self, command: CreateOrderCommand):
        order = Order(
            id=generate_id(),
            user_id=command.user_id,
            items=command.items,
            status="PENDING"
        )
        
        # Save to write database
        write_db.save(order)
        
        # Publish event
        event_store.append(OrderCreatedEvent(
            order_id=order.id,
            data=order.to_dict()
        ))
        
        return order.id

# Read Model (Queries)
class OrderQueryService:
    def get_order(self, order_id):
        # Read from optimized read database
        return read_db.orders.find_one({"_id": order_id})
    
    def get_user_orders(self, user_id, filters):
        # Complex queries on read model
        return read_db.orders.find({
            "user_id": user_id,
            "status": {"$in": filters.statuses},
            "created_at": {"$gte": filters.start_date}
        }).sort("created_at", -1)

# Event Handler (Sync read model)
@event_handler("OrderCreated")
def sync_order_to_read_model(event):
    order_data = event.data
    
    # Update read model with denormalized data
    read_db.orders.insert_one({
        "_id": order_data["id"],
        "user_id": order_data["user_id"],
        "user_name": user_service.get_name(order_data["user_id"]),
        "items": order_data["items"],
        "total": calculate_total(order_data["items"]),
        "status": order_data["status"],
        "created_at": order_data["created_at"]
    })
```

---

## Best Practices

### 1. Design for Failure
- Assume components will fail
- Implement timeouts and retries
- Use circuit breakers
- Design for graceful degradation

### 2. Scalability Principles
- Horizontal scaling over vertical
- Stateless components
- Database sharding
- Caching at multiple levels

### 3. Monitoring and Observability
- Log everything important
- Track key metrics
- Set up alerts
- Use distributed tracing

### 4. Security
- Defense in depth
- Encryption in transit and at rest
- Regular security audits
- Principle of least privilege

### 5. Cost Optimization
- Right-size resources
- Use auto-scaling
- Implement caching
- Monitor and optimize

This guide provides a comprehensive foundation for designing high-level systems. Each pattern and example can be adapted based on specific requirements and constraints.
