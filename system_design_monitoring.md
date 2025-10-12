# Monitoring & Observability Guide

## Table of Contents

1. [Three Pillars of Observability](#three-pillars-of-observability)
2. [Logging](#logging)
3. [Metrics](#metrics)
4. [Distributed Tracing](#distributed-tracing)
5. [Alerting](#alerting)
6. [Monitoring Tools](#monitoring-tools)

---

## Three Pillars of Observability

**Logs**: Discrete events with timestamps
**Metrics**: Numerical measurements over time
**Traces**: Request flow across services

---

## Logging

### Structured Logging

```python
import logging
import json
from datetime import datetime

class StructuredLogger:
    def __init__(self, service_name):
        self.service_name = service_name
        self.logger = logging.getLogger(service_name)
        self.logger.setLevel(logging.INFO)
        
        # JSON formatter
        handler = logging.StreamHandler()
        handler.setFormatter(self.JsonFormatter())
        self.logger.addHandler(handler)
    
    class JsonFormatter(logging.Formatter):
        def format(self, record):
            log_data = {
                'timestamp': datetime.utcnow().isoformat(),
                'level': record.levelname,
                'service': record.name,
                'message': record.getMessage(),
                'logger': record.name,
                'path': record.pathname,
                'line': record.lineno,
            }
            
            # Add extra fields
            if hasattr(record, 'user_id'):
                log_data['user_id'] = record.user_id
            if hasattr(record, 'request_id'):
                log_data['request_id'] = record.request_id
            
            return json.dumps(log_data)
    
    def info(self, message, **kwargs):
        extra = {k: v for k, v in kwargs.items()}
        self.logger.info(message, extra=extra)
    
    def error(self, message, exc_info=None, **kwargs):
        extra = {k: v for k, v in kwargs.items()}
        self.logger.error(message, exc_info=exc_info, extra=extra)

# Usage
logger = StructuredLogger('order-service')

logger.info('Order created', 
    user_id=123,
    order_id='ORD-456',
    total=99.99,
    request_id='req-789'
)

try:
    # Some operation
    pass
except Exception as e:
    logger.error('Order processing failed',
        exc_info=True,
        user_id=123,
        order_id='ORD-456'
    )
```

### ELK Stack Integration (Elasticsearch, Logstash, Kibana)

```python
from elasticsearch import Elasticsearch
from datetime import datetime

class ElasticsearchLogger:
    def __init__(self, hosts=['localhost:9200'], index_prefix='logs'):
        self.es = Elasticsearch(hosts)
        self.index_prefix = index_prefix
    
    def log(self, level, message, **metadata):
        """Send log to Elasticsearch"""
        log_entry = {
            '@timestamp': datetime.utcnow().isoformat(),
            'level': level,
            'message': message,
            **metadata
        }
        
        # Index with date-based index
        index_name = f"{self.index_prefix}-{datetime.utcnow().strftime('%Y.%m.%d')}"
        
        self.es.index(index=index_name, body=log_entry)
    
    def search_logs(self, query, size=100):
        """Search logs"""
        body = {
            'query': {
                'bool': {
                    'must': [
                        {'match': {'message': query}}
                    ]
                }
            },
            'sort': [{'@timestamp': 'desc'}],
            'size': size
        }
        
        results = self.es.search(index=f"{self.index_prefix}-*", body=body)
        return [hit['_source'] for hit in results['hits']['hits']]

# Usage
es_logger = ElasticsearchLogger()

es_logger.log('ERROR', 'Database connection failed',
    service='user-service',
    error_code='DB_CONN_TIMEOUT',
    database='users_db'
)

# Search logs
errors = es_logger.search_logs('Database connection failed')
```

### Log Aggregation Pattern

```python
import requests

class CentralizedLogger:
    def __init__(self, log_aggregator_url):
        self.aggregator_url = log_aggregator_url
    
    def send_logs(self, logs):
        """Batch send logs to central aggregator"""
        try:
            response = requests.post(
                f"{self.aggregator_url}/api/logs",
                json={'logs': logs},
                timeout=5
            )
            return response.status_code == 200
        except Exception as e:
            print(f"Failed to send logs: {e}")
            return False

# Usage
logger = CentralizedLogger('http://log-aggregator:8080')

logs = [
    {'level': 'INFO', 'message': 'User logged in', 'user_id': 123},
    {'level': 'ERROR', 'message': 'Payment failed', 'order_id': 456}
]

logger.send_logs(logs)
```

---

## Metrics

### Prometheus Metrics

```python
from prometheus_client import Counter, Histogram, Gauge, Summary, start_http_server
import time
import random

class MetricsCollector:
    def __init__(self):
        # Counter: monotonically increasing
        self.request_count = Counter(
            'http_requests_total',
            'Total HTTP requests',
            ['method', 'endpoint', 'status']
        )
        
        # Histogram: distribution of values
        self.request_duration = Histogram(
            'http_request_duration_seconds',
            'HTTP request latency',
            ['method', 'endpoint']
        )
        
        # Gauge: value that can go up or down
        self.active_connections = Gauge(
            'active_connections',
            'Number of active connections'
        )
        
        # Summary: similar to histogram
        self.response_size = Summary(
            'http_response_size_bytes',
            'HTTP response size'
        )
    
    def record_request(self, method, endpoint, status, duration, size):
        """Record request metrics"""
        self.request_count.labels(
            method=method,
            endpoint=endpoint,
            status=status
        ).inc()
        
        self.request_duration.labels(
            method=method,
            endpoint=endpoint
        ).observe(duration)
        
        self.response_size.observe(size)
    
    def increment_connections(self):
        self.active_connections.inc()
    
    def decrement_connections(self):
        self.active_connections.dec()

# Usage
metrics = MetricsCollector()

# Start metrics server (Prometheus scrapes this)
start_http_server(8000)

# Record metrics
def handle_request(method, endpoint):
    metrics.increment_connections()
    
    start_time = time.time()
    
    # Process request
    time.sleep(random.uniform(0.1, 0.5))
    
    duration = time.time() - start_time
    status = 200
    response_size = random.randint(1000, 10000)
    
    metrics.record_request(method, endpoint, status, duration, response_size)
    metrics.decrement_connections()

# Prometheus configuration (prometheus.yml)
"""
scrape_configs:
  - job_name: 'my-service'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:8000']
"""
```

### StatsD Metrics

```python
from statsd import StatsClient

class StatsDMetrics:
    def __init__(self, host='localhost', port=8125):
        self.statsd = StatsClient(host=host, port=port, prefix='myapp')
    
    def increment_counter(self, metric_name, value=1, **tags):
        """Increment a counter"""
        self.statsd.incr(metric_name, value)
    
    def record_timing(self, metric_name, value):
        """Record timing in milliseconds"""
        self.statsd.timing(metric_name, value)
    
    def set_gauge(self, metric_name, value):
        """Set gauge value"""
        self.statsd.gauge(metric_name, value)
    
    def time_function(self, metric_name):
        """Decorator to time function execution"""
        def decorator(func):
            def wrapper(*args, **kwargs):
                start = time.time()
                result = func(*args, **kwargs)
                duration = (time.time() - start) * 1000  # Convert to ms
                self.record_timing(metric_name, duration)
                return result
            return wrapper
        return decorator

# Usage
metrics = StatsDMetrics()

@metrics.time_function('api.user.get')
def get_user(user_id):
    # Fetch user
    time.sleep(0.1)
    return {'id': user_id, 'name': 'John'}

# Increment counter
metrics.increment_counter('api.requests', tags={'endpoint': '/users', 'method': 'GET'})

# Set gauge
metrics.set_gauge('queue.size', 150)
```

### Application Performance Monitoring (APM)

```python
class APMMetrics:
    """Track application performance"""
    
    def __init__(self):
        self.metrics = {
            'response_times': [],
            'error_rates': {},
            'throughput': 0,
            'apdex_score': 0.0  # Application Performance Index
        }
    
    def calculate_apdex(self, response_times, threshold=0.5):
        """
        Apdex Score = (Satisfied + Tolerating/2) / Total
        Satisfied: response_time <= threshold
        Tolerating: threshold < response_time <= 4*threshold
        Frustrated: response_time > 4*threshold
        """
        satisfied = sum(1 for t in response_times if t <= threshold)
        tolerating = sum(1 for t in response_times 
                        if threshold < t <= 4 * threshold)
        total = len(response_times)
        
        if total == 0:
            return 1.0
        
        return (satisfied + tolerating / 2) / total
    
    def calculate_percentiles(self, values):
        """Calculate p50, p95, p99"""
        sorted_values = sorted(values)
        n = len(sorted_values)
        
        if n == 0:
            return {}
        
        return {
            'p50': sorted_values[int(n * 0.50)],
            'p95': sorted_values[int(n * 0.95)],
            'p99': sorted_values[int(n * 0.99)],
            'max': sorted_values[-1],
            'min': sorted_values[0],
            'avg': sum(values) / n
        }

# Usage
apm = APMMetrics()

# Collect response times
response_times = [0.1, 0.2, 0.15, 0.8, 0.3, 0.5, 1.2, 0.4]

apdex = apm.calculate_apdex(response_times, threshold=0.5)
print(f"Apdex Score: {apdex}")  # 0.75 (good)

percentiles = apm.calculate_percentiles(response_times)
print(f"P95: {percentiles['p95']}s")
print(f"P99: {percentiles['p99']}s")
```

---

## Distributed Tracing

### OpenTelemetry Implementation

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.resources import Resource
import time

class DistributedTracing:
    def __init__(self, service_name):
        # Set up tracer
        resource = Resource.create({"service.name": service_name})
        provider = TracerProvider(resource=resource)
        
        # Configure Jaeger exporter
        jaeger_exporter = JaegerExporter(
            agent_host_name="localhost",
            agent_port=6831,
        )
        
        provider.add_span_processor(BatchSpanProcessor(jaeger_exporter))
        trace.set_tracer_provider(provider)
        
        self.tracer = trace.get_tracer(service_name)
    
    def trace_operation(self, operation_name):
        """Decorator to trace function"""
        def decorator(func):
            def wrapper(*args, **kwargs):
                with self.tracer.start_as_current_span(operation_name) as span:
                    # Add attributes
                    span.set_attribute("function", func.__name__)
                    span.set_attribute("args", str(args))
                    
                    try:
                        result = func(*args, **kwargs)
                        span.set_attribute("success", True)
                        return result
                    except Exception as e:
                        span.set_attribute("success", False)
                        span.set_attribute("error", str(e))
                        span.record_exception(e)
                        raise
            return wrapper
        return decorator

# Usage
tracing = DistributedTracing('order-service')

@tracing.trace_operation('create_order')
def create_order(user_id, items):
    with tracing.tracer.start_as_current_span('validate_user'):
        validate_user(user_id)
    
    with tracing.tracer.start_as_current_span('process_payment'):
        process_payment(items)
    
    with tracing.tracer.start_as_current_span('update_inventory'):
        update_inventory(items)
    
    return {'order_id': 'ORD-123', 'status': 'created'}

@tracing.trace_operation('validate_user')
def validate_user(user_id):
    time.sleep(0.05)
    return True

@tracing.trace_operation('process_payment')
def process_payment(items):
    time.sleep(0.1)
    return True

@tracing.trace_operation('update_inventory')
def update_inventory(items):
    time.sleep(0.08)
    return True

# Call creates trace with multiple spans
order = create_order(user_id=123, items=[{'id': 1, 'qty': 2}])
```

### Trace Context Propagation

```python
import requests
from opentelemetry.propagate import inject, extract

class TraceContextPropagation:
    def __init__(self, tracer):
        self.tracer = tracer
    
    def call_downstream_service(self, url, data):
        """Propagate trace context to downstream service"""
        with self.tracer.start_as_current_span('http_request') as span:
            # Create headers with trace context
            headers = {}
            inject(headers)  # Injects traceparent header
            
            span.set_attribute("http.url", url)
            span.set_attribute("http.method", "POST")
            
            # Make request with trace context
            response = requests.post(url, json=data, headers=headers)
            
            span.set_attribute("http.status_code", response.status_code)
            
            return response.json()
    
    def receive_request(self, request_headers):
        """Extract trace context from incoming request"""
        # Extract parent trace context
        ctx = extract(request_headers)
        
        # Start span with parent context
        with self.tracer.start_as_current_span('handle_request', context=ctx):
            # Process request
            pass
```

---

## Alerting

### Alert Rules

```python
class AlertManager:
    def __init__(self):
        self.alert_channels = []
    
    def add_channel(self, channel):
        self.alert_channels.append(channel)
    
    def check_threshold(self, metric_name, current_value, threshold, comparison='greater'):
        """Check if metric exceeds threshold"""
        triggered = False
        
        if comparison == 'greater' and current_value > threshold:
            triggered = True
        elif comparison == 'less' and current_value < threshold:
            triggered = True
        
        if triggered:
            self.send_alert(
                severity='critical',
                title=f'{metric_name} threshold exceeded',
                message=f'{metric_name} is {current_value} (threshold: {threshold})'
            )
    
    def check_error_rate(self, total_requests, failed_requests, threshold_percent=5):
        """Alert if error rate exceeds threshold"""
        if total_requests == 0:
            return
        
        error_rate = (failed_requests / total_requests) * 100
        
        if error_rate > threshold_percent:
            self.send_alert(
                severity='warning',
                title='High error rate detected',
                message=f'Error rate: {error_rate:.2f}% (threshold: {threshold_percent}%)'
            )
    
    def check_latency(self, response_times, threshold_ms=1000):
        """Alert if p95 latency exceeds threshold"""
        if not response_times:
            return
        
        sorted_times = sorted(response_times)
        p95 = sorted_times[int(len(sorted_times) * 0.95)] * 1000  # Convert to ms
        
        if p95 > threshold_ms:
            self.send_alert(
                severity='warning',
                title='High latency detected',
                message=f'P95 latency: {p95:.2f}ms (threshold: {threshold_ms}ms)'
            )
    
    def send_alert(self, severity, title, message):
        """Send alert to all configured channels"""
        alert = {
            'severity': severity,
            'title': title,
            'message': message,
            'timestamp': datetime.now().isoformat()
        }
        
        for channel in self.alert_channels:
            channel.send(alert)

# Alert Channels
class SlackAlertChannel:
    def __init__(self, webhook_url):
        self.webhook_url = webhook_url
    
    def send(self, alert):
        """Send alert to Slack"""
        payload = {
            'text': f"*{alert['severity'].upper()}*: {alert['title']}",
            'attachments': [{
                'text': alert['message'],
                'color': 'danger' if alert['severity'] == 'critical' else 'warning'
            }]
        }
        requests.post(self.webhook_url, json=payload)

class EmailAlertChannel:
    def __init__(self, smtp_config):
        self.smtp_config = smtp_config
    
    def send(self, alert):
        """Send alert via email"""
        # Email sending logic
        print(f"Sending email alert: {alert['title']}")

class PagerDutyChannel:
    def __init__(self, integration_key):
        self.integration_key = integration_key
    
    def send(self, alert):
        """Send alert to PagerDuty"""
        if alert['severity'] == 'critical':
            # Trigger PagerDuty incident
            print(f"PagerDuty incident: {alert['title']}")

# Usage
alert_manager = AlertManager()
alert_manager.add_channel(SlackAlertChannel('https://hooks.slack.com/...'))
alert_manager.add_channel(EmailAlertChannel({'host': 'smtp.gmail.com'}))

# Check metrics and alert
alert_manager.check_threshold('cpu_usage', 85, 80, 'greater')
alert_manager.check_error_rate(total_requests=1000, failed_requests=60, threshold_percent=5)
alert_manager.check_latency([0.1, 0.2, 0.5, 1.2, 0.8], threshold_ms=1000)
```

### Prometheus Alerting Rules

```yaml
# prometheus-alerts.yml
groups:
  - name: service_alerts
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) 
          / 
          sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"
      
      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, 
            rate(http_request_duration_seconds_bucket[5m])
          ) > 1.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "P95 latency is {{ $value }}s"
      
      # Service down
      - alert: ServiceDown
        expr: up{job="my-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service is down"
          description: "{{ $labels.instance }} is unreachable"
```

---

## Monitoring Tools

### Health Check Endpoint

```python
from flask import Flask, jsonify
import time

app = Flask(__name__)

class HealthCheck:
    def __init__(self):
        self.start_time = time.time()
        self.dependencies = {}
    
    def register_dependency(self, name, check_function):
        """Register health check for dependency"""
        self.dependencies[name] = check_function
    
    def check_health(self):
        """Check health of service and dependencies"""
        health_status = {
            'status': 'healthy',
            'uptime_seconds': int(time.time() - self.start_time),
            'dependencies': {}
        }
        
        all_healthy = True
        
        for name, check_func in self.dependencies.items():
            try:
                is_healthy = check_func()
                health_status['dependencies'][name] = {
                    'status': 'healthy' if is_healthy else 'unhealthy'
                }
                if not is_healthy:
                    all_healthy = False
            except Exception as e:
                health_status['dependencies'][name] = {
                    'status': 'unhealthy',
                    'error': str(e)
                }
                all_healthy = False
        
        if not all_healthy:
            health_status['status'] = 'degraded'
        
        return health_status

health_check = HealthCheck()

# Register dependency checks
def check_database():
    # Check database connection
    return True

def check_cache():
    # Check Redis connection
    return True

health_check.register_dependency('database', check_database)
health_check.register_dependency('cache', check_cache)

@app.route('/health')
def health():
    health_status = health_check.check_health()
    status_code = 200 if health_status['status'] == 'healthy' else 503
    return jsonify(health_status), status_code

@app.route('/readiness')
def readiness():
    """Kubernetes readiness probe"""
    # Check if service is ready to accept traffic
    return jsonify({'ready': True}), 200

@app.route('/liveness')
def liveness():
    """Kubernetes liveness probe"""
    # Check if service is alive (basic check)
    return jsonify({'alive': True}), 200
```

This comprehensive monitoring and observability guide covers all essential practices for production systems!
