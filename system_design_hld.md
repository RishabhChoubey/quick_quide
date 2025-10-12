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
