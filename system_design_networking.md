# Networking & Infrastructure Guide

## Table of Contents

1. [Load Balancing](#load-balancing)
2. [API Gateways](#api-gateways)
3. [Proxy Patterns](#proxy-patterns)
4. [DNS & Service Discovery](#dns--service-discovery)
5. [HTTPS & Security](#https--security)
6. [CDN (Content Delivery Network)](#cdn-content-delivery-network)
7. [Network Protocols](#network-protocols)

---

## Load Balancing

### Load Balancing Algorithms

#### 1. Round Robin

```python
class RoundRobinLoadBalancer:
    def __init__(self, servers):
        self.servers = servers
        self.current = 0
    
    def get_server(self):
        server = self.servers[self.current]
        self.current = (self.current + 1) % len(self.servers)
        return server

# Usage
lb = RoundRobinLoadBalancer(['server1', 'server2', 'server3'])
print(lb.get_server())  # server1
print(lb.get_server())  # server2
print(lb.get_server())  # server3
print(lb.get_server())  # server1
```

#### 2. Weighted Round Robin

```python
class WeightedRoundRobinLoadBalancer:
    def __init__(self, servers_with_weights):
        # servers_with_weights = [('server1', 5), ('server2', 3), ('server3', 2)]
        self.servers = []
        for server, weight in servers_with_weights:
            self.servers.extend([server] * weight)
        self.current = 0
    
    def get_server(self):
        server = self.servers[self.current]
        self.current = (self.current + 1) % len(self.servers)
        return server

# Usage
lb = WeightedRoundRobinLoadBalancer([
    ('high-spec-server', 5),
    ('medium-server', 3),
    ('low-server', 2)
])
# high-spec-server will get 50% of traffic
```

#### 3. Least Connections

```python
import threading

class LeastConnectionsLoadBalancer:
    def __init__(self, servers):
        self.servers = {server: 0 for server in servers}
        self.lock = threading.Lock()
    
    def get_server(self):
        with self.lock:
            # Select server with fewest connections
            server = min(self.servers, key=self.servers.get)
            self.servers[server] += 1
            return server
    
    def release_connection(self, server):
        with self.lock:
            if server in self.servers and self.servers[server] > 0:
                self.servers[server] -= 1

# Usage
lb = LeastConnectionsLoadBalancer(['server1', 'server2', 'server3'])
server = lb.get_server()
# ... handle request ...
lb.release_connection(server)
```

#### 4. IP Hash (Sticky Sessions)

```python
import hashlib

class IPHashLoadBalancer:
    def __init__(self, servers):
        self.servers = servers
    
    def get_server(self, client_ip):
        hash_value = int(hashlib.md5(client_ip.encode()).hexdigest(), 16)
        index = hash_value % len(self.servers)
        return self.servers[index]

# Usage
lb = IPHashLoadBalancer(['server1', 'server2', 'server3'])
server = lb.get_server('192.168.1.100')  # Always routes to same server
```

#### 5. Health Check Integration

```python
import requests
import time
import threading

class LoadBalancerWithHealthCheck:
    def __init__(self, servers, health_check_interval=10):
        self.servers = servers
        self.healthy_servers = set(servers)
        self.current = 0
        self.lock = threading.Lock()
        
        # Start health check thread
        self.health_check_thread = threading.Thread(
            target=self._health_check_loop,
            args=(health_check_interval,),
            daemon=True
        )
        self.health_check_thread.start()
    
    def _health_check_loop(self, interval):
        while True:
            for server in self.servers:
                if self._is_healthy(server):
                    with self.lock:
                        self.healthy_servers.add(server)
                else:
                    with self.lock:
                        self.healthy_servers.discard(server)
            time.sleep(interval)
    
    def _is_healthy(self, server):
        try:
            response = requests.get(f"http://{server}/health", timeout=2)
            return response.status_code == 200
        except:
            return False
    
    def get_server(self):
        with self.lock:
            if not self.healthy_servers:
                raise Exception("No healthy servers available")
            
            healthy_list = list(self.healthy_servers)
            server = healthy_list[self.current % len(healthy_list)]
            self.current += 1
            return server
```

### Layer 4 vs Layer 7 Load Balancing

```
Layer 4 (Transport Layer):
- Routes based on IP and port
- Faster, less CPU intensive
- No visibility into HTTP headers
- Example: AWS NLB

Layer 7 (Application Layer):
- Routes based on HTTP headers, URL, cookies
- Content-based routing
- SSL termination
- Example: AWS ALB, Nginx
```

#### Nginx Configuration

```nginx
# Layer 7 Load Balancer Configuration
upstream backend {
    # Load balancing method
    least_conn;  # or: ip_hash, random, hash
    
    # Backend servers
    server backend1.example.com:8080 weight=3;
    server backend2.example.com:8080 weight=2;
    server backend3.example.com:8080 weight=1 backup;
    
    # Health checks
    server backend4.example.com:8080 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name api.example.com;
    
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Connection settings
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
    
    # Path-based routing
    location /api/v1/ {
        proxy_pass http://backend_v1;
    }
    
    location /api/v2/ {
        proxy_pass http://backend_v2;
    }
}
```

---

## API Gateways

### Purpose
- **Single entry point** for all clients
- **Authentication & Authorization**
- **Rate limiting**
- **Request/Response transformation**
- **Routing & Load balancing**
- **Caching**
- **Monitoring & Logging**

### API Gateway Implementation

```python
from flask import Flask, request, jsonify
import requests
import time
from functools import wraps

app = Flask(__name__)

class APIGateway:
    def __init__(self):
        self.rate_limiter = RateLimiter()
        self.services = {
            'users': 'http://user-service:8080',
            'orders': 'http://order-service:8080',
            'products': 'http://product-service:8080'
        }
    
    def authenticate(self, token):
        """JWT validation"""
        try:
            # Validate JWT token
            return {'user_id': 123, 'role': 'user'}
        except:
            return None
    
    def authorize(self, user, resource, action):
        """Check permissions"""
        # Check if user has permission for action on resource
        return True
    
    def route_request(self, service, path, method, headers, data=None):
        """Forward request to microservice"""
        service_url = self.services.get(service)
        if not service_url:
            return {'error': 'Service not found'}, 404
        
        url = f"{service_url}{path}"
        
        try:
            if method == 'GET':
                response = requests.get(url, headers=headers, timeout=5)
            elif method == 'POST':
                response = requests.post(url, headers=headers, json=data, timeout=5)
            elif method == 'PUT':
                response = requests.put(url, headers=headers, json=data, timeout=5)
            elif method == 'DELETE':
                response = requests.delete(url, headers=headers, timeout=5)
            
            return response.json(), response.status_code
        except requests.Timeout:
            return {'error': 'Service timeout'}, 504
        except Exception as e:
            return {'error': 'Service unavailable'}, 503

gateway = APIGateway()

def require_auth(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.headers.get('Authorization')
        if not token:
            return jsonify({'error': 'No token provided'}), 401
        
        user = gateway.authenticate(token)
        if not user:
            return jsonify({'error': 'Invalid token'}), 401
        
        request.user = user
        return f(*args, **kwargs)
    return decorated

@app.route('/<service>/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE'])
@require_auth
def proxy(service, path):
    # Rate limiting
    client_id = request.user['user_id']
    if not gateway.rate_limiter.allow_request(client_id):
        return jsonify({'error': 'Rate limit exceeded'}), 429
    
    # Route to microservice
    result, status = gateway.route_request(
        service,
        f'/{path}',
        request.method,
        request.headers,
        request.get_json()
    )
    
    return jsonify(result), status
```

### Rate Limiting

```python
import time
from collections import defaultdict

class RateLimiter:
    def __init__(self, max_requests=100, window_seconds=60):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests = defaultdict(list)
    
    def allow_request(self, client_id):
        """Token bucket algorithm"""
        now = time.time()
        window_start = now - self.window_seconds
        
        # Remove old requests outside window
        self.requests[client_id] = [
            req_time for req_time in self.requests[client_id]
            if req_time > window_start
        ]
        
        # Check if under limit
        if len(self.requests[client_id]) < self.max_requests:
            self.requests[client_id].append(now)
            return True
        
        return False

# Sliding Window Log
class SlidingWindowRateLimiter:
    def __init__(self, max_requests=100, window_seconds=60):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests = defaultdict(lambda: {'count': 0, 'reset_time': 0})
    
    def allow_request(self, client_id):
        now = time.time()
        client_data = self.requests[client_id]
        
        # Reset if window expired
        if now > client_data['reset_time']:
            client_data['count'] = 0
            client_data['reset_time'] = now + self.window_seconds
        
        # Check limit
        if client_data['count'] < self.max_requests:
            client_data['count'] += 1
            return True
        
        return False
```

---

## Proxy Patterns

### Forward Proxy

**Client → Forward Proxy → Internet**

```python
import requests

class ForwardProxy:
    """Client uses this to access external resources"""
    
    def __init__(self, cache_enabled=True):
        self.cache = {}
        self.cache_enabled = cache_enabled
    
    def request(self, url, headers=None):
        # Check cache
        if self.cache_enabled and url in self.cache:
            print(f"Cache hit for {url}")
            return self.cache[url]
        
        # Forward request
        print(f"Fetching {url}")
        response = requests.get(url, headers=headers)
        
        # Cache response
        if self.cache_enabled:
            self.cache[url] = response.content
        
        return response.content

# Usage
proxy = ForwardProxy()
content = proxy.request('https://api.example.com/data')
```

### Reverse Proxy

**Client → Reverse Proxy → Backend Servers**

```python
from flask import Flask, request
import requests

app = Flask(__name__)

class ReverseProxy:
    """Server-side proxy, client doesn't know backend structure"""
    
    def __init__(self):
        self.backend_servers = [
            'http://backend1:8080',
            'http://backend2:8080',
            'http://backend3:8080'
        ]
        self.current = 0
    
    def get_backend(self):
        server = self.backend_servers[self.current]
        self.current = (self.current + 1) % len(self.backend_servers)
        return server

reverse_proxy = ReverseProxy()

@app.route('/<path:path>', methods=['GET', 'POST'])
def proxy_request(path):
    backend = reverse_proxy.get_backend()
    url = f"{backend}/{path}"
    
    if request.method == 'GET':
        resp = requests.get(url, params=request.args)
    else:
        resp = requests.post(url, json=request.get_json())
    
    return resp.content, resp.status_code
```

---

## DNS & Service Discovery

### DNS Basics

```
Client Query: api.example.com
         ↓
Root DNS Server → .com DNS Server → example.com DNS Server
         ↓
Returns: 203.0.113.10 (IP address)
```

### DNS Record Types

```
A Record:     example.com → 203.0.113.10 (IPv4)
AAAA Record:  example.com → 2001:db8::1 (IPv6)
CNAME:        www.example.com → example.com (Alias)
MX Record:    example.com → mail.example.com (Email)
TXT Record:   example.com → "v=spf1 include:_spf.google.com ~all" (Metadata)
```

### Service Discovery

#### Consul

```python
import consul

class ServiceDiscovery:
    def __init__(self, consul_host='localhost', consul_port=8500):
        self.consul = consul.Consul(host=consul_host, port=consul_port)
    
    def register_service(self, service_name, service_port, service_address):
        """Register service with Consul"""
        self.consul.agent.service.register(
            service_name,
            service_id=f"{service_name}-{service_address}-{service_port}",
            address=service_address,
            port=service_port,
            check=consul.Check.http(
                f"http://{service_address}:{service_port}/health",
                interval="10s"
            )
        )
        print(f"Registered {service_name} at {service_address}:{service_port}")
    
    def discover_service(self, service_name):
        """Discover healthy service instances"""
        _, services = self.consul.health.service(service_name, passing=True)
        
        instances = [
            {
                'address': service['Service']['Address'],
                'port': service['Service']['Port']
            }
            for service in services
        ]
        
        return instances
    
    def deregister_service(self, service_id):
        """Remove service from registry"""
        self.consul.agent.service.deregister(service_id)

# Usage
sd = ServiceDiscovery()

# Register service
sd.register_service('user-service', 8080, '192.168.1.10')

# Discover services
instances = sd.discover_service('user-service')
print(f"Available instances: {instances}")
```

---

## HTTPS & Security

### SSL/TLS Handshake

```
Client                                   Server
  |                                        |
  |-------- ClientHello ----------------→ |
  |       (supported ciphers)              |
  |                                        |
  |←------- ServerHello ----------------- |
  |       (chosen cipher, certificate)    |
  |                                        |
  |-------- ClientKeyExchange ----------→ |
  |       (encrypted pre-master secret)   |
  |                                        |
  |-------- ChangeCipherSpec -----------→ |
  |-------- Finished -------------------→ |
  |                                        |
  |←------- ChangeCipherSpec ------------ |
  |←------- Finished -------------------- |
  |                                        |
  |←----→ Encrypted Application Data ←--→|
```

### Certificate Management

```python
from cryptography import x509
from cryptography.x509.oid import NameOID
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization
from datetime import datetime, timedelta

def generate_self_signed_cert():
    """Generate self-signed certificate"""
    # Generate private key
    private_key = rsa.generate_private_key(
        public_exponent=65537,
        key_size=2048
    )
    
    # Generate certificate
    subject = issuer = x509.Name([
        x509.NameAttribute(NameOID.COUNTRY_NAME, "US"),
        x509.NameAttribute(NameOID.STATE_OR_PROVINCE_NAME, "California"),
        x509.NameAttribute(NameOID.LOCALITY_NAME, "San Francisco"),
        x509.NameAttribute(NameOID.ORGANIZATION_NAME, "My Company"),
        x509.NameAttribute(NameOID.COMMON_NAME, "example.com"),
    ])
    
    cert = x509.CertificateBuilder().subject_name(
        subject
    ).issuer_name(
        issuer
    ).public_key(
        private_key.public_key()
    ).serial_number(
        x509.random_serial_number()
    ).not_valid_before(
        datetime.utcnow()
    ).not_valid_after(
        datetime.utcnow() + timedelta(days=365)
    ).add_extension(
        x509.SubjectAlternativeName([
            x509.DNSName("example.com"),
            x509.DNSName("www.example.com"),
        ]),
        critical=False,
    ).sign(private_key, hashes.SHA256())
    
    # Save private key
    with open("private_key.pem", "wb") as f:
        f.write(private_key.private_bytes(
            encoding=serialization.Encoding.PEM,
            format=serialization.PrivateFormat.TraditionalOpenSSL,
            encryption_algorithm=serialization.NoEncryption()
        ))
    
    # Save certificate
    with open("certificate.pem", "wb") as f:
        f.write(cert.public_bytes(serialization.Encoding.PEM))
    
    print("Certificate generated successfully")
```

### Security Headers

```python
from flask import Flask, make_response

app = Flask(__name__)

def add_security_headers(response):
    """Add security headers to response"""
    response.headers['Strict-Transport-Security'] = 'max-age=31536000; includeSubDomains'
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    response.headers['X-XSS-Protection'] = '1; mode=block'
    response.headers['Content-Security-Policy'] = "default-src 'self'"
    return response

@app.after_request
def apply_security_headers(response):
    return add_security_headers(response)
```

---

## CDN (Content Delivery Network)

### How CDN Works

```
User (Tokyo) → CDN Edge (Tokyo) → Origin Server (US)
                     ↓
                 Cache Hit: Serve from edge
                 Cache Miss: Fetch from origin, cache, then serve
```

### CDN Implementation Concept

```python
import hashlib
import time

class SimpleCDN:
    def __init__(self, origin_server):
        self.origin_server = origin_server
        self.edge_cache = {}
        self.ttl = 3600  # 1 hour
    
    def get_content(self, url):
        cache_key = hashlib.md5(url.encode()).hexdigest()
        
        # Check cache
        if cache_key in self.edge_cache:
            cached_item = self.edge_cache[cache_key]
            
            # Check if cache is still valid
            if time.time() < cached_item['expires_at']:
                print(f"CDN Cache HIT for {url}")
                return cached_item['content']
            else:
                print(f"CDN Cache EXPIRED for {url}")
                del self.edge_cache[cache_key]
        
        # Cache miss - fetch from origin
        print(f"CDN Cache MISS for {url} - fetching from origin")
        content = self.fetch_from_origin(url)
        
        # Cache content
        self.edge_cache[cache_key] = {
            'content': content,
            'expires_at': time.time() + self.ttl
        }
        
        return content
    
    def fetch_from_origin(self, url):
        # Simulate fetching from origin server
        return f"Content from origin for {url}"
    
    def invalidate_cache(self, url):
        """Manually invalidate cached content"""
        cache_key = hashlib.md5(url.encode()).hexdigest()
        if cache_key in self.edge_cache:
            del self.edge_cache[cache_key]
            print(f"Invalidated cache for {url}")

# Usage
cdn = SimpleCDN(origin_server="https://origin.example.com")

# First request - cache miss
content = cdn.get_content("/images/logo.png")

# Second request - cache hit
content = cdn.get_content("/images/logo.png")

# Invalidate after content update
cdn.invalidate_cache("/images/logo.png")
```

---

## Network Protocols

### HTTP/1.1 vs HTTP/2 vs HTTP/3

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|---------|---------|
| **Transport** | TCP | TCP | QUIC (UDP) |
| **Multiplexing** | No (6 connections) | Yes | Yes |
| **Header Compression** | No | Yes (HPACK) | Yes (QPACK) |
| **Server Push** | No | Yes | Yes |
| **Connection** | Multiple | Single | Single |
| **Head-of-line blocking** | Yes | Yes (TCP level) | No |

### WebSocket

```python
# WebSocket for real-time bi-directional communication
from flask import Flask
from flask_socketio import SocketIO, emit

app = Flask(__name__)
socketio = SocketIO(app)

@socketio.on('connect')
def handle_connect():
    print('Client connected')
    emit('message', {'data': 'Connected to server'})

@socketio.on('message')
def handle_message(message):
    print(f'Received: {message}')
    # Broadcast to all clients
    emit('message', message, broadcast=True)

@socketio.on('disconnect')
def handle_disconnect():
    print('Client disconnected')

if __name__ == '__main__':
    socketio.run(app, port=5000)
```

This comprehensive networking guide covers all essential infrastructure components for building scalable distributed systems!
