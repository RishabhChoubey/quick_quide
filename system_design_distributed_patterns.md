# Advanced Distributed Patterns Guide

## Table of Contents

1. [Saga Pattern](#saga-pattern)
2. [Orchestration vs Choreography](#orchestration-vs-choreography)
3. [Anti-Corruption Layer](#anti-corruption-layer)
4. [Backend for Frontend (BFF)](#backend-for-frontend-bff)
5. [Strangler Fig Pattern](#strangler-fig-pattern)
6. [CQRS (Command Query Responsibility Segregation)](#cqrs-command-query-responsibility-segregation)
7. [Event Sourcing](#event-sourcing)
8. [Two-Phase Commit (2PC)](#two-phase-commit-2pc)
9. [Outbox Pattern](#outbox-pattern)
10. [API Composition](#api-composition)

---

## Saga Pattern

**Problem**: Maintaining data consistency across microservices without distributed transactions.

**Solution**: Break transaction into multiple local transactions, each with compensating actions.

### Types of Sagas

#### 1. Choreography-based Saga

Each service publishes events that trigger other services.

```python
# Order Service
class OrderService:
    def create_order(self, order_data):
        # Step 1: Create order
        order = Order(
            id=generate_id(),
            customer_id=order_data['customer_id'],
            total=order_data['total'],
            status='PENDING'
        )
        self.save_order(order)
        
        # Publish event
        self.event_bus.publish('OrderCreated', {
            'order_id': order.id,
            'customer_id': order.customer_id,
            'total': order.total
        })
        
        return order
    
    def handle_payment_failed(self, event):
        # Compensating transaction
        order = self.get_order(event['order_id'])
        order.status = 'CANCELLED'
        self.save_order(order)
        
        print(f"Order {order.id} cancelled due to payment failure")

# Payment Service
class PaymentService:
    def handle_order_created(self, event):
        # Step 2: Process payment
        try:
            payment = self.process_payment(
                customer_id=event['customer_id'],
                amount=event['total']
            )
            
            # Publish success event
            self.event_bus.publish('PaymentCompleted', {
                'order_id': event['order_id'],
                'payment_id': payment.id
            })
        except InsufficientFundsError:
            # Publish failure event
            self.event_bus.publish('PaymentFailed', {
                'order_id': event['order_id'],
                'reason': 'Insufficient funds'
            })

# Inventory Service
class InventoryService:
    def handle_payment_completed(self, event):
        # Step 3: Reserve inventory
        try:
            self.reserve_items(event['order_id'])
            
            # Publish success event
            self.event_bus.publish('InventoryReserved', {
                'order_id': event['order_id']
            })
        except OutOfStockError:
            # Compensating transaction
            self.event_bus.publish('InventoryReservationFailed', {
                'order_id': event['order_id']
            })
    
    def handle_payment_failed(self, event):
        # No action needed - inventory not reserved yet
        pass

# Shipping Service
class ShippingService:
    def handle_inventory_reserved(self, event):
        # Step 4: Create shipment
        shipment = self.create_shipment(event['order_id'])
        
        self.event_bus.publish('ShipmentCreated', {
            'order_id': event['order_id'],
            'shipment_id': shipment.id
        })
    
    def handle_inventory_reservation_failed(self, event):
        # Trigger refund
        self.event_bus.publish('RefundRequired', {
            'order_id': event['order_id']
        })
```

#### 2. Orchestration-based Saga

Central orchestrator coordinates the saga.

```python
from enum import Enum

class SagaStep(Enum):
    CREATE_ORDER = 1
    PROCESS_PAYMENT = 2
    RESERVE_INVENTORY = 3
    CREATE_SHIPMENT = 4

class SagaStatus(Enum):
    STARTED = 'STARTED'
    COMPLETED = 'COMPLETED'
    FAILED = 'FAILED'
    COMPENSATING = 'COMPENSATING'

# Saga Orchestrator
class OrderSagaOrchestrator:
    def __init__(self, order_service, payment_service, 
                 inventory_service, shipping_service):
        self.order_service = order_service
        self.payment_service = payment_service
        self.inventory_service = inventory_service
        self.shipping_service = shipping_service
    
    def execute_saga(self, order_data):
        saga_state = {
            'status': SagaStatus.STARTED,
            'completed_steps': [],
            'order_id': None,
            'payment_id': None,
            'reservation_id': None,
            'shipment_id': None
        }
        
        try:
            # Step 1: Create Order
            order = self.order_service.create_order(order_data)
            saga_state['order_id'] = order.id
            saga_state['completed_steps'].append(SagaStep.CREATE_ORDER)
            
            # Step 2: Process Payment
            payment = self.payment_service.process_payment(
                order.customer_id,
                order.total
            )
            saga_state['payment_id'] = payment.id
            saga_state['completed_steps'].append(SagaStep.PROCESS_PAYMENT)
            
            # Step 3: Reserve Inventory
            reservation = self.inventory_service.reserve_items(order.id)
            saga_state['reservation_id'] = reservation.id
            saga_state['completed_steps'].append(SagaStep.RESERVE_INVENTORY)
            
            # Step 4: Create Shipment
            shipment = self.shipping_service.create_shipment(order.id)
            saga_state['shipment_id'] = shipment.id
            saga_state['completed_steps'].append(SagaStep.CREATE_SHIPMENT)
            
            # All steps completed
            saga_state['status'] = SagaStatus.COMPLETED
            self.order_service.mark_order_completed(order.id)
            
            return saga_state
            
        except Exception as e:
            # Saga failed - start compensation
            saga_state['status'] = SagaStatus.COMPENSATING
            self.compensate(saga_state)
            saga_state['status'] = SagaStatus.FAILED
            raise SagaFailedException(f"Saga failed: {str(e)}")
    
    def compensate(self, saga_state):
        """Execute compensating transactions in reverse order"""
        print("Starting compensation...")
        
        # Reverse the completed steps
        for step in reversed(saga_state['completed_steps']):
            try:
                if step == SagaStep.CREATE_SHIPMENT:
                    self.shipping_service.cancel_shipment(
                        saga_state['shipment_id']
                    )
                    print("Shipment cancelled")
                
                elif step == SagaStep.RESERVE_INVENTORY:
                    self.inventory_service.release_items(
                        saga_state['reservation_id']
                    )
                    print("Inventory released")
                
                elif step == SagaStep.PROCESS_PAYMENT:
                    self.payment_service.refund_payment(
                        saga_state['payment_id']
                    )
                    print("Payment refunded")
                
                elif step == SagaStep.CREATE_ORDER:
                    self.order_service.cancel_order(
                        saga_state['order_id']
                    )
                    print("Order cancelled")
                    
            except Exception as e:
                print(f"Compensation failed for {step}: {str(e)}")
                # Log for manual intervention

# Usage
orchestrator = OrderSagaOrchestrator(
    order_service,
    payment_service,
    inventory_service,
    shipping_service
)

try:
    result = orchestrator.execute_saga({
        'customer_id': 'cust_123',
        'items': [{'product_id': 'prod_1', 'quantity': 2}],
        'total': 100.00
    })
    print(f"Order completed: {result['order_id']}")
except SagaFailedException as e:
    print(f"Order failed: {str(e)}")
```

### Saga Pattern Comparison

| **Aspect** | **Choreography** | **Orchestration** |
|------------|------------------|-------------------|
| **Coordination** | Decentralized | Centralized |
| **Coupling** | Loose | Tighter |
| **Complexity** | Harder to track | Easier to track |
| **Failure Handling** | Distributed | Centralized |
| **Best for** | Simple workflows | Complex workflows |

---

## Orchestration vs Choreography

### Choreography

**Each service knows what to do based on events from other services.**

```python
# Event-driven choreography example

class OrderService:
    def create_order(self, data):
        order = self.save_order(data)
        self.event_bus.publish('order.created', order)

class InventoryService:
    def __init__(self):
        self.event_bus.subscribe('order.created', self.handle_order_created)
    
    def handle_order_created(self, order):
        self.reserve_inventory(order)
        self.event_bus.publish('inventory.reserved', order)

class ShippingService:
    def __init__(self):
        self.event_bus.subscribe('inventory.reserved', self.handle_inventory_reserved)
    
    def handle_inventory_reserved(self, order):
        self.create_shipment(order)
        self.event_bus.publish('shipment.created', order)

class NotificationService:
    def __init__(self):
        self.event_bus.subscribe('shipment.created', self.handle_shipment_created)
    
    def handle_shipment_created(self, order):
        self.send_notification(order.customer_id, "Order shipped!")
```

**Pros:**
- Loose coupling
- Services are independent
- Easy to add new services

**Cons:**
- Hard to track workflow
- Difficult to debug
- Cyclic dependencies risk

### Orchestration

**Central service coordinates the workflow.**

```python
class OrderWorkflowOrchestrator:
    def __init__(self, inventory_service, shipping_service, notification_service):
        self.inventory = inventory_service
        self.shipping = shipping_service
        self.notification = notification_service
    
    def process_order(self, order):
        # Step 1: Reserve inventory
        inventory_result = self.inventory.reserve(order)
        if not inventory_result.success:
            return self.handle_failure(order, "Inventory reservation failed")
        
        # Step 2: Create shipment
        shipment_result = self.shipping.create_shipment(order)
        if not shipment_result.success:
            self.inventory.release(order)  # Compensate
            return self.handle_failure(order, "Shipment creation failed")
        
        # Step 3: Send notification
        self.notification.send(order.customer_id, "Order shipped!")
        
        return {"status": "success", "order_id": order.id}
    
    def handle_failure(self, order, reason):
        # Centralized failure handling
        self.notification.send(order.customer_id, f"Order failed: {reason}")
        return {"status": "failed", "reason": reason}
```

**Pros:**
- Clear workflow visibility
- Easier to debug
- Centralized control

**Cons:**
- Single point of failure
- Orchestrator can become complex
- Tight coupling with orchestrator

---

## Anti-Corruption Layer

**Problem**: Protecting your domain model from external systems' complexity.

**Solution**: Create a translation layer between systems.

```python
# External legacy system with poor API design
class LegacyCustomerSystem:
    def get_cust_info(self, cust_num):
        # Returns poorly structured data
        return {
            'cust_num': cust_num,
            'f_name': 'John',
            'l_name': 'Doe',
            'addr_line_1': '123 Main St',
            'addr_line_2': '',
            'city': 'NYC',
            'state': 'NY',
            'zip': '10001',
            'ph_num': '1234567890',
            'email_addr': 'john@example.com',
            'cust_type': 'P',  # P = Premium, R = Regular
            'loyalty_pts': 1500
        }

# Your clean domain model
class Customer:
    def __init__(self, id, name, address, contact, tier, loyalty_points):
        self.id = id
        self.name = name
        self.address = address
        self.contact = contact
        self.tier = tier
        self.loyalty_points = loyalty_points

class Name:
    def __init__(self, first_name, last_name):
        self.first_name = first_name
        self.last_name = last_name
    
    def full_name(self):
        return f"{self.first_name} {self.last_name}"

class Address:
    def __init__(self, street, city, state, zip_code):
        self.street = street
        self.city = city
        self.state = state
        self.zip_code = zip_code

class Contact:
    def __init__(self, phone, email):
        self.phone = phone
        self.email = email

class CustomerTier(Enum):
    REGULAR = "REGULAR"
    PREMIUM = "PREMIUM"

# Anti-Corruption Layer
class CustomerAdapter:
    """Translates between legacy system and our domain model"""
    
    def __init__(self, legacy_system):
        self.legacy_system = legacy_system
    
    def get_customer(self, customer_id):
        # Get data from legacy system
        legacy_data = self.legacy_system.get_cust_info(customer_id)
        
        # Transform to our domain model
        return self._to_domain_model(legacy_data)
    
    def _to_domain_model(self, legacy_data):
        # Create clean domain objects
        name = Name(
            first_name=legacy_data['f_name'],
            last_name=legacy_data['l_name']
        )
        
        # Combine address fields
        street = legacy_data['addr_line_1']
        if legacy_data['addr_line_2']:
            street += f", {legacy_data['addr_line_2']}"
        
        address = Address(
            street=street,
            city=legacy_data['city'],
            state=legacy_data['state'],
            zip_code=legacy_data['zip']
        )
        
        contact = Contact(
            phone=self._format_phone(legacy_data['ph_num']),
            email=legacy_data['email_addr']
        )
        
        # Map customer type
        tier = (CustomerTier.PREMIUM if legacy_data['cust_type'] == 'P' 
                else CustomerTier.REGULAR)
        
        return Customer(
            id=legacy_data['cust_num'],
            name=name,
            address=address,
            contact=contact,
            tier=tier,
            loyalty_points=legacy_data['loyalty_pts']
        )
    
    def _format_phone(self, phone_number):
        # Format phone number
        return f"({phone_number[:3]}) {phone_number[3:6]}-{phone_number[6:]}"

# Your application code - clean and isolated from legacy system
class CustomerService:
    def __init__(self, customer_adapter):
        self.adapter = customer_adapter
    
    def get_customer_summary(self, customer_id):
        customer = self.adapter.get_customer(customer_id)
        
        return {
            'name': customer.name.full_name(),
            'email': customer.contact.email,
            'tier': customer.tier.value,
            'points': customer.loyalty_points
        }

# Usage
legacy_system = LegacyCustomerSystem()
adapter = CustomerAdapter(legacy_system)
service = CustomerService(adapter)

summary = service.get_customer_summary('12345')
print(summary)
# Output: Clean, well-structured data
```

---

## Backend for Frontend (BFF)

**Problem**: Frontend applications have different data requirements. Forcing them to use the same API leads to over-fetching or multiple requests.

**Solution**: Create dedicated backends for each frontend type.

```python
from flask import Flask, jsonify

# Shared microservices
class UserService:
    def get_user(self, user_id):
        return {
            'id': user_id,
            'name': 'John Doe',
            'email': 'john@example.com',
            'profile_picture': 'https://cdn.example.com/user123.jpg',
            'bio': 'Software engineer...',
            'location': 'New York',
            'joined_date': '2020-01-01'
        }

class ProductService:
    def get_products(self, user_id):
        return [
            {
                'id': 'prod_1',
                'name': 'Laptop',
                'price': 999.99,
                'image': 'https://cdn.example.com/laptop.jpg',
                'description': 'High-performance laptop...',
                'specs': {'ram': '16GB', 'storage': '512GB SSD'},
                'reviews': 150,
                'rating': 4.5
            },
            # More products...
        ]

class OrderService:
    def get_orders(self, user_id):
        return [
            {
                'id': 'order_1',
                'date': '2024-01-15',
                'total': 999.99,
                'status': 'Delivered',
                'items': [{'product_id': 'prod_1', 'quantity': 1}],
                'shipping_address': '123 Main St',
                'tracking_number': 'TRK123456'
            }
        ]

# BFF for Web Application
class WebBFF:
    def __init__(self, user_service, product_service, order_service):
        self.user_service = user_service
        self.product_service = product_service
        self.order_service = order_service
        self.app = Flask(__name__)
        self._setup_routes()
    
    def _setup_routes(self):
        @self.app.route('/api/dashboard/<user_id>')
        def get_dashboard(user_id):
            # Web needs comprehensive data
            user = self.user_service.get_user(user_id)
            products = self.product_service.get_products(user_id)
            orders = self.order_service.get_orders(user_id)
            
            return jsonify({
                'user': user,
                'recommended_products': products[:10],
                'recent_orders': orders[:5]
            })

# BFF for Mobile Application
class MobileBFF:
    def __init__(self, user_service, product_service, order_service):
        self.user_service = user_service
        self.product_service = product_service
        self.order_service = order_service
        self.app = Flask(__name__)
        self._setup_routes()
    
    def _setup_routes(self):
        @self.app.route('/api/dashboard/<user_id>')
        def get_dashboard(user_id):
            # Mobile needs minimal data (bandwidth conscious)
            user = self.user_service.get_user(user_id)
            products = self.product_service.get_products(user_id)
            orders = self.order_service.get_orders(user_id)
            
            return jsonify({
                'user': {
                    'name': user['name'],
                    'profile_picture': user['profile_picture']
                },
                'recommended_products': [
                    {
                        'id': p['id'],
                        'name': p['name'],
                        'price': p['price'],
                        'image': p['image']  # Smaller thumbnail
                    }
                    for p in products[:5]
                ],
                'recent_orders': [
                    {
                        'id': o['id'],
                        'total': o['total'],
                        'status': o['status']
                    }
                    for o in orders[:3]
                ]
            })

# BFF for Smart TV Application
class SmartTVBFF:
    def __init__(self, user_service, product_service):
        self.user_service = user_service
        self.product_service = product_service
        self.app = Flask(__name__)
        self._setup_routes()
    
    def _setup_routes(self):
        @self.app.route('/api/browse/<user_id>')
        def browse_products(user_id):
            # TV needs large images, minimal text
            products = self.product_service.get_products(user_id)
            
            return jsonify({
                'products': [
                    {
                        'id': p['id'],
                        'name': p['name'],
                        'price': p['price'],
                        'image_hd': p['image'].replace('.jpg', '_hd.jpg'),
                        'rating': p['rating']
                    }
                    for p in products[:20]
                ]
            })

# Benefits of BFF:
# 1. Optimized for each client
# 2. Reduces over-fetching
# 3. Client-specific caching
# 4. Independent scaling
# 5. Easier to evolve each frontend independently
```

---

## Strangler Fig Pattern

**Problem**: Migrating from monolith to microservices without big-bang rewrite.

**Solution**: Gradually replace parts of the monolith with microservices.

```python
from flask import Flask, request, jsonify
import requests

# Legacy Monolith
class LegacyMonolith:
    def __init__(self):
        self.app = Flask(__name__)
        self._setup_routes()
    
    def _setup_routes(self):
        @self.app.route('/api/users/<user_id>')
        def get_user(user_id):
            # Legacy code
            return jsonify({'id': user_id, 'name': 'John'})
        
        @self.app.route('/api/products')
        def get_products():
            # Legacy code
            return jsonify([{'id': 1, 'name': 'Product 1'}])
        
        @self.app.route('/api/orders/<order_id>')
        def get_order(order_id):
            # Legacy code
            return jsonify({'id': order_id, 'total': 100})

# New Microservices
class UserMicroservice:
    def __init__(self):
        self.app = Flask(__name__)
        
        @self.app.route('/users/<user_id>')
        def get_user(user_id):
            # New implementation with better features
            return jsonify({
                'id': user_id,
                'name': 'John Doe',
                'email': 'john@example.com',
                'preferences': {}
            })

class ProductMicroservice:
    def __init__(self):
        self.app = Flask(__name__)
        
        @self.app.route('/products')
        def get_products():
            # New implementation
            return jsonify([
                {'id': 1, 'name': 'Product 1', 'price': 99.99},
                {'id': 2, 'name': 'Product 2', 'price': 149.99}
            ])

# Strangler Facade (API Gateway)
class StranglerFacade:
    def __init__(self, monolith_url, user_service_url, product_service_url):
        self.monolith_url = monolith_url
        self.user_service_url = user_service_url
        self.product_service_url = product_service_url
        self.app = Flask(__name__)
        self._setup_routes()
    
    def _setup_routes(self):
        @self.app.route('/api/users/<user_id>')
        def get_user(user_id):
            # Route to NEW user microservice
            response = requests.get(f"{self.user_service_url}/users/{user_id}")
            return jsonify(response.json())
        
        @self.app.route('/api/products')
        def get_products():
            # Route to NEW product microservice
            response = requests.get(f"{self.product_service_url}/products")
            return jsonify(response.json())
        
        @self.app.route('/api/orders/<order_id>')
        def get_order(order_id):
            # Still route to OLD monolith (not migrated yet)
            response = requests.get(f"{self.monolith_url}/api/orders/{order_id}")
            return jsonify(response.json())

# Migration Strategy
"""
Phase 1: Setup strangler facade
- All requests go through facade
- All routed to monolith initially

Phase 2: Migrate user service
- Deploy user microservice
- Update facade to route user requests to new service
- Keep monolith user code as fallback

Phase 3: Migrate product service
- Deploy product microservice
- Update facade routing
- Monitor and validate

Phase 4: Migrate order service
- Deploy order microservice
- Update facade routing
- Finally, decommission monolith

Benefits:
- No big-bang rewrite
- Incremental migration
- Easy rollback
- Test in production
- Business continuity maintained
"""
```

---

## CQRS (Command Query Responsibility Segregation)

**Separate read and write operations for better performance and scalability.**

```python
from abc import ABC, abstractmethod
from datetime import datetime

# Commands (Write operations)
class Command(ABC):
    pass

class CreateOrderCommand(Command):
    def __init__(self, customer_id, items, total):
        self.customer_id = customer_id
        self.items = items
        self.total = total

class UpdateOrderStatusCommand(Command):
    def __init__(self, order_id, status):
        self.order_id = order_id
        self.status = status

# Command Handlers (Write side)
class CommandHandler(ABC):
    @abstractmethod
    def handle(self, command):
        pass

class CreateOrderCommandHandler(CommandHandler):
    def __init__(self, write_db, event_bus):
        self.write_db = write_db
        self.event_bus = event_bus
    
    def handle(self, command):
        # Write to normalized database
        order = {
            'id': self._generate_id(),
            'customer_id': command.customer_id,
            'items': command.items,
            'total': command.total,
            'status': 'PENDING',
            'created_at': datetime.now()
        }
        
        self.write_db.orders.insert(order)
        
        # Publish event for read side
        self.event_bus.publish('OrderCreated', order)
        
        return order['id']

# Queries (Read operations)
class Query(ABC):
    pass

class GetOrderByIdQuery(Query):
    def __init__(self, order_id):
        self.order_id = order_id

class GetCustomerOrdersQuery(Query):
    def __init__(self, customer_id):
        self.customer_id = customer_id

class GetOrderStatisticsQuery(Query):
    def __init__(self, start_date, end_date):
        self.start_date = start_date
        self.end_date = end_date

# Query Handlers (Read side)
class QueryHandler(ABC):
    @abstractmethod
    def handle(self, query):
        pass

class GetOrderByIdQueryHandler(QueryHandler):
    def __init__(self, read_db):
        self.read_db = read_db  # Denormalized, optimized for reads
    
    def handle(self, query):
        # Read from optimized read model
        return self.read_db.order_views.find_one({
            'order_id': query.order_id
        })

class GetCustomerOrdersQueryHandler(QueryHandler):
    def __init__(self, read_db):
        self.read_db = read_db
    
    def handle(self, query):
        # Pre-joined, denormalized data for fast reads
        return self.read_db.customer_order_views.find({
            'customer_id': query.customer_id
        })

# Read Model Projector (Updates read database from events)
class OrderReadModelProjector:
    def __init__(self, read_db, event_bus):
        self.read_db = read_db
        event_bus.subscribe('OrderCreated', self.handle_order_created)
        event_bus.subscribe('OrderUpdated', self.handle_order_updated)
    
    def handle_order_created(self, event):
        # Create denormalized view optimized for queries
        order_view = {
            'order_id': event['id'],
            'customer_id': event['customer_id'],
            'customer_name': self._get_customer_name(event['customer_id']),
            'items': event['items'],
            'total': event['total'],
            'status': event['status'],
            'created_at': event['created_at']
        }
        
        self.read_db.order_views.insert(order_view)
    
    def handle_order_updated(self, event):
        self.read_db.order_views.update(
            {'order_id': event['order_id']},
            {'$set': {'status': event['status']}}
        )

# CQRS Facade
class OrderService:
    def __init__(self, command_bus, query_bus):
        self.command_bus = command_bus
        self.query_bus = query_bus
    
    # Write operations
    def create_order(self, customer_id, items, total):
        command = CreateOrderCommand(customer_id, items, total)
        return self.command_bus.dispatch(command)
    
    # Read operations
    def get_order(self, order_id):
        query = GetOrderByIdQuery(order_id)
        return self.query_bus.dispatch(query)
    
    def get_customer_orders(self, customer_id):
        query = GetCustomerOrdersQuery(customer_id)
        return self.query_bus.dispatch(query)

# Benefits:
# - Separate scaling of reads and writes
# - Optimized read models
# - Complex queries without affecting writes
# - Event-driven architecture
# - Better performance
```

This guide covers advanced distributed system patterns essential for building scalable, maintainable microservices architectures. Each pattern solves specific challenges in distributed systems.
