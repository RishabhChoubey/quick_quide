# Messaging & Event-Driven Systems Guide

## Table of Contents

1. [Message Queue Fundamentals](#message-queue-fundamentals)
2. [Apache Kafka](#apache-kafka)
3. [RabbitMQ](#rabbitmq)
4. [Pub-Sub Pattern](#pub-sub-pattern)
5. [Polling vs Webhooks](#polling-vs-webhooks)
6. [Event-Driven Architecture](#event-driven-architecture)
7. [Message Patterns](#message-patterns)
8. [Best Practices](#best-practices)

---

## Message Queue Fundamentals

### Why Use Message Queues?

**Benefits:**
- **Decoupling**: Services don't need to know about each other
- **Scalability**: Handle traffic spikes with buffering
- **Reliability**: Messages persist until processed
- **Asynchronous Processing**: Non-blocking operations
- **Load Balancing**: Distribute work across consumers

### Basic Concepts

```
Producer → Queue → Consumer
```

**Key Components:**
- **Producer**: Sends messages
- **Queue/Topic**: Stores messages
- **Consumer**: Receives and processes messages
- **Broker**: Manages queues and routing

---

## Apache Kafka

### Architecture

```
Topic: orders
├── Partition 0: [msg1, msg2, msg3, ...]
├── Partition 1: [msg4, msg5, msg6, ...]
└── Partition 2: [msg7, msg8, msg9, ...]

Producer → Broker → Consumer Groups
```

### Basic Kafka Producer (Python)

```python
from kafka import KafkaProducer
import json
from datetime import datetime

class OrderProducer:
    def __init__(self, bootstrap_servers=['localhost:9092']):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            acks='all',  # Wait for all replicas
            retries=3,
            max_in_flight_requests_per_connection=1  # Ensure ordering
        )
    
    def send_order(self, order_data):
        """Send order to Kafka topic"""
        message = {
            'order_id': order_data['order_id'],
            'customer_id': order_data['customer_id'],
            'total': order_data['total'],
            'items': order_data['items'],
            'timestamp': datetime.now().isoformat()
        }
        
        # Send to topic 'orders' with key for partitioning
        future = self.producer.send(
            'orders',
            key=str(order_data['customer_id']).encode('utf-8'),
            value=message
        )
        
        # Block until message is sent (optional)
        record_metadata = future.get(timeout=10)
        
        print(f"Message sent to topic: {record_metadata.topic}")
        print(f"Partition: {record_metadata.partition}")
        print(f"Offset: {record_metadata.offset}")
        
        return record_metadata
    
    def close(self):
        self.producer.flush()
        self.producer.close()

# Usage
producer = OrderProducer()
producer.send_order({
    'order_id': 'ORD-123',
    'customer_id': 'CUST-456',
    'total': 99.99,
    'items': [{'product': 'Laptop', 'qty': 1}]
})
producer.close()
```

### Kafka Consumer

```python
from kafka import KafkaConsumer
import json

class OrderConsumer:
    def __init__(self, bootstrap_servers=['localhost:9092'], 
                 group_id='order-processing-group'):
        self.consumer = KafkaConsumer(
            'orders',
            bootstrap_servers=bootstrap_servers,
            group_id=group_id,
            auto_offset_reset='earliest',  # Start from beginning
            enable_auto_commit=False,  # Manual commit for reliability
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )
    
    def process_orders(self):
        """Process orders from Kafka"""
        try:
            for message in self.consumer:
                order = message.value
                
                print(f"Processing order: {order['order_id']}")
                print(f"Topic: {message.topic}")
                print(f"Partition: {message.partition}")
                print(f"Offset: {message.offset}")
                
                # Process order
                success = self.process_order(order)
                
                if success:
                    # Commit offset only after successful processing
                    self.consumer.commit()
                    print(f"Order {order['order_id']} processed successfully")
                else:
                    # Don't commit - message will be reprocessed
                    print(f"Failed to process order {order['order_id']}")
                    
        except KeyboardInterrupt:
            print("Shutting down consumer...")
        finally:
            self.consumer.close()
    
    def process_order(self, order):
        """Business logic to process order"""
        try:
            # Validate order
            if order['total'] <= 0:
                return False
            
            # Process payment
            payment_success = self.process_payment(order)
            
            # Update inventory
            inventory_success = self.update_inventory(order)
            
            return payment_success and inventory_success
        except Exception as e:
            print(f"Error processing order: {str(e)}")
            return False
    
    def process_payment(self, order):
        # Payment processing logic
        print(f"Processing payment: ${order['total']}")
        return True
    
    def update_inventory(self, order):
        # Inventory update logic
        print(f"Updating inventory for {len(order['items'])} items")
        return True

# Usage
consumer = OrderConsumer(group_id='order-processor-1')
consumer.process_orders()
```

### Consumer Groups for Scalability

```python
# Start multiple consumers in the same group for parallel processing

# Terminal 1
consumer1 = OrderConsumer(group_id='order-processing-group')
consumer1.process_orders()

# Terminal 2
consumer2 = OrderConsumer(group_id='order-processing-group')
consumer2.process_orders()

# Terminal 3
consumer3 = OrderConsumer(group_id='order-processing-group')
consumer3.process_orders()

# Kafka automatically distributes partitions among consumers
# Each partition is consumed by only one consumer in the group
```

### Java Kafka Producer

```java
import org.apache.kafka.clients.producer.*;
import java.util.Properties;

public class KafkaOrderProducer {
    private Producer<String, String> producer;
    
    public KafkaOrderProducer() {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("key.serializer", 
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", 
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put("acks", "all");
        props.put("retries", 3);
        
        producer = new KafkaProducer<>(props);
    }
    
    public void sendOrder(String orderId, String orderJson) {
        ProducerRecord<String, String> record = 
            new ProducerRecord<>("orders", orderId, orderJson);
        
        producer.send(record, new Callback() {
            @Override
            public void onCompletion(RecordMetadata metadata, Exception e) {
                if (e != null) {
                    System.err.println("Error sending message: " + e.getMessage());
                } else {
                    System.out.println("Message sent to partition " + 
                                     metadata.partition() + 
                                     " with offset " + metadata.offset());
                }
            }
        });
    }
    
    public void close() {
        producer.close();
    }
}
```

### Kafka Streams (Real-time Processing)

```java
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.kstream.*;

public class OrderStreamProcessor {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "order-processor");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        
        StreamsBuilder builder = new StreamsBuilder();
        
        // Input stream
        KStream<String, String> orders = builder.stream("orders");
        
        // Filter high-value orders
        KStream<String, String> highValueOrders = orders.filter(
            (key, value) -> parseOrderValue(value) > 1000
        );
        
        // Send to different topic
        highValueOrders.to("high-value-orders");
        
        // Aggregate by customer
        KTable<String, Long> orderCountByCustomer = orders
            .groupByKey()
            .count();
        
        // Start processing
        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
        
        // Shutdown hook
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
    
    private static double parseOrderValue(String orderJson) {
        // Parse JSON and extract total
        return 0.0;
    }
}
```

---

## RabbitMQ

### Architecture

```
Producer → Exchange → Queue → Consumer
```

**Exchange Types:**
- **Direct**: Route by exact key match
- **Fanout**: Broadcast to all queues
- **Topic**: Route by pattern matching
- **Headers**: Route by message headers

### Python RabbitMQ Producer

```python
import pika
import json

class RabbitMQProducer:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host=host)
        )
        self.channel = self.connection.channel()
        
        # Declare exchange
        self.channel.exchange_declare(
            exchange='orders',
            exchange_type='topic',
            durable=True
        )
    
    def send_order(self, order_data, routing_key='order.created'):
        """Send order to RabbitMQ"""
        message = json.dumps(order_data)
        
        self.channel.basic_publish(
            exchange='orders',
            routing_key=routing_key,
            body=message,
            properties=pika.BasicProperties(
                delivery_mode=2,  # Make message persistent
                content_type='application/json'
            )
        )
        
        print(f"Sent order {order_data['order_id']} with routing key {routing_key}")
    
    def close(self):
        self.connection.close()

# Usage
producer = RabbitMQProducer()

# Send different types of orders
producer.send_order({
    'order_id': 'ORD-123',
    'type': 'standard',
    'total': 50.00
}, routing_key='order.created.standard')

producer.send_order({
    'order_id': 'ORD-124',
    'type': 'express',
    'total': 150.00
}, routing_key='order.created.express')

producer.close()
```

### RabbitMQ Consumer

```python
import pika
import json

class RabbitMQConsumer:
    def __init__(self, host='localhost', queue_name='order_processing'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host=host)
        )
        self.channel = self.connection.channel()
        
        # Declare exchange
        self.channel.exchange_declare(
            exchange='orders',
            exchange_type='topic',
            durable=True
        )
        
        # Declare queue
        self.channel.queue_declare(
            queue=queue_name,
            durable=True
        )
        
        # Bind queue to exchange with routing pattern
        self.channel.queue_bind(
            exchange='orders',
            queue=queue_name,
            routing_key='order.created.*'  # Match all order types
        )
        
        # Set QoS - process one message at a time
        self.channel.basic_qos(prefetch_count=1)
    
    def callback(self, ch, method, properties, body):
        """Process received message"""
        order = json.loads(body)
        
        print(f"Received order: {order['order_id']}")
        print(f"Routing key: {method.routing_key}")
        
        try:
            # Process order
            self.process_order(order)
            
            # Acknowledge message
            ch.basic_ack(delivery_tag=method.delivery_tag)
            print(f"Order {order['order_id']} processed successfully")
            
        except Exception as e:
            print(f"Error processing order: {str(e)}")
            # Reject and requeue message
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
    
    def process_order(self, order):
        # Business logic
        if order['type'] == 'express':
            print("Priority processing for express order")
        else:
            print("Standard processing")
    
    def start_consuming(self):
        """Start consuming messages"""
        self.channel.basic_consume(
            queue='order_processing',
            on_message_callback=self.callback,
            auto_ack=False  # Manual acknowledgment
        )
        
        print("Waiting for messages. Press CTRL+C to exit")
        try:
            self.channel.start_consuming()
        except KeyboardInterrupt:
            self.channel.stop_consuming()
        finally:
            self.connection.close()

# Usage
consumer = RabbitMQConsumer()
consumer.start_consuming()
```

### Work Queues (Load Distribution)

```python
# Multiple workers consuming from same queue

# Worker 1
consumer1 = RabbitMQConsumer(queue_name='work_queue')
consumer1.start_consuming()

# Worker 2
consumer2 = RabbitMQConsumer(queue_name='work_queue')
consumer2.start_consuming()

# RabbitMQ uses round-robin to distribute messages
```

### Dead Letter Queue (DLQ)

```python
class RabbitMQWithDLQ:
    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()
        
        # Declare DLQ
        self.channel.queue_declare(queue='orders_dlq', durable=True)
        
        # Declare main queue with DLQ configuration
        self.channel.queue_declare(
            queue='orders',
            durable=True,
            arguments={
                'x-dead-letter-exchange': '',
                'x-dead-letter-routing-key': 'orders_dlq',
                'x-message-ttl': 60000,  # 60 seconds
                'x-max-retries': 3
            }
        )
    
    def callback(self, ch, method, properties, body):
        try:
            order = json.loads(body)
            self.process_order(order)
            ch.basic_ack(delivery_tag=method.delivery_tag)
        except Exception as e:
            # After max retries, message goes to DLQ
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
```

---

## Pub-Sub Pattern

### Redis Pub-Sub

```python
import redis

class RedisPubSub:
    def __init__(self, host='localhost', port=6379):
        self.redis_client = redis.Redis(host=host, port=port, decode_responses=True)
    
    def publish(self, channel, message):
        """Publish message to channel"""
        self.redis_client.publish(channel, message)
        print(f"Published to {channel}: {message}")
    
    def subscribe(self, channels, callback):
        """Subscribe to channels"""
        pubsub = self.redis_client.pubsub()
        pubsub.subscribe(**{channel: callback for channel in channels})
        
        print(f"Subscribed to channels: {channels}")
        
        # Listen for messages
        for message in pubsub.listen():
            if message['type'] == 'message':
                print(f"Received on {message['channel']}: {message['data']}")

# Publisher
publisher = RedisPubSub()
publisher.publish('notifications', 'New order received!')

# Subscriber
def handle_notification(message):
    print(f"Handling: {message['data']}")

subscriber = RedisPubSub()
subscriber.subscribe(['notifications', 'alerts'], handle_notification)
```

### AWS SNS + SQS (Fan-out Pattern)

```python
import boto3
import json

class SNSFanOut:
    def __init__(self):
        self.sns = boto3.client('sns')
        self.sqs = boto3.client('sqs')
    
    def create_topic(self, topic_name):
        """Create SNS topic"""
        response = self.sns.create_topic(Name=topic_name)
        return response['TopicArn']
    
    def subscribe_queue(self, topic_arn, queue_url):
        """Subscribe SQS queue to SNS topic"""
        queue_arn = self.sqs.get_queue_attributes(
            QueueUrl=queue_url,
            AttributeNames=['QueueArn']
        )['Attributes']['QueueArn']
        
        self.sns.subscribe(
            TopicArn=topic_arn,
            Protocol='sqs',
            Endpoint=queue_arn
        )
    
    def publish_event(self, topic_arn, event_data):
        """Publish event to topic"""
        self.sns.publish(
            TopicArn=topic_arn,
            Message=json.dumps(event_data),
            Subject='Order Event'
        )
        print(f"Published event to topic {topic_arn}")

# Setup
sns_fanout = SNSFanOut()

# Create topic
topic_arn = sns_fanout.create_topic('order-events')

# Subscribe multiple queues (different services)
sns_fanout.subscribe_queue(topic_arn, email_queue_url)
sns_fanout.subscribe_queue(topic_arn, analytics_queue_url)
sns_fanout.subscribe_queue(topic_arn, inventory_queue_url)

# Publish once, all services receive
sns_fanout.publish_event(topic_arn, {
    'event': 'order.created',
    'order_id': 'ORD-123',
    'customer_id': 'CUST-456'
})
```

---

## Polling vs Webhooks

### Polling (Pull Model)

**Client repeatedly checks for new data**

```python
import time
import requests

class PollingClient:
    def __init__(self, api_url, interval=5):
        self.api_url = api_url
        self.interval = interval
        self.last_checked_id = 0
    
    def poll(self):
        """Poll API for new orders"""
        while True:
            try:
                response = requests.get(
                    f"{self.api_url}/orders",
                    params={'since_id': self.last_checked_id}
                )
                
                if response.status_code == 200:
                    orders = response.json()
                    
                    if orders:
                        print(f"Found {len(orders)} new orders")
                        for order in orders:
                            self.process_order(order)
                            self.last_checked_id = max(self.last_checked_id, 
                                                      order['id'])
                    else:
                        print("No new orders")
                else:
                    print(f"Error: {response.status_code}")
                
            except Exception as e:
                print(f"Polling error: {str(e)}")
            
            # Wait before next poll
            time.sleep(self.interval)
    
    def process_order(self, order):
        print(f"Processing order: {order['id']}")

# Usage
client = PollingClient('https://api.example.com', interval=10)
client.poll()

# Pros:
# - Simple to implement
# - Client controls timing
# - Works with any server

# Cons:
# - Inefficient (many empty responses)
# - Higher latency
# - Wastes resources
```

### Webhooks (Push Model)

**Server notifies client when events occur**

```python
from flask import Flask, request, jsonify
import hmac
import hashlib

app = Flask(__name__)

class WebhookServer:
    def __init__(self, secret_key):
        self.secret_key = secret_key
    
    def verify_signature(self, payload, signature):
        """Verify webhook signature for security"""
        expected_signature = hmac.new(
            self.secret_key.encode(),
            payload.encode(),
            hashlib.sha256
        ).hexdigest()
        
        return hmac.compare_digest(signature, expected_signature)
    
    @app.route('/webhook/orders', methods=['POST'])
    def handle_order_webhook(self):
        """Receive order webhook"""
        # Get signature from header
        signature = request.headers.get('X-Webhook-Signature')
        payload = request.get_data(as_text=True)
        
        # Verify signature
        if not self.verify_signature(payload, signature):
            return jsonify({'error': 'Invalid signature'}), 401
        
        # Process webhook
        data = request.json
        event_type = data.get('event')
        order = data.get('order')
        
        print(f"Received webhook: {event_type}")
        
        if event_type == 'order.created':
            self.handle_order_created(order)
        elif event_type == 'order.updated':
            self.handle_order_updated(order)
        
        # Return 200 to acknowledge receipt
        return jsonify({'status': 'success'}), 200
    
    def handle_order_created(self, order):
        print(f"New order: {order['id']}")
        # Process order immediately
    
    def handle_order_updated(self, order):
        print(f"Order updated: {order['id']}")

# Start webhook server
webhook = WebhookServer(secret_key='your-secret-key')
app.run(port=5000)

# Pros:
# - Real-time notifications
# - Efficient (no unnecessary requests)
# - Lower latency

# Cons:
# - Requires public endpoint
# - Need to handle retries
# - More complex security
```

### Webhook Sender (Service that sends webhooks)

```python
import requests
import hmac
import hashlib
import json

class WebhookSender:
    def __init__(self, webhook_urls, secret_key):
        self.webhook_urls = webhook_urls
        self.secret_key = secret_key
    
    def send_webhook(self, event_type, data, max_retries=3):
        """Send webhook to registered URLs"""
        payload = json.dumps({
            'event': event_type,
            'data': data,
            'timestamp': datetime.now().isoformat()
        })
        
        # Generate signature
        signature = hmac.new(
            self.secret_key.encode(),
            payload.encode(),
            hashlib.sha256
        ).hexdigest()
        
        headers = {
            'Content-Type': 'application/json',
            'X-Webhook-Signature': signature
        }
        
        # Send to all registered webhooks
        for url in self.webhook_urls:
            retry_count = 0
            while retry_count < max_retries:
                try:
                    response = requests.post(
                        url,
                        data=payload,
                        headers=headers,
                        timeout=10
                    )
                    
                    if response.status_code == 200:
                        print(f"Webhook sent successfully to {url}")
                        break
                    else:
                        print(f"Webhook failed with status {response.status_code}")
                        retry_count += 1
                        
                except Exception as e:
                    print(f"Error sending webhook: {str(e)}")
                    retry_count += 1
                    time.sleep(2 ** retry_count)  # Exponential backoff
            
            if retry_count == max_retries:
                print(f"Failed to send webhook to {url} after {max_retries} retries")

# Usage
sender = WebhookSender(
    webhook_urls=[
        'https://client1.example.com/webhook',
        'https://client2.example.com/webhook'
    ],
    secret_key='shared-secret'
)

# Send webhook when order created
sender.send_webhook('order.created', {
    'order_id': 'ORD-123',
    'customer_id': 'CUST-456',
    'total': 99.99
})
```

---

## Event-Driven Architecture

### Event Bus Implementation

```python
from typing import Callable, Dict, List
import json

class EventBus:
    def __init__(self):
        self.subscribers: Dict[str, List[Callable]] = {}
    
    def subscribe(self, event_type: str, handler: Callable):
        """Subscribe to an event type"""
        if event_type not in self.subscribers:
            self.subscribers[event_type] = []
        self.subscribers[event_type].append(handler)
        print(f"Subscribed to {event_type}")
    
    def publish(self, event_type: str, data: dict):
        """Publish an event"""
        print(f"Publishing event: {event_type}")
        
        if event_type in self.subscribers:
            for handler in self.subscribers[event_type]:
                try:
                    handler(data)
                except Exception as e:
                    print(f"Error in handler: {str(e)}")
    
    def unsubscribe(self, event_type: str, handler: Callable):
        """Unsubscribe from an event type"""
        if event_type in self.subscribers:
            self.subscribers[event_type].remove(handler)

# Create event bus
event_bus = EventBus()

# Services subscribe to events
class EmailService:
    def __init__(self, event_bus):
        event_bus.subscribe('order.created', self.send_confirmation)
        event_bus.subscribe('order.shipped', self.send_shipping_notification)
    
    def send_confirmation(self, order_data):
        print(f"Sending email confirmation for order {order_data['order_id']}")
    
    def send_shipping_notification(self, order_data):
        print(f"Sending shipping notification for order {order_data['order_id']}")

class InventoryService:
    def __init__(self, event_bus):
        event_bus.subscribe('order.created', self.reserve_inventory)
    
    def reserve_inventory(self, order_data):
        print(f"Reserving inventory for order {order_data['order_id']}")

class AnalyticsService:
    def __init__(self, event_bus):
        event_bus.subscribe('order.created', self.track_order)
        event_bus.subscribe('order.cancelled', self.track_cancellation)
    
    def track_order(self, order_data):
        print(f"Tracking order {order_data['order_id']} in analytics")
    
    def track_cancellation(self, order_data):
        print(f"Tracking cancellation of order {order_data['order_id']}")

# Initialize services
email_service = EmailService(event_bus)
inventory_service = InventoryService(event_bus)
analytics_service = AnalyticsService(event_bus)

# Publish event
event_bus.publish('order.created', {
    'order_id': 'ORD-123',
    'customer_id': 'CUST-456',
    'total': 99.99
})
# All subscribed services receive the event
```

This comprehensive guide covers all major messaging and event-driven patterns used in modern distributed systems!
