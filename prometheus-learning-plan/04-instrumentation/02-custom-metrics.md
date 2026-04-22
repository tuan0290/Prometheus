# Custom Metrics

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Khi Nào Tạo Custom Metrics](#khi-nào-tạo-custom-metrics)
- [Chọn Metric Type Phù Hợp](#chọn-metric-type-phù-hợp)
- [Metric Naming Conventions](#metric-naming-conventions)
- [Label Best Practices](#label-best-practices)
- [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
- [Anti-Patterns Cần Tránh](#anti-patterns-cần-tránh)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)
- [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

## Giới Thiệu

Custom metrics cho phép bạn đo lường các khía cạnh cụ thể của application mà standard exporters không cung cấp. Việc tạo custom metrics đúng cách giúp bạn có được observability sâu vào business logic và application behavior.

**Lợi ích của custom metrics:**
- Đo lường business metrics (orders, revenue, user signups)
- Theo dõi application-specific behavior
- Monitoring internal queues và processing pipelines
- Tracking feature usage và user behavior
- Measuring SLIs (Service Level Indicators)

## Khi Nào Tạo Custom Metrics

### Nên Tạo Custom Metrics Khi:

✅ **Business Metrics**
- Số lượng orders được xử lý
- Revenue generated
- User registrations
- Feature adoption rates

✅ **Application Performance**
- Cache hit/miss rates
- Queue depths và processing times
- Database query performance
- External API call latencies

✅ **Resource Utilization**
- Connection pool usage
- Thread pool statistics
- Memory cache sizes
- Custom resource limits

✅ **SLI/SLO Tracking**
- Request success rates
- Latency percentiles
- Availability metrics
- Error budgets

### Không Nên Tạo Custom Metrics Khi:

❌ **Standard Exporters Đã Cung Cấp**
- System metrics (CPU, memory, disk) → Dùng node_exporter
- HTTP server metrics → Dùng built-in instrumentation
- Database metrics → Dùng database exporters

❌ **Logs Phù Hợp Hơn**
- Detailed error messages
- Debug information
- Audit trails
- Individual transaction details

❌ **Traces Phù Hợp Hơn**
- Request flow qua nhiều services
- Distributed transaction tracking
- Detailed timing breakdowns

## Chọn Metric Type Phù Hợp

### Counter - Đếm Sự Kiện

**Sử dụng khi:** Đếm events xảy ra, giá trị chỉ tăng

**Ví dụ:**
```python
from prometheus_client import Counter

# Orders processed
orders_total = Counter(
    'orders_processed_total',
    'Total number of orders processed',
    ['status', 'payment_method']
)

def process_order(order):
    try:
        # Process order logic
        result = handle_payment(order)
        orders_total.labels(status='success', payment_method=order.payment_method).inc()
    except Exception as e:
        orders_total.labels(status='failed', payment_method=order.payment_method).inc()
        raise
```

**Query patterns:**
```promql
# Rate of successful orders per second
rate(orders_processed_total{status="success"}[5m])

# Percentage of failed orders
rate(orders_processed_total{status="failed"}[5m]) 
  / 
rate(orders_processed_total[5m]) * 100
```

### Gauge - Giá Trị Hiện Tại

**Sử dụng khi:** Đo giá trị có thể tăng hoặc giảm

**Ví dụ:**
```go
package main

import (
    "github.com/prometheus/client_golang/prometheus"
)

var (
    queueDepth = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "queue_depth",
            Help: "Current depth of processing queue",
        },
        []string{"queue_name"},
    )
    
    cacheSize = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "cache_size_bytes",
            Help: "Current size of cache in bytes",
        },
    )
)

func init() {
    prometheus.MustRegister(queueDepth, cacheSize)
}

func updateQueueMetrics(queueName string, depth int) {
    queueDepth.WithLabelValues(queueName).Set(float64(depth))
}

func updateCacheSize(size int64) {
    cacheSize.Set(float64(size))
}
```

**Query patterns:**
```promql
# Current queue depth
queue_depth{queue_name="orders"}

# Average queue depth over time
avg_over_time(queue_depth{queue_name="orders"}[5m])

# Alert when queue is too deep
queue_depth > 1000
```

### Histogram - Phân Phối Observations

**Sử dụng khi:** Đo duration, size, hoặc bất kỳ giá trị nào cần phân tích phân phối

**Ví dụ:**
```java
import io.prometheus.client.Histogram;

public class OrderService {
    private static final Histogram orderProcessingDuration = Histogram.build()
        .name("order_processing_duration_seconds")
        .help("Time to process an order")
        .labelNames("order_type")
        .buckets(0.1, 0.5, 1, 2, 5, 10, 30)
        .register();
    
    private static final Histogram orderValue = Histogram.build()
        .name("order_value_dollars")
        .help("Order value in dollars")
        .buckets(10, 50, 100, 500, 1000, 5000)
        .register();
    
    public void processOrder(Order order) {
        Histogram.Timer timer = orderProcessingDuration
            .labels(order.getType())
            .startTimer();
        
        try {
            // Process order
            handleOrder(order);
            
            // Record order value
            orderValue.observe(order.getTotalValue());
        } finally {
            timer.observeDuration();
        }
    }
}
```

**Query patterns:**
```promql
# 95th percentile processing time
histogram_quantile(0.95, 
  rate(order_processing_duration_seconds_bucket[5m])
)

# Average order value
rate(order_value_dollars_sum[5m]) 
  / 
rate(order_value_dollars_count[5m])

# Percentage of orders processed under 1 second
sum(rate(order_processing_duration_seconds_bucket{le="1"}[5m]))
  /
sum(rate(order_processing_duration_seconds_count[5m]))
```

### Summary - Quantiles Trực Tiếp

**Sử dụng khi:** Cần quantiles chính xác nhưng không cần aggregation qua instances

**Ví dụ:**
```python
from prometheus_client import Summary

api_latency = Summary(
    'api_request_latency_seconds',
    'API request latency',
    ['endpoint', 'method']
)

def handle_api_request(endpoint, method):
    with api_latency.labels(endpoint=endpoint, method=method).time():
        # Handle request
        return process_request(endpoint, method)
```

**Lưu ý:** Histogram thường được ưu tiên hơn Summary vì có thể aggregate qua nhiều instances.

## Metric Naming Conventions

### Cấu Trúc Tên Metric

**Format:** `<namespace>_<subsystem>_<name>_<unit>`

**Ví dụ:**
```
myapp_http_requests_total
myapp_database_query_duration_seconds
myapp_cache_size_bytes
myapp_queue_depth_messages
```

### Quy Tắc Đặt Tên

**1. Sử dụng snake_case**
```
✅ http_requests_total
❌ httpRequestsTotal
❌ http-requests-total
```

**2. Thêm unit suffix**
```
✅ request_duration_seconds
✅ response_size_bytes
✅ temperature_celsius
❌ request_duration (không rõ unit)
```

**3. Counter phải có suffix _total**
```
✅ http_requests_total
✅ errors_total
❌ http_requests (không rõ là counter)
```

**4. Sử dụng base units**
```
✅ duration_seconds (không phải milliseconds)
✅ size_bytes (không phải kilobytes)
✅ ratio (0-1, không phải percentage)
```

**5. Tên phải mô tả "what" không phải "how"**
```
✅ http_requests_total
❌ counter_of_http_requests
```

### Ví Dụ Naming Tốt

```python
from prometheus_client import Counter, Histogram, Gauge

# Counter - events that happen
http_requests_total = Counter(
    'myapp_http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

# Histogram - measurements
http_request_duration_seconds = Histogram(
    'myapp_http_request_duration_seconds',
    'HTTP request duration in seconds',
    ['method', 'endpoint']
)

# Gauge - current state
database_connections_active = Gauge(
    'myapp_database_connections_active',
    'Number of active database connections',
    ['database']
)

# Gauge - size
cache_size_bytes = Gauge(
    'myapp_cache_size_bytes',
    'Size of cache in bytes',
    ['cache_name']
)
```

## Label Best Practices

### Nguyên Tắc Sử Dụng Labels

**Labels cho phép filtering và aggregation**, nhưng mỗi unique label combination tạo ra một time series mới.

### Quy Tắc Labels

**1. Sử dụng labels cho dimensions có cardinality thấp**

```python
# ✅ GOOD - Low cardinality
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']  # Limited values
)

# ❌ BAD - High cardinality
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['user_id', 'session_id']  # Unlimited values
)
```

**2. Không đưa unbounded values vào labels**

```go
// ❌ BAD - User ID changes for every user
requests.WithLabelValues(userID, endpoint).Inc()

// ✅ GOOD - User tier has limited values
requests.WithLabelValues(userTier, endpoint).Inc()
```

**3. Label names nên là snake_case**

```yaml
✅ http_method, status_code, error_type
❌ httpMethod, StatusCode, error-type
```

**4. Sử dụng labels để enable aggregation**

```python
# With proper labels, you can aggregate:
# - By endpoint: sum by (endpoint)
# - By method: sum by (method)
# - By status: sum by (status)
# - Total: sum()

requests_total = Counter(
    'requests_total',
    'Total requests',
    ['method', 'endpoint', 'status']
)
```

### Cardinality Management

**Cardinality = số lượng unique time series**

**Ví dụ:**
```python
# Metric với 3 labels:
# - method: 4 values (GET, POST, PUT, DELETE)
# - endpoint: 10 values
# - status: 5 values (200, 400, 404, 500, 503)
# 
# Total cardinality = 4 × 10 × 5 = 200 time series
```

**Tránh high cardinality:**

```python
# ❌ BAD - Unbounded cardinality
user_requests = Counter(
    'user_requests_total',
    'Requests per user',
    ['user_id']  # Millions of users = millions of time series
)

# ✅ GOOD - Bounded cardinality
user_requests = Counter(
    'user_requests_total',
    'Requests by user tier',
    ['user_tier']  # Only: free, premium, enterprise
)
```

## Ví Dụ Thực Tế

### E-commerce Application

```go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    // Business metrics
    OrdersTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "ecommerce_orders_total",
            Help: "Total number of orders",
        },
        []string{"status", "payment_method"},
    )
    
    OrderValue = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "ecommerce_order_value_dollars",
            Help:    "Order value in dollars",
            Buckets: []float64{10, 25, 50, 100, 250, 500, 1000, 2500},
        },
        []string{"category"},
    )
    
    // Performance metrics
    CheckoutDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "ecommerce_checkout_duration_seconds",
            Help:    "Time to complete checkout",
            Buckets: prometheus.DefBuckets,
        },
        []string{"payment_method"},
    )
    
    // Inventory metrics
    InventoryLevel = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "ecommerce_inventory_items",
            Help: "Current inventory level",
        },
        []string{"product_category", "warehouse"},
    )
    
    // Cart metrics
    ActiveCarts = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "ecommerce_active_carts",
            Help: "Number of active shopping carts",
        },
    )
)
```

### API Service

```python
from prometheus_client import Counter, Histogram, Gauge
import time

# Request metrics
api_requests_total = Counter(
    'api_requests_total',
    'Total API requests',
    ['method', 'endpoint', 'status']
)

api_request_duration_seconds = Histogram(
    'api_request_duration_seconds',
    'API request duration',
    ['method', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.5, 1, 2, 5]
)

# Rate limiting
api_rate_limit_exceeded_total = Counter(
    'api_rate_limit_exceeded_total',
    'Number of rate limit exceeded errors',
    ['endpoint', 'client_tier']
)

# External dependencies
external_api_calls_total = Counter(
    'external_api_calls_total',
    'Calls to external APIs',
    ['service', 'status']
)

external_api_duration_seconds = Histogram(
    'external_api_duration_seconds',
    'External API call duration',
    ['service']
)

# Resource usage
active_requests = Gauge(
    'api_active_requests',
    'Number of requests currently being processed'
)

def track_request(method, endpoint):
    """Decorator to track API requests"""
    def decorator(func):
        def wrapper(*args, **kwargs):
            active_requests.inc()
            start = time.time()
            
            try:
                result = func(*args, **kwargs)
                status = 200
                return result
            except Exception as e:
                status = 500
                raise
            finally:
                duration = time.time() - start
                api_request_duration_seconds.labels(
                    method=method,
                    endpoint=endpoint
                ).observe(duration)
                api_requests_total.labels(
                    method=method,
                    endpoint=endpoint,
                    status=status
                ).inc()
                active_requests.dec()
        
        return wrapper
    return decorator

# Usage
@track_request('GET', '/api/users')
def get_users():
    return fetch_users()
```

### Background Job Processor

```java
import io.prometheus.client.*;

public class JobProcessor {
    private static final Counter jobsProcessed = Counter.build()
        .name("jobs_processed_total")
        .help("Total jobs processed")
        .labelNames("job_type", "status")
        .register();
    
    private static final Histogram jobDuration = Histogram.build()
        .name("job_duration_seconds")
        .help("Job processing duration")
        .labelNames("job_type")
        .register();
    
    private static final Gauge queueSize = Gauge.build()
        .name("job_queue_size")
        .help("Number of jobs in queue")
        .labelNames("job_type")
        .register();
    
    private static final Gauge activeJobs = Gauge.build()
        .name("jobs_active")
        .help("Number of jobs currently processing")
        .labelNames("job_type")
        .register();
    
    public void processJob(Job job) {
        String jobType = job.getType();
        
        activeJobs.labels(jobType).inc();
        Histogram.Timer timer = jobDuration.labels(jobType).startTimer();
        
        try {
            // Process job
            executeJob(job);
            jobsProcessed.labels(jobType, "success").inc();
        } catch (Exception e) {
            jobsProcessed.labels(jobType, "failed").inc();
            throw e;
        } finally {
            timer.observeDuration();
            activeJobs.labels(jobType).dec();
            updateQueueSize(jobType);
        }
    }
    
    private void updateQueueSize(String jobType) {
        int size = getQueueSize(jobType);
        queueSize.labels(jobType).set(size);
    }
}
```

## Anti-Patterns Cần Tránh

### ❌ High Cardinality Labels

```python
# BAD - User ID creates millions of time series
requests_by_user = Counter(
    'requests_by_user',
    'Requests per user',
    ['user_id']  # DON'T DO THIS
)

# GOOD - Use aggregated dimensions
requests_by_tier = Counter(
    'requests_by_tier',
    'Requests by user tier',
    ['user_tier']  # free, premium, enterprise
)
```

### ❌ Timestamps in Labels

```python
# BAD - Timestamp creates infinite cardinality
events = Counter(
    'events_total',
    'Events',
    ['timestamp']  # DON'T DO THIS
)

# GOOD - Let Prometheus handle timestamps
events = Counter(
    'events_total',
    'Events',
    ['event_type']
)
```

### ❌ Using Gauge for Counters

```python
# BAD - Gauge for counting events
requests_count = Gauge('requests_count', 'Request count')
requests_count.inc()  # Wrong metric type

# GOOD - Counter for counting events
requests_total = Counter('requests_total', 'Total requests')
requests_total.inc()
```

### ❌ Missing Units

```python
# BAD - No unit specified
request_duration = Histogram('request_duration', 'Request duration')

# GOOD - Unit in name
request_duration_seconds = Histogram(
    'request_duration_seconds',
    'Request duration in seconds'
)
```

### ❌ Inconsistent Naming

```python
# BAD - Inconsistent naming
http_requests = Counter('httpRequests', 'HTTP requests')
api_calls = Counter('api-calls', 'API calls')

# GOOD - Consistent snake_case
http_requests_total = Counter('http_requests_total', 'HTTP requests')
api_calls_total = Counter('api_calls_total', 'API calls')
```

## Tài Liệu Liên Quan

- [Client Libraries](./01-client-libraries.md) - Prometheus client libraries
- [Best Practices](./04-best-practices.md) - Naming và cardinality best practices
- [Metrics Types](../01-fundamentals/03-metrics-types.md) - Chi tiết về metric types
- [Lab 02 - Custom Metrics](../07-labs/lab-02-custom-metrics/README.md) - Bài tập thực hành

## Tài Liệu Tham Khảo

- [Metric and Label Naming](https://prometheus.io/docs/practices/naming/) - Official naming conventions
- [Instrumentation](https://prometheus.io/docs/practices/instrumentation/) - Instrumentation best practices
- [Histograms and Summaries](https://prometheus.io/docs/practices/histograms/) - When to use each type
