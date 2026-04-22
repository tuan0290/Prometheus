# Lab 02 - Custom Metrics Instructions

## Mục Tiêu

Trong lab này, bạn sẽ tạo một Python application expose custom Prometheus metrics sử dụng prometheus_client library. Bạn sẽ implement các metric types: Counter, Gauge, Histogram, và Summary.

## Yêu Cầu

- Lab 01 đã hoàn thành (Prometheus running)
- Python 3.7+ installed
- pip package manager
- Text editor (vim, nano, hoặc IDE)

## Phần 1: Setup Environment

### Bước 1: Install Dependencies

```bash
# Install Python và pip (nếu chưa có)
# Ubuntu/Debian
sudo apt update
sudo apt install -y python3 python3-pip

# CentOS/RHEL
sudo yum install -y python3 python3-pip

# Install Prometheus client library
pip3 install prometheus_client flask
```

### Bước 2: Tạo Project Directory

```bash
mkdir -p ~/prometheus-labs/custom-metrics
cd ~/prometheus-labs/custom-metrics
```

## Phần 2: Create Simple Metrics Application

### Bước 1: Create Basic App với Counter

Tạo file `app.py`:

```python
from flask import Flask
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)

# Define a Counter metric
REQUEST_COUNT = Counter(
    'app_requests_total',
    'Total number of requests',
    ['method', 'endpoint', 'status']
)

@app.route('/')
def index():
    REQUEST_COUNT.labels(method='GET', endpoint='/', status='200').inc()
    return 'Hello from Prometheus Lab!'

@app.route('/metrics')
def metrics():
    return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### Bước 2: Run Application

```bash
python3 app.py
```

Application sẽ chạy trên port 8080.

### Bước 3: Test Metrics Endpoint

Trong terminal khác:

```bash
# Generate some requests
curl http://localhost:8080/
curl http://localhost:8080/
curl http://localhost:8080/

# Check metrics
curl http://localhost:8080/metrics
```

Bạn sẽ thấy metric:
```
# HELP app_requests_total Total number of requests
# TYPE app_requests_total counter
app_requests_total{endpoint="/",method="GET",status="200"} 3.0
```

## Phần 3: Add More Metric Types

### Bước 1: Expand Application với Gauge, Histogram, Summary

Update `app.py`:

```python
from flask import Flask, request
from prometheus_client import (
    Counter, Gauge, Histogram, Summary,
    generate_latest, CONTENT_TYPE_LATEST
)
import time
import random
import threading

app = Flask(__name__)

# ─── Metrics Definitions ───────────────────────────────────────

# Counter: Monotonically increasing value
REQUEST_COUNT = Counter(
    'app_requests_total',
    'Total number of requests',
    ['method', 'endpoint', 'status']
)

# Gauge: Value that can go up or down
ACTIVE_USERS = Gauge(
    'app_active_users',
    'Number of active users'
)

QUEUE_SIZE = Gauge(
    'app_queue_size',
    'Current job queue size'
)

# Histogram: Observations in buckets
REQUEST_LATENCY = Histogram(
    'app_request_duration_seconds',
    'HTTP request latency in seconds',
    ['endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

# Summary: Similar to histogram but calculates quantiles
PROCESS_TIME = Summary(
    'app_process_time_seconds',
    'Time spent processing a job'
)

# ─── Background Simulator ──────────────────────────────────────

def simulate_metrics():
    """Background thread to simulate changing metrics"""
    while True:
        # Simulate active users (gauge)
        ACTIVE_USERS.set(random.randint(10, 200))
        
        # Simulate queue size (gauge)
        QUEUE_SIZE.set(random.randint(0, 50))
        
        time.sleep(5)

# Start background thread
threading.Thread(target=simulate_metrics, daemon=True).start()

# ─── Routes ────────────────────────────────────────────────────

@app.route('/')
def index():
    REQUEST_COUNT.labels(method='GET', endpoint='/', status='200').inc()
    return '''
    <h1>Prometheus Custom Metrics Lab</h1>
    <p>Available endpoints:</p>
    <ul>
        <li><a href="/metrics">/metrics</a> - Prometheus metrics</li>
        <li><a href="/work">/work</a> - Simulate work with latency</li>
        <li><a href="/process">/process</a> - Simulate job processing</li>
        <li><a href="/error">/error</a> - Trigger error</li>
    </ul>
    '''

@app.route('/work')
def work():
    # Measure request latency with histogram
    start = time.time()
    
    # Simulate work
    work_duration = random.uniform(0.01, 0.5)
    time.sleep(work_duration)
    
    # Record latency
    REQUEST_LATENCY.labels(endpoint='/work').observe(time.time() - start)
    REQUEST_COUNT.labels(method='GET', endpoint='/work', status='200').inc()
    
    return f'Work completed in {work_duration:.3f} seconds'

@app.route('/process')
def process():
    # Measure processing time with summary
    with PROCESS_TIME.time():
        # Simulate processing
        time.sleep(random.uniform(0.1, 1.0))
    
    REQUEST_COUNT.labels(method='GET', endpoint='/process', status='200').inc()
    return 'Job processed'

@app.route('/error')
def error():
    REQUEST_COUNT.labels(method='GET', endpoint='/error', status='500').inc()
    return 'Internal Server Error', 500

@app.route('/metrics')
def metrics():
    return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}

if __name__ == '__main__':
    print('Starting Prometheus Custom Metrics App on port 8080...')
    print('Metrics available at http://localhost:8080/metrics')
    app.run(host='0.0.0.0', port=8080)
```

### Bước 2: Run Updated Application

```bash
# Stop previous app (Ctrl+C)
# Run new version
python3 app.py
```

### Bước 3: Generate Traffic

Trong terminal khác, generate traffic:

```bash
# Generate various requests
for i in {1..10}; do
    curl -s http://localhost:8080/ > /dev/null
    curl -s http://localhost:8080/work > /dev/null
    curl -s http://localhost:8080/process > /dev/null
    curl -s http://localhost:8080/error > /dev/null
    sleep 1
done
```

### Bước 4: Inspect Metrics

```bash
curl http://localhost:8080/metrics
```

Bạn sẽ thấy các metrics:

```
# Counter
app_requests_total{endpoint="/",method="GET",status="200"} 10.0
app_requests_total{endpoint="/work",method="GET",status="200"} 10.0
app_requests_total{endpoint="/error",method="GET",status="500"} 10.0

# Gauge
app_active_users 145.0
app_queue_size 23.0

# Histogram
app_request_duration_seconds_bucket{endpoint="/work",le="0.01"} 0.0
app_request_duration_seconds_bucket{endpoint="/work",le="0.05"} 2.0
app_request_duration_seconds_bucket{endpoint="/work",le="0.1"} 5.0
app_request_duration_seconds_bucket{endpoint="/work",le="+Inf"} 10.0
app_request_duration_seconds_sum{endpoint="/work"} 2.456
app_request_duration_seconds_count{endpoint="/work"} 10.0

# Summary
app_process_time_seconds_sum 5.234
app_process_time_seconds_count 10.0
```

## Phần 4: Configure Prometheus Scraping

### Bước 1: Update Prometheus Config

Edit `/etc/prometheus/prometheus.yml`:

```bash
sudo vim /etc/prometheus/prometheus.yml
```

Add scrape config:

```yaml
scrape_configs:
  # ... existing configs ...

  - job_name: 'custom_app'
    static_configs:
      - targets: ['localhost:8080']
        labels:
          app: 'custom-metrics-lab'
          environment: 'lab'
```

### Bước 2: Reload Prometheus

```bash
# Method 1: Systemctl reload
sudo systemctl reload prometheus

# Method 2: API reload
curl -X POST http://localhost:9090/-/reload
```

### Bước 3: Verify Target

1. Truy cập Prometheus UI: `http://<server-ip>:9090`
2. Navigate to **Status** → **Targets**
3. Verify `custom_app` target is UP

## Phần 5: Query Custom Metrics

### Trong Prometheus UI

Try các queries sau:

#### Counter Queries

```promql
# Total requests
app_requests_total

# Requests per second (rate)
rate(app_requests_total[1m])

# Total requests by endpoint
sum by (endpoint) (app_requests_total)

# Error rate
rate(app_requests_total{status="500"}[1m])
```

#### Gauge Queries

```promql
# Current active users
app_active_users

# Current queue size
app_queue_size

# Average queue size over 5 minutes
avg_over_time(app_queue_size[5m])
```

#### Histogram Queries

```promql
# Request latency histogram
app_request_duration_seconds_bucket

# 95th percentile latency
histogram_quantile(0.95, rate(app_request_duration_seconds_bucket[5m]))

# 99th percentile latency
histogram_quantile(0.99, rate(app_request_duration_seconds_bucket[5m]))

# Average latency
rate(app_request_duration_seconds_sum[5m]) / rate(app_request_duration_seconds_count[5m])
```

#### Summary Queries

```promql
# Process time summary
app_process_time_seconds_sum

# Average process time
rate(app_process_time_seconds_sum[5m]) / rate(app_process_time_seconds_count[5m])
```

## Phần 6: Run as Systemd Service (Optional)

### Bước 1: Create Service File

```bash
sudo vim /etc/systemd/system/custom-metrics-app.service
```

Nội dung:

```ini
[Unit]
Description=Custom Metrics Application
After=network.target

[Service]
Type=simple
User=prometheus
WorkingDirectory=/home/<your-user>/prometheus-labs/custom-metrics
ExecStart=/usr/bin/python3 /home/<your-user>/prometheus-labs/custom-metrics/app.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Bước 2: Enable và Start Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable custom-metrics-app
sudo systemctl start custom-metrics-app
sudo systemctl status custom-metrics-app
```

## Troubleshooting

### Issue 1: Port Already in Use

**Triệu chứng**: `Address already in use`

**Giải pháp**:
```bash
# Find process using port 8080
sudo lsof -i :8080

# Kill process
sudo kill -9 <PID>

# Or change port in app.py
app.run(host='0.0.0.0', port=8081)
```

### Issue 2: Module Not Found

**Triệu chứng**: `ModuleNotFoundError: No module named 'prometheus_client'`

**Giải pháp**:
```bash
# Install for current user
pip3 install --user prometheus_client flask

# Or install globally
sudo pip3 install prometheus_client flask
```

### Issue 3: Target DOWN trong Prometheus

**Triệu chứng**: custom_app target shows DOWN

**Giải pháp**:
```bash
# Check app is running
curl http://localhost:8080/metrics

# Check Prometheus can reach app
# If running on different hosts, check firewall
sudo ufw allow 8080/tcp

# Check Prometheus config
sudo /usr/local/bin/promtool check config /etc/prometheus/prometheus.yml
```

### Issue 4: Metrics Not Updating

**Triệu chứng**: Metrics values không thay đổi

**Giải pháp**:
```bash
# Generate traffic
for i in {1..20}; do curl http://localhost:8080/work; sleep 1; done

# Check metrics directly
curl http://localhost:8080/metrics | grep app_requests_total

# Wait for scrape interval (15s default)
```

## Best Practices

### 1. Metric Naming

```python
# Good: Clear, descriptive names
http_requests_total
database_query_duration_seconds
cache_hit_ratio

# Bad: Vague or inconsistent
requests
db_time
hits
```

### 2. Label Usage

```python
# Good: Low cardinality labels
REQUEST_COUNT.labels(method='GET', endpoint='/api', status='200')

# Bad: High cardinality (avoid user IDs, timestamps)
REQUEST_COUNT.labels(user_id='12345', timestamp='2024-01-01')
```

### 3. Metric Types

- **Counter**: Use cho values that only increase (requests, errors, bytes sent)
- **Gauge**: Use cho values that can go up/down (temperature, memory usage, queue size)
- **Histogram**: Use cho distributions (latency, request size)
- **Summary**: Similar to histogram, use khi cần client-side quantiles

### 4. Bucket Selection (Histogram)

```python
# Good: Buckets cover expected range
buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]

# Bad: Too few or wrong range
buckets=[1, 10, 100]  # If latency is in milliseconds
```

## Advanced: Custom Collector

Tạo custom collector cho complex metrics:

```python
from prometheus_client.core import GaugeMetricFamily, REGISTRY

class CustomCollector:
    def collect(self):
        # Create metric family
        metric = GaugeMetricFamily(
            'custom_system_info',
            'Custom system information',
            labels=['hostname', 'version']
        )
        
        # Add samples
        metric.add_metric(['myhost', '1.0.0'], 1)
        
        yield metric

# Register collector
REGISTRY.register(CustomCollector())
```

## Tiếp Theo

Sau khi hoàn thành lab này, bạn đã có:
- ✅ Python app exposing custom metrics
- ✅ Implemented Counter, Gauge, Histogram, Summary
- ✅ Prometheus scraping custom app
- ✅ Có thể query custom metrics

Tiếp tục với: [Verification](./verification.md) để kiểm tra kết quả, sau đó chuyển sang [Lab 03 - PromQL Queries](../lab-03-promql-queries/README.md)
