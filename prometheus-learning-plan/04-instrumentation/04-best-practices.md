# Best Practices

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Metric Naming Conventions](#metric-naming-conventions)
- [Label Best Practices](#label-best-practices)
- [Cardinality Management](#cardinality-management)
- [Metric Types Selection](#metric-types-selection)
- [Instrumentation Patterns](#instrumentation-patterns)
- [Performance Considerations](#performance-considerations)
- [Testing và Validation](#testing-và-validation)
- [Common Mistakes](#common-mistakes)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)
- [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

## Giới Thiệu

Best practices cho instrumentation giúp bạn tạo ra metrics hữu ích, performant, và dễ maintain. Tuân theo các quy tắc này sẽ đảm bảo metrics của bạn hoạt động tốt với Prometheus ecosystem và dễ dàng query.

**Lợi ích của việc follow best practices:**
- Metrics dễ hiểu và query
- Performance tốt hơn (low cardinality)
- Consistency across services
- Easier troubleshooting
- Better alerting capabilities

## Metric Naming Conventions

### Cấu Trúc Tên Chuẩn

**Format:** `<namespace>_<subsystem>_<name>_<unit>`

```
myapp_http_requests_total
myapp_database_query_duration_seconds
myapp_cache_size_bytes
```

### Quy Tắc Đặt Tên

#### 1. Sử Dụng Snake Case

```
✅ GOOD
http_requests_total
database_connections_active
cache_hit_ratio

❌ BAD
httpRequestsTotal      # camelCase
http-requests-total    # kebab-case
HTTPRequestsTotal      # PascalCase
```

#### 2. Thêm Unit Suffix

**Base units được khuyến nghị:**

| Measurement | Unit | Example |
|-------------|------|---------|
| Time | `seconds` | `request_duration_seconds` |
| Size | `bytes` | `response_size_bytes` |
| Percentage | `ratio` (0-1) | `cpu_usage_ratio` |
| Temperature | `celsius` | `temperature_celsius` |

```
✅ GOOD
request_duration_seconds     # Not milliseconds
response_size_bytes          # Not kilobytes
cpu_usage_ratio              # 0-1, not percentage

❌ BAD
request_duration             # Missing unit
response_size_kb             # Use bytes
cpu_usage_percent            # Use ratio
```

#### 3. Counter Suffix _total

```
✅ GOOD
http_requests_total
errors_total
bytes_transmitted_total

❌ BAD
http_requests               # Ambiguous
error_count                 # Use _total
total_bytes_transmitted     # _total at end
```

#### 4. Tên Mô Tả "What" Không "How"

```
✅ GOOD
http_requests_total         # What is being measured
database_queries_total      # Clear and descriptive

❌ BAD
counter_1                   # Not descriptive
my_counter                  # Too generic
http_request_counter        # Describes implementation
```

#### 5. Namespace và Subsystem

**Namespace:** Application hoặc library name
**Subsystem:** Component hoặc module

```
# Format: <namespace>_<subsystem>_<name>_<unit>

myapp_http_requests_total
myapp_http_request_duration_seconds
myapp_database_connections_active
myapp_database_query_duration_seconds
myapp_cache_hits_total
myapp_cache_size_bytes
```

### Ví Dụ Naming Tốt

```python
from prometheus_client import Counter, Histogram, Gauge

# ✅ Counters with _total suffix
http_requests_total = Counter(
    'myapp_http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

api_errors_total = Counter(
    'myapp_api_errors_total',
    'Total API errors',
    ['error_type']
)

# ✅ Histograms with unit suffix
http_request_duration_seconds = Histogram(
    'myapp_http_request_duration_seconds',
    'HTTP request duration in seconds',
    ['method', 'endpoint']
)

response_size_bytes = Histogram(
    'myapp_response_size_bytes',
    'HTTP response size in bytes',
    ['endpoint']
)

# ✅ Gauges with descriptive names
database_connections_active = Gauge(
    'myapp_database_connections_active',
    'Number of active database connections',
    ['database']
)

cache_size_bytes = Gauge(
    'myapp_cache_size_bytes',
    'Size of cache in bytes',
    ['cache_name']
)

cpu_usage_ratio = Gauge(
    'myapp_cpu_usage_ratio',
    'CPU usage ratio (0-1)',
    ['core']
)
```

## Label Best Practices

### Nguyên Tắc Cơ Bản

**Labels enable filtering và aggregation**, nhưng mỗi unique combination tạo ra một time series mới.

### Quy Tắc Labels

#### 1. Low Cardinality

**Cardinality = số lượng unique time series**

```python
# ✅ GOOD - Low cardinality (limited values)
requests_total = Counter(
    'requests_total',
    'Total requests',
    ['method', 'endpoint', 'status']
)
# method: ~10 values (GET, POST, PUT, DELETE, ...)
# endpoint: ~50 values
# status: ~10 values (200, 400, 404, 500, ...)
# Total: 10 × 50 × 10 = 5,000 time series ✅

# ❌ BAD - High cardinality (unbounded values)
requests_total = Counter(
    'requests_total',
    'Total requests',
    ['user_id', 'session_id', 'request_id']
)
# user_id: millions of users
# session_id: unlimited sessions
# request_id: every request is unique
# Total: millions/billions of time series ❌
```

#### 2. Không Dùng Unbounded Values

```go
// ❌ BAD - Unbounded values
requests.WithLabelValues(
    userID,           // Millions of users
    sessionID,        // Unlimited sessions
    clientIP,         // Many IPs
    timestamp,        // Infinite values
).Inc()

// ✅ GOOD - Bounded values
requests.WithLabelValues(
    userTier,         // free, premium, enterprise
    region,           // us-east, us-west, eu-west
    clientType,       // web, mobile, api
).Inc()
```

#### 3. Label Names Snake Case

```yaml
✅ GOOD
http_method
status_code
error_type
user_tier
region_name

❌ BAD
httpMethod
StatusCode
error-type
UserTier
```

#### 4. Enable Aggregation

Labels nên cho phép aggregation theo nhiều dimensions:

```python
requests_total = Counter(
    'requests_total',
    'Total requests',
    ['method', 'endpoint', 'status', 'region']
)

# Có thể aggregate theo:
# - Method: sum by (method)
# - Endpoint: sum by (endpoint)
# - Status: sum by (status)
# - Region: sum by (region)
# - Method + Region: sum by (method, region)
# - Total: sum()
```

#### 5. Không Dùng Labels Cho High-Cardinality Data

```python
# ❌ BAD - Email addresses in labels
user_logins = Counter(
    'user_logins_total',
    'User logins',
    ['email']  # Millions of unique emails
)

# ✅ GOOD - Aggregate by domain
user_logins = Counter(
    'user_logins_total',
    'User logins',
    ['email_domain']  # gmail.com, yahoo.com, etc.
)

# ❌ BAD - Full URLs in labels
page_views = Counter(
    'page_views_total',
    'Page views',
    ['url']  # Unlimited URLs with query params
)

# ✅ GOOD - URL patterns
page_views = Counter(
    'page_views_total',
    'Page views',
    ['url_pattern']  # /users/:id, /products/:id
)
```

### Label Cardinality Examples

```python
# Example 1: Reasonable cardinality
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)
# method: 7 values (GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS)
# endpoint: 20 values
# status: 8 values (200, 201, 400, 401, 403, 404, 500, 503)
# Total cardinality: 7 × 20 × 8 = 1,120 time series ✅

# Example 2: High cardinality (BAD)
user_requests = Counter(
    'user_requests_total',
    'Requests per user',
    ['user_id', 'ip_address', 'user_agent']
)
# user_id: 1,000,000 users
# ip_address: 100,000 IPs
# user_agent: 1,000 agents
# Total cardinality: 1M × 100K × 1K = 100 trillion time series ❌

# Example 3: Reduced cardinality (GOOD)
user_requests = Counter(
    'user_requests_total',
    'Requests by user tier',
    ['user_tier', 'region', 'client_type']
)
# user_tier: 3 values (free, premium, enterprise)
# region: 5 values (us-east, us-west, eu-west, eu-east, ap-south)
# client_type: 3 values (web, mobile, api)
# Total cardinality: 3 × 5 × 3 = 45 time series ✅
```

## Cardinality Management

### Tính Toán Cardinality

**Formula:** Cardinality = Product of all label value counts

```
metric{label1, label2, label3}

Cardinality = |label1 values| × |label2 values| × |label3 values|
```

### Cardinality Limits

**Khuyến nghị:**

| Cardinality | Status | Action |
|-------------|--------|--------|
| < 1,000 | ✅ Good | No action needed |
| 1,000 - 10,000 | ⚠️ Warning | Review labels |
| 10,000 - 100,000 | ⚠️ High | Reduce labels |
| > 100,000 | ❌ Critical | Redesign metrics |

### Strategies để Giảm Cardinality

#### 1. Aggregate High-Cardinality Dimensions

```python
# ❌ BAD - User ID (millions)
requests_by_user = Counter(
    'requests_by_user',
    'Requests per user',
    ['user_id']
)

# ✅ GOOD - User tier (3-5 values)
requests_by_tier = Counter(
    'requests_by_tier',
    'Requests by user tier',
    ['user_tier']  # free, premium, enterprise
)
```

#### 2. Use URL Patterns Instead of Full URLs

```python
# ❌ BAD - Full URLs with query params
page_views = Counter(
    'page_views_total',
    'Page views',
    ['url']  # /products?id=123&sort=price&page=5
)

# ✅ GOOD - URL patterns
page_views = Counter(
    'page_views_total',
    'Page views',
    ['url_pattern']  # /products
)
```

#### 3. Drop Unnecessary Labels

```python
# ❌ BAD - Too many labels
requests = Counter(
    'requests_total',
    'Requests',
    ['method', 'endpoint', 'status', 'user_agent', 'referer', 'ip']
)

# ✅ GOOD - Essential labels only
requests = Counter(
    'requests_total',
    'Requests',
    ['method', 'endpoint', 'status']
)
```

#### 4. Use Separate Metrics for Different Cardinalities

```python
# ✅ GOOD - Separate metrics
# Low cardinality metric for general monitoring
requests_total = Counter(
    'requests_total',
    'Total requests',
    ['method', 'status']
)

# High cardinality metric for specific debugging (short retention)
requests_by_endpoint = Counter(
    'requests_by_endpoint_total',
    'Requests by endpoint',
    ['method', 'endpoint', 'status']
)
```

### Monitoring Cardinality

**Query để check cardinality:**

```promql
# Total time series count
count({__name__=~".+"})

# Time series per metric
count by (__name__) ({__name__=~".+"})

# Time series per job
count by (job) ({__name__=~".+"})

# Top metrics by cardinality
topk(10, count by (__name__) ({__name__=~".+"}))
```

## Metric Types Selection

### Khi Nào Dùng Counter

**Use cases:**
- Events xảy ra (requests, errors, tasks completed)
- Cumulative values (bytes transmitted, messages sent)

**Characteristics:**
- Chỉ tăng (hoặc reset về 0 khi restart)
- Query với `rate()` hoặc `increase()`

```python
# ✅ GOOD use cases for Counter
http_requests_total = Counter('http_requests_total', 'Total requests')
errors_total = Counter('errors_total', 'Total errors')
bytes_transmitted_total = Counter('bytes_transmitted_total', 'Bytes sent')
tasks_completed_total = Counter('tasks_completed_total', 'Tasks completed')
```

### Khi Nào Dùng Gauge

**Use cases:**
- Current state (temperature, memory usage)
- Values có thể tăng hoặc giảm
- Snapshots (queue depth, active connections)

**Characteristics:**
- Có thể tăng, giảm, hoặc set
- Query trực tiếp hoặc với `avg_over_time()`

```python
# ✅ GOOD use cases for Gauge
temperature_celsius = Gauge('temperature_celsius', 'Temperature')
memory_usage_bytes = Gauge('memory_usage_bytes', 'Memory usage')
queue_depth = Gauge('queue_depth', 'Queue depth')
active_connections = Gauge('active_connections', 'Active connections')
```

### Khi Nào Dùng Histogram

**Use cases:**
- Request durations
- Response sizes
- Bất kỳ measurement cần phân tích distribution

**Characteristics:**
- Tự động tạo buckets
- Có thể tính quantiles với `histogram_quantile()`
- Có thể aggregate qua instances

```python
# ✅ GOOD use cases for Histogram
request_duration_seconds = Histogram(
    'request_duration_seconds',
    'Request duration',
    buckets=[0.01, 0.05, 0.1, 0.5, 1, 2, 5, 10]
)

response_size_bytes = Histogram(
    'response_size_bytes',
    'Response size',
    buckets=[100, 1000, 10000, 100000, 1000000]
)
```

**Chọn buckets phù hợp:**

```python
# For latency (seconds)
buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]

# For response size (bytes)
buckets=[100, 1000, 10000, 100000, 1000000, 10000000]

# For custom ranges
buckets=[1, 5, 10, 50, 100, 500, 1000]
```

### Khi Nào Dùng Summary

**Use cases:**
- Khi cần quantiles chính xác
- Không cần aggregate qua instances
- Client-side quantile calculation

**Lưu ý:** Histogram thường được prefer hơn Summary

```python
# Summary example (use sparingly)
request_latency = Summary(
    'request_latency_seconds',
    'Request latency'
)
```

**Histogram vs Summary:**

| Feature | Histogram | Summary |
|---------|-----------|---------|
| Quantiles | Approximate | Exact |
| Aggregation | ✅ Yes | ❌ No |
| Performance | Better | Worse |
| Recommendation | ✅ Preferred | Use rarely |

## Instrumentation Patterns

### Pattern 1: Request/Response Tracking

```python
from prometheus_client import Counter, Histogram
import time

requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

request_duration_seconds = Histogram(
    'http_request_duration_seconds',
    'Request duration',
    ['method', 'endpoint']
)

def track_request(method, endpoint):
    def decorator(func):
        def wrapper(*args, **kwargs):
            start = time.time()
            status = 200
            
            try:
                result = func(*args, **kwargs)
                return result
            except Exception as e:
                status = 500
                raise
            finally:
                duration = time.time() - start
                request_duration_seconds.labels(
                    method=method,
                    endpoint=endpoint
                ).observe(duration)
                requests_total.labels(
                    method=method,
                    endpoint=endpoint,
                    status=status
                ).inc()
        
        return wrapper
    return decorator

@track_request('GET', '/api/users')
def get_users():
    return fetch_users()
```

### Pattern 2: In-Progress Tracking

```python
from prometheus_client import Gauge

in_progress_requests = Gauge(
    'requests_in_progress',
    'Requests currently being processed',
    ['endpoint']
)

def track_in_progress(endpoint):
    def decorator(func):
        def wrapper(*args, **kwargs):
            in_progress_requests.labels(endpoint=endpoint).inc()
            try:
                return func(*args, **kwargs)
            finally:
                in_progress_requests.labels(endpoint=endpoint).dec()
        return wrapper
    return decorator

@track_in_progress('/api/users')
def get_users():
    return fetch_users()
```

### Pattern 3: Error Rate Tracking

```python
from prometheus_client import Counter

operations_total = Counter(
    'operations_total',
    'Total operations',
    ['operation', 'status']
)

def track_operation(operation_name):
    def decorator(func):
        def wrapper(*args, **kwargs):
            try:
                result = func(*args, **kwargs)
                operations_total.labels(
                    operation=operation_name,
                    status='success'
                ).inc()
                return result
            except Exception as e:
                operations_total.labels(
                    operation=operation_name,
                    status='error'
                ).inc()
                raise
        return wrapper
    return decorator

@track_operation('database_query')
def query_database():
    return db.execute(query)
```

## Performance Considerations

### 1. Minimize Label Cardinality

```python
# ❌ BAD - High cardinality
requests.labels(user_id=user.id).inc()  # Millions of users

# ✅ GOOD - Low cardinality
requests.labels(user_tier=user.tier).inc()  # 3-5 tiers
```

### 2. Reuse Metric Objects

```python
# ✅ GOOD - Create once, reuse
requests_total = Counter('requests_total', 'Requests', ['endpoint'])

def handle_request(endpoint):
    requests_total.labels(endpoint=endpoint).inc()

# ❌ BAD - Creating new metrics repeatedly
def handle_request(endpoint):
    Counter('requests_total', 'Requests', ['endpoint']).labels(endpoint=endpoint).inc()
```

### 3. Batch Updates When Possible

```python
# ✅ GOOD - Batch increment
counter.inc(batch_size)

# ❌ BAD - Individual increments in loop
for item in items:
    counter.inc()
```

### 4. Use Appropriate Histogram Buckets

```python
# ✅ GOOD - Reasonable bucket count
buckets=[0.01, 0.05, 0.1, 0.5, 1, 5, 10]  # 7 buckets

# ❌ BAD - Too many buckets
buckets=[0.001, 0.002, 0.003, ..., 10]  # 100+ buckets
```

## Testing và Validation

### Unit Testing Metrics

```python
import pytest
from prometheus_client import REGISTRY

def test_counter_increments():
    # Get initial value
    before = requests_total.labels(method='GET', endpoint='/api/users', status='200')._value.get()
    
    # Perform action
    handle_request('GET', '/api/users')
    
    # Verify increment
    after = requests_total.labels(method='GET', endpoint='/api/users', status='200')._value.get()
    assert after == before + 1

def test_histogram_observes():
    # Perform action
    with request_duration_seconds.labels(method='GET', endpoint='/api/users').time():
        time.sleep(0.1)
    
    # Verify observation
    metric = request_duration_seconds.labels(method='GET', endpoint='/api/users')
    assert metric._sum.get() > 0
    assert metric._count.get() > 0
```

### Validate Metric Names

```python
import re

def validate_metric_name(name):
    # Must match: [a-zA-Z_:][a-zA-Z0-9_:]*
    pattern = r'^[a-zA-Z_:][a-zA-Z0-9_:]*$'
    assert re.match(pattern, name), f"Invalid metric name: {name}"
    
    # Should have unit suffix
    if '_total' not in name and '_seconds' not in name and '_bytes' not in name:
        print(f"Warning: {name} missing unit suffix")

validate_metric_name('http_requests_total')  # ✅
validate_metric_name('request_duration_seconds')  # ✅
validate_metric_name('invalid-name')  # ❌ Fails
```

## Common Mistakes

### ❌ Mistake 1: Using Gauge for Counters

```python
# ❌ BAD
request_count = Gauge('request_count', 'Request count')
request_count.inc()

# ✅ GOOD
requests_total = Counter('requests_total', 'Total requests')
requests_total.inc()
```

### ❌ Mistake 2: High Cardinality Labels

```python
# ❌ BAD
requests.labels(user_id=user_id, ip=ip_address).inc()

# ✅ GOOD
requests.labels(user_tier=user_tier, region=region).inc()
```

### ❌ Mistake 3: Missing Units

```python
# ❌ BAD
duration = Histogram('request_duration', 'Duration')

# ✅ GOOD
duration_seconds = Histogram('request_duration_seconds', 'Duration in seconds')
```

### ❌ Mistake 4: Inconsistent Naming

```python
# ❌ BAD
http_requests = Counter('httpRequests', 'Requests')
api_calls = Counter('api-calls', 'API calls')

# ✅ GOOD
http_requests_total = Counter('http_requests_total', 'HTTP requests')
api_calls_total = Counter('api_calls_total', 'API calls')
```

### ❌ Mistake 5: Not Using _total for Counters

```python
# ❌ BAD
requests = Counter('requests', 'Requests')

# ✅ GOOD
requests_total = Counter('requests_total', 'Total requests')
```

## Tài Liệu Liên Quan

- [Client Libraries](./01-client-libraries.md) - Prometheus client libraries
- [Custom Metrics](./02-custom-metrics.md) - Tạo custom metrics
- [Metrics Types](../01-fundamentals/03-metrics-types.md) - Chi tiết về metric types
- [Best Practices Checklist](../09-reference/05-best-practices-checklist.md) - Checklist tổng hợp

## Tài Liệu Tham Khảo

- [Metric and Label Naming](https://prometheus.io/docs/practices/naming/) - Official naming conventions
- [Instrumentation](https://prometheus.io/docs/practices/instrumentation/) - Instrumentation best practices
- [Histograms and Summaries](https://prometheus.io/docs/practices/histograms/) - When to use each
- [Cardinality](https://prometheus.io/docs/practices/naming/#labels) - Managing cardinality
