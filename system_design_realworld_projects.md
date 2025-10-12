# Real-World System Design Projects

## Table of Contents

1. [Design Netflix / Video Streaming Platform](#design-netflix--video-streaming-platform)
2. [Design WhatsApp / Real-time Chat System](#design-whatsapp--real-time-chat-system)
3. [Design Amazon / E-commerce Platform](#design-amazon--e-commerce-platform)
4. [Design Uber / Ride-Sharing Platform](#design-uber--ride-sharing-platform)
5. [Design Twitter / Social Media Feed](#design-twitter--social-media-feed)
6. [Design TinyURL / URL Shortener](#design-tinyurl--url-shortener)
7. [Design System Design Interview Tips](#system-design-interview-tips)

---

## Design Netflix / Video Streaming Platform

### Requirements

**Functional:**
- Upload videos
- Stream videos with adaptive bitrate
- Search videos
- Recommendations
- User profiles and watch history

**Non-Functional:**
- 200M users, 100M daily active
- Low latency video streaming
- High availability (99.99%)
- Global distribution

### Architecture

```
┌─────────────┐
│   Client    │ (Web/Mobile/TV)
└──────┬──────┘
       │
       ├──────────────────────┐
       │                      │
┌──────▼──────┐      ┌────────▼────────┐
│  CDN Edge   │      │   API Gateway   │
│  (CloudFront)│      │                 │
└─────────────┘      └────────┬────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────▼────────┐  ┌────────▼────────┐  ┌────────▼────────┐
│ Video Service  │  │ User Service    │  │ Search Service  │
└───────┬────────┘  └────────┬────────┘  └────────┬────────┘
        │                    │                     │
        │                    │                     │
┌───────▼────────┐  ┌────────▼────────┐  ┌────────▼────────┐
│   S3/Blob      │  │  PostgreSQL     │  │  Elasticsearch  │
│   Storage      │  │  (User Data)    │  │                 │
└────────────────┘  └─────────────────┘  └─────────────────┘
        │
        ▼
┌────────────────┐
│ Encoding Queue │ (Kafka)
│ & Transcoding  │
└────────────────┘
```

### Components

#### 1. Video Upload & Processing

```python
import boto3
import uuid
from kafka import KafkaProducer
import json

class VideoUploadService:
    def __init__(self):
        self.s3 = boto3.client('s3')
        self.producer = KafkaProducer(
            bootstrap_servers=['kafka:9092'],
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )
        self.bucket = 'netflix-raw-videos'
    
    def upload_video(self, video_file, user_id, metadata):
        """Upload raw video and trigger processing"""
        video_id = str(uuid.uuid4())
        
        # Upload to S3
        key = f"raw/{user_id}/{video_id}/{video_file.filename}"
        self.s3.upload_fileobj(video_file, self.bucket, key)
        
        # Trigger transcoding job
        job = {
            'video_id': video_id,
            'user_id': user_id,
            's3_key': key,
            'metadata': metadata,
            'output_formats': [
                {'resolution': '4K', 'bitrate': '15Mbps'},
                {'resolution': '1080p', 'bitrate': '5Mbps'},
                {'resolution': '720p', 'bitrate': '2.5Mbps'},
                {'resolution': '480p', 'bitrate': '1Mbps'},
                {'resolution': '360p', 'bitrate': '500Kbps'}
            ]
        }
        
        self.producer.send('video-encoding-jobs', value=job)
        
        return video_id

class VideoTranscodingWorker:
    def __init__(self):
        self.s3 = boto3.client('s3')
        self.consumer = KafkaConsumer(
            'video-encoding-jobs',
            bootstrap_servers=['kafka:9092']
        )
    
    def process_jobs(self):
        """Process transcoding jobs"""
        for message in self.consumer:
            job = json.loads(message.value)
            
            # Download from S3
            video_file = self.download_video(job['s3_key'])
            
            # Transcode to multiple formats
            for format_spec in job['output_formats']:
                output_file = self.transcode(
                    video_file,
                    resolution=format_spec['resolution'],
                    bitrate=format_spec['bitrate']
                )
                
                # Upload to S3
                output_key = f"encoded/{job['video_id']}/{format_spec['resolution']}/playlist.m3u8"
                self.s3.upload_file(output_file, 'netflix-encoded-videos', output_key)
            
            # Update database
            self.mark_video_ready(job['video_id'])
    
    def transcode(self, input_file, resolution, bitrate):
        """Transcode video using FFmpeg"""
        # FFmpeg command for HLS (HTTP Live Streaming)
        import subprocess
        
        output_file = f"/tmp/output_{resolution}.m3u8"
        
        cmd = [
            'ffmpeg',
            '-i', input_file,
            '-vf', f'scale=-2:{resolution.replace("p", "")}',
            '-b:v', bitrate,
            '-c:v', 'libx264',
            '-c:a', 'aac',
            '-hls_time', '10',
            '-hls_playlist_type', 'vod',
            '-f', 'hls',
            output_file
        ]
        
        subprocess.run(cmd, check=True)
        return output_file
```

#### 2. Adaptive Bitrate Streaming (HLS)

```python
class StreamingService:
    def __init__(self):
        self.cdn_url = "https://cdn.netflix.com"
    
    def get_master_playlist(self, video_id):
        """Generate master playlist for adaptive streaming"""
        playlist = "#EXTM3U\n"
        playlist += "#EXT-X-VERSION:3\n\n"
        
        # Different quality levels
        qualities = [
            {'resolution': '1920x1080', 'bandwidth': 5000000, 'name': '1080p'},
            {'resolution': '1280x720', 'bandwidth': 2500000, 'name': '720p'},
            {'resolution': '854x480', 'bandwidth': 1000000, 'name': '480p'},
            {'resolution': '640x360', 'bandwidth': 500000, 'name': '360p'}
        ]
        
        for quality in qualities:
            playlist += f"#EXT-X-STREAM-INF:BANDWIDTH={quality['bandwidth']},RESOLUTION={quality['resolution']}\n"
            playlist += f"{self.cdn_url}/videos/{video_id}/{quality['name']}/playlist.m3u8\n\n"
        
        return playlist
```

#### 3. Recommendation System

```python
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

class RecommendationEngine:
    def __init__(self):
        self.user_profiles = {}
        self.video_features = {}
    
    def update_user_profile(self, user_id, watched_videos, ratings):
        """Build user profile based on watch history"""
        # Calculate average feature vector
        feature_vectors = [self.video_features[v_id] for v_id in watched_videos]
        weights = ratings / np.sum(ratings)
        
        self.user_profiles[user_id] = np.average(feature_vectors, weights=weights, axis=0)
    
    def get_recommendations(self, user_id, n=10):
        """Get video recommendations using collaborative filtering"""
        if user_id not in self.user_profiles:
            return self.get_trending_videos(n)
        
        user_profile = self.user_profiles[user_id]
        
        # Calculate similarity scores
        scores = {}
        for video_id, features in self.video_features.items():
            similarity = cosine_similarity([user_profile], [features])[0][0]
            scores[video_id] = similarity
        
        # Sort and return top N
        recommended = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        return [video_id for video_id, score in recommended[:n]]
    
    def get_trending_videos(self, n=10):
        """Get trending videos for new users"""
        # Query from cache or database
        return self.redis.zrevrange('trending_videos', 0, n-1)
```

---

## Design WhatsApp / Real-time Chat System

### Requirements

**Functional:**
- One-on-one messaging
- Group chat
- Message history
- Online status
- Read receipts
- Media sharing

**Non-Functional:**
- 2B users, 100M concurrent connections
- Real-time delivery (<100ms)
- High availability
- End-to-end encryption

### Architecture

```
┌──────────────┐
│    Client    │ (Mobile/Web)
└──────┬───────┘
       │
       │ WebSocket
       │
┌──────▼───────────────┐
│  Load Balancer       │
│  (WebSocket support) │
└──────┬───────────────┘
       │
┌──────▼───────────────┐
│  Chat Servers        │ (Stateful)
│  (WebSocket)         │
└──────┬───────────────┘
       │
       ├────────────────────┬──────────────────┐
       │                    │                  │
┌──────▼──────┐   ┌────────▼────────┐  ┌─────▼──────┐
│  Message    │   │  User Service   │  │  Presence  │
│  Queue      │   │                 │  │  Service   │
│  (Kafka)    │   └─────────────────┘  └────────────┘
└──────┬──────┘
       │
┌──────▼──────┐
│  Cassandra  │ (Message Storage)
│  (Partitioned│
│  by user_id) │
└─────────────┘
```

### Components

#### 1. WebSocket Connection Manager

```python
from flask_socketio import SocketIO, emit, join_room, leave_room
import redis

class ConnectionManager:
    def __init__(self):
        self.socketio = SocketIO()
        self.redis = redis.Redis()
        self.connections = {}  # user_id -> socket_id mapping
    
    def on_connect(self, user_id, socket_id):
        """Handle user connection"""
        # Store connection
        self.connections[user_id] = socket_id
        
        # Update presence
        self.redis.hset('user_presence', user_id, 'online')
        self.redis.setex(f'user:{user_id}:heartbeat', 30, 'alive')
        
        # Notify contacts
        contacts = self.get_user_contacts(user_id)
        for contact_id in contacts:
            self.emit_to_user(contact_id, 'user_online', {'user_id': user_id})
        
        # Join user's personal room
        join_room(user_id)
    
    def on_disconnect(self, user_id):
        """Handle user disconnect"""
        if user_id in self.connections:
            del self.connections[user_id]
        
        # Update presence
        self.redis.hset('user_presence', user_id, 'offline')
        
        # Notify contacts
        contacts = self.get_user_contacts(user_id)
        for contact_id in contacts:
            self.emit_to_user(contact_id, 'user_offline', {'user_id': user_id})
    
    def emit_to_user(self, user_id, event, data):
        """Send message to specific user"""
        if user_id in self.connections:
            emit(event, data, room=self.connections[user_id])
        else:
            # User offline - queue message
            self.queue_offline_message(user_id, event, data)

connection_manager = ConnectionManager()

@socketio.on('connect')
def handle_connect(auth):
    user_id = authenticate(auth['token'])
    connection_manager.on_connect(user_id, request.sid)

@socketio.on('disconnect')
def handle_disconnect():
    user_id = get_current_user()
    connection_manager.on_disconnect(user_id)
```

#### 2. Message Service

```python
from kafka import KafkaProducer, KafkaConsumer
from cassandra.cluster import Cluster
import uuid
from datetime import datetime

class MessageService:
    def __init__(self):
        self.producer = KafkaProducer(bootstrap_servers=['kafka:9092'])
        self.cassandra = Cluster(['cassandra:9042']).connect('whatsapp')
        self.redis = redis.Redis()
    
    def send_message(self, sender_id, recipient_id, content, message_type='text'):
        """Send message"""
        message_id = str(uuid.uuid4())
        timestamp = datetime.utcnow()
        
        message = {
            'message_id': message_id,
            'sender_id': sender_id,
            'recipient_id': recipient_id,
            'content': content,
            'type': message_type,
            'timestamp': timestamp.isoformat(),
            'status': 'sent'
        }
        
        # Store in Cassandra
        self.store_message(message)
        
        # Send to Kafka for delivery
        self.producer.send('messages', value=json.dumps(message).encode())
        
        # Try immediate delivery if recipient online
        if self.is_user_online(recipient_id):
            connection_manager.emit_to_user(
                recipient_id,
                'new_message',
                message
            )
        
        return message_id
    
    def store_message(self, message):
        """Store message in Cassandra"""
        query = """
        INSERT INTO messages (message_id, sender_id, recipient_id, content, 
                            message_type, timestamp, status)
        VALUES (?, ?, ?, ?, ?, ?, ?)
        """
        
        self.cassandra.execute(query, (
            uuid.UUID(message['message_id']),
            message['sender_id'],
            message['recipient_id'],
            message['content'],
            message['type'],
            datetime.fromisoformat(message['timestamp']),
            message['status']
        ))
        
        # Also store in recipient's inbox (for efficient retrieval)
        inbox_query = """
        INSERT INTO user_messages (user_id, message_id, sender_id, 
                                  content, timestamp)
        VALUES (?, ?, ?, ?, ?)
        """
        
        self.cassandra.execute(inbox_query, (
            message['recipient_id'],
            uuid.UUID(message['message_id']),
            message['sender_id'],
            message['content'],
            datetime.fromisoformat(message['timestamp'])
        ))
    
    def get_conversation(self, user1_id, user2_id, limit=50):
        """Get conversation history"""
        # Check cache first
        cache_key = f"conversation:{min(user1_id, user2_id)}:{max(user1_id, user2_id)}"
        cached = self.redis.lrange(cache_key, 0, limit-1)
        
        if cached:
            return [json.loads(msg) for msg in cached]
        
        # Query from Cassandra
        query = """
        SELECT * FROM messages
        WHERE (sender_id = ? AND recipient_id = ?)
           OR (sender_id = ? AND recipient_id = ?)
        ORDER BY timestamp DESC
        LIMIT ?
        """
        
        rows = self.cassandra.execute(query, (user1_id, user2_id, user2_id, user1_id, limit))
        
        messages = [self._row_to_dict(row) for row in rows]
        
        # Cache for future requests
        for msg in messages:
            self.redis.lpush(cache_key, json.dumps(msg))
        self.redis.expire(cache_key, 3600)
        
        return messages
    
    def mark_as_read(self, message_id, user_id):
        """Mark message as read"""
        # Update in Cassandra
        query = """
        UPDATE messages
        SET status = 'read', read_at = ?
        WHERE message_id = ?
        """
        
        self.cassandra.execute(query, (datetime.utcnow(), uuid.UUID(message_id)))
        
        # Send read receipt
        message = self.get_message(message_id)
        connection_manager.emit_to_user(
            message['sender_id'],
            'message_read',
            {'message_id': message_id, 'reader_id': user_id}
        )
```

#### 3. Group Chat

```python
class GroupChatService:
    def __init__(self):
        self.cassandra = Cluster(['cassandra:9042']).connect('whatsapp')
        self.redis = redis.Redis()
    
    def create_group(self, creator_id, member_ids, group_name):
        """Create group chat"""
        group_id = str(uuid.uuid4())
        
        # Store group info
        query = """
        INSERT INTO groups (group_id, name, creator_id, created_at)
        VALUES (?, ?, ?, ?)
        """
        
        self.cassandra.execute(query, (
            uuid.UUID(group_id),
            group_name,
            creator_id,
            datetime.utcnow()
        ))
        
        # Add members
        for member_id in member_ids:
            self.add_group_member(group_id, member_id)
        
        return group_id
    
    def send_group_message(self, sender_id, group_id, content):
        """Send message to group"""
        message_id = str(uuid.uuid4())
        
        message = {
            'message_id': message_id,
            'group_id': group_id,
            'sender_id': sender_id,
            'content': content,
            'timestamp': datetime.utcnow().isoformat()
        }
        
        # Store message
        self.store_group_message(message)
        
        # Get group members
        members = self.get_group_members(group_id)
        
        # Send to all members (except sender)
        for member_id in members:
            if member_id != sender_id:
                connection_manager.emit_to_user(
                    member_id,
                    'group_message',
                    message
                )
        
        return message_id
```

### Cassandra Schema

```sql
-- Messages table (partitioned by user for efficient retrieval)
CREATE TABLE user_messages (
    user_id TEXT,
    message_id UUID,
    sender_id TEXT,
    content TEXT,
    message_type TEXT,
    timestamp TIMESTAMP,
    status TEXT,
    PRIMARY KEY (user_id, timestamp, message_id)
) WITH CLUSTERING ORDER BY (timestamp DESC);

-- Groups table
CREATE TABLE groups (
    group_id UUID PRIMARY KEY,
    name TEXT,
    creator_id TEXT,
    created_at TIMESTAMP
);

-- Group members
CREATE TABLE group_members (
    group_id UUID,
    user_id TEXT,
    joined_at TIMESTAMP,
    PRIMARY KEY (group_id, user_id)
);

-- Group messages
CREATE TABLE group_messages (
    group_id UUID,
    message_id UUID,
    sender_id TEXT,
    content TEXT,
    timestamp TIMESTAMP,
    PRIMARY KEY (group_id, timestamp, message_id)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

---

## Design Amazon / E-commerce Platform

### Requirements

**Functional:**
- Product catalog & search
- Shopping cart
- Order placement
- Payment processing
- Inventory management
- Order tracking

**Non-Functional:**
- 300M users, 50M daily active
- Handle traffic spikes (Black Friday)
- High availability (99.99%)
- Consistency for inventory and payments

### Architecture

```
┌────────────┐
│   Client   │
└─────┬──────┘
      │
┌─────▼──────────────┐
│   CDN + WAF        │
└─────┬──────────────┘
      │
┌─────▼──────────────┐
│  API Gateway       │
│  (Rate Limiting)   │
└─────┬──────────────┘
      │
      ├──────────────┬──────────────┬──────────────┐
      │              │              │              │
┌─────▼──────┐ ┌────▼────┐  ┌─────▼──────┐ ┌────▼────────┐
│  Product   │ │  Cart   │  │   Order    │ │  Payment    │
│  Service   │ │ Service │  │  Service   │ │  Service    │
└─────┬──────┘ └────┬────┘  └─────┬──────┘ └────┬────────┘
      │             │              │             │
┌─────▼──────┐ ┌────▼────┐  ┌─────▼──────┐ ┌────▼────────┐
│Elasticsearch│ │  Redis  │  │ PostgreSQL │ │  Stripe     │
└────────────┘ └─────────┘  └────────────┘ └─────────────┘
```

### Components

#### 1. Product Catalog Service

```python
from elasticsearch import Elasticsearch
import redis

class ProductCatalogService:
    def __init__(self):
        self.es = Elasticsearch(['elasticsearch:9200'])
        self.redis = redis.Redis()
        self.cache_ttl = 3600
    
    def search_products(self, query, filters=None, page=1, size=20):
        """Search products with filters"""
        must_clauses = [
            {
                'multi_match': {
                    'query': query,
                    'fields': ['name^3', 'description', 'brand^2', 'category'],
                    'fuzziness': 'AUTO'
                }
            }
        ]
        
        # Apply filters
        filter_clauses = []
        if filters:
            if 'category' in filters:
                filter_clauses.append({'term': {'category': filters['category']}})
            
            if 'price_range' in filters:
                filter_clauses.append({
                    'range': {
                        'price': {
                            'gte': filters['price_range']['min'],
                            'lte': filters['price_range']['max']
                        }
                    }
                })
            
            if 'brand' in filters:
                filter_clauses.append({'terms': {'brand': filters['brand']}})
        
        # Construct query
        search_body = {
            'query': {
                'bool': {
                    'must': must_clauses,
                    'filter': filter_clauses
                }
            },
            'from': (page - 1) * size,
            'size': size,
            'sort': [{'_score': 'desc'}, {'popularity': 'desc'}]
        }
        
        results = self.es.search(index='products', body=search_body)
        
        return {
            'products': [hit['_source'] for hit in results['hits']['hits']],
            'total': results['hits']['total']['value'],
            'page': page
        }
    
    def get_product(self, product_id):
        """Get product details with caching"""
        # Check cache
        cache_key = f"product:{product_id}"
        cached = self.redis.get(cache_key)
        
        if cached:
            return json.loads(cached)
        
        # Query from Elasticsearch
        result = self.es.get(index='products', id=product_id)
        product = result['_source']
        
        # Cache result
        self.redis.setex(cache_key, self.cache_ttl, json.dumps(product))
        
        return product
```

#### 2. Shopping Cart Service

```python
class ShoppingCartService:
    def __init__(self):
        self.redis = redis.Redis()
    
    def add_to_cart(self, user_id, product_id, quantity=1):
        """Add item to cart"""
        cart_key = f"cart:{user_id}"
        
        # Get current cart
        cart_data = self.redis.hget(cart_key, product_id)
        
        if cart_data:
            current_qty = int(cart_data)
            new_qty = current_qty + quantity
        else:
            new_qty = quantity
        
        # Update cart
        self.redis.hset(cart_key, product_id, new_qty)
        self.redis.expire(cart_key, 86400 * 7)  # 7 days expiry
        
        return self.get_cart(user_id)
    
    def get_cart(self, user_id):
        """Get user's cart"""
        cart_key = f"cart:{user_id}"
        cart_items = self.redis.hgetall(cart_key)
        
        if not cart_items:
            return {'items': [], 'total': 0}
        
        # Fetch product details
        products = []
        total = 0
        
        for product_id, quantity in cart_items.items():
            product = self.get_product(product_id.decode())
            quantity = int(quantity)
            
            item = {
                'product_id': product_id.decode(),
                'name': product['name'],
                'price': product['price'],
                'quantity': quantity,
                'subtotal': product['price'] * quantity
            }
            
            products.append(item)
            total += item['subtotal']
        
        return {
            'items': products,
            'total': total
        }
```

#### 3. Order Service (with Saga pattern)

```python
class OrderService:
    def __init__(self):
        self.db = get_database_connection()
        self.inventory_service = InventoryService()
        self.payment_service = PaymentService()
    
    def create_order(self, user_id, cart_items, payment_method):
        """Create order using Saga pattern"""
        order_id = str(uuid.uuid4())
        
        try:
            # Step 1: Create order record
            order = self._create_order_record(order_id, user_id, cart_items)
            
            # Step 2: Reserve inventory
            reservation = self.inventory_service.reserve_items(cart_items)
            
            # Step 3: Process payment
            payment = self.payment_service.charge(
                user_id,
                order['total'],
                payment_method
            )
            
            # Step 4: Confirm order
            self._confirm_order(order_id)
            
            # Step 5: Clear cart
            self._clear_cart(user_id)
            
            return {
                'order_id': order_id,
                'status': 'confirmed',
                'total': order['total']
            }
            
        except Exception as e:
            # Compensating transactions
            self._handle_order_failure(order_id, e)
            raise
    
    def _handle_order_failure(self, order_id, error):
        """Rollback order on failure"""
        try:
            # Release inventory
            self.inventory_service.release_reservation(order_id)
            
            # Refund payment (if charged)
            self.payment_service.refund(order_id)
            
            # Cancel order
            self._cancel_order(order_id, reason=str(error))
            
        except Exception as rollback_error:
            # Log for manual intervention
            logger.error(f"Rollback failed for order {order_id}: {rollback_error}")
```

#### 4. Inventory Management

```python
class InventoryService:
    def __init__(self):
        self.db = get_database_connection()
        self.redis = redis.Redis()
    
    def check_availability(self, product_id, quantity):
        """Check if product is available"""
        # Use Redis for real-time inventory
        cache_key = f"inventory:{product_id}"
        available = self.redis.get(cache_key)
        
        if available is None:
            # Fetch from database
            available = self._get_inventory_from_db(product_id)
            self.redis.setex(cache_key, 300, available)
        else:
            available = int(available)
        
        return available >= quantity
    
    def reserve_items(self, cart_items):
        """Reserve inventory for order"""
        reservation_id = str(uuid.uuid4())
        
        for item in cart_items:
            product_id = item['product_id']
            quantity = item['quantity']
            
            # Atomic decrement in Redis
            cache_key = f"inventory:{product_id}"
            new_qty = self.redis.decrby(cache_key, quantity)
            
            if new_qty < 0:
                # Rollback
                self.redis.incrby(cache_key, quantity)
                raise OutOfStockError(f"Product {product_id} out of stock")
            
            # Record reservation
            self._record_reservation(reservation_id, product_id, quantity)
        
        return reservation_id
    
    def _record_reservation(self, reservation_id, product_id, quantity):
        """Record inventory reservation"""
        query = """
        INSERT INTO inventory_reservations 
        (reservation_id, product_id, quantity, expires_at)
        VALUES (?, ?, ?, NOW() + INTERVAL '15 minutes')
        """
        
        self.db.execute(query, (reservation_id, product_id, quantity))
```

This comprehensive guide provides complete architectures and implementation details for real-world system design projects, perfect for interviews and practical applications!
