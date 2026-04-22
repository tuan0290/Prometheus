# Domain 4: Instrumentation and Exporters (20%)

## Tổng Quan

Domain này chiếm 20% kỳ thi PCA. Tập trung vào client libraries, metric naming, exporters, và Pushgateway.

## 1. Client Libraries

### Supported Languages

| Language | Library | Import |
|----------|---------|--------|
| Go | `github.com/prometheus/client_golang` | Official |
| Python | `prometheus_client` | Official |
| Java | `io.prometheus:simpleclient` | Official |
| Ruby | `prometheus-client` | Official |
| Node.js | `prom-client` | Community |

### Python Example

```python
from prometheus_client import Counter, Gauge, Histogram, Summary
from prometheus_client import start_http_server, generate_latest

# Counter
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

# Gauge
ACTIVE_CONNECTIONS = Gauge(
    'active_connections',
    'Number of active connections'
)

# Histogram
REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
)

# Summary
PROCESS_TIME = Summary(
    'process_time_seconds',
    'Time to process request'
)

# Usage
REQUEST_COUNT.labels(method='GET', endpoint='/api', status='200').inc()
ACTIVE_CONNECTIONS.set(42)
REQUEST_LATENCY.observe(0.123)

with PROCESS_TIME.time():
    do_work()
```

### Go Example

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    requestCount = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "status"},
    )

    requestLatency = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request latency",
            Buckets: prometheus.DefBuckets,
        },
        []string{"endpoint"},
    )
)

// Usage
requestCount.WithLabelValues("GET", "200").Inc()
requestLatency.WithLabelValues("/api").Observe(0.123)
```

## 2. Metric Naming Conventions

### Format

```
<namespace>_<subsystem>_<name>_<unit>

Ví dụ:
http_requests_total          # namespace=http, name=requests, unit=total
node_memory_MemTotal_bytes   # namespace=node, subsystem=memory, name=MemTotal, unit=bytes
process_cpu_seconds_total    # namespace=process, name=cpu, unit=seconds_total
```

### Rules

1. **Lowercase**: Tất cả lowercase
2. **Underscores**: Dùng `_` thay vì `-` hoặc `.`
3. **Units**: Luôn include units (seconds, bytes, total)
4. **Base units**: Dùng base units (seconds không milliseconds, bytes không kilobytes)
5. **Suffix**: Counters kết thúc bằng `_total`

### Good vs Bad Names

```
# Good
http_requests_total
http_request_duration_seconds
node_memory_MemAvailable_bytes
process_open_fds

# Bad
httpRequestsTotal    # camelCase
http-requests        # hyphens
http_requests_ms     # non-base unit
requests             # too vague
```

## 3. Label Best Practices

### Good Labels

```python
# Low cardinality, meaningful
REQUEST_COUNT.labels(
    method='GET',        # ~5 values
    endpoint='/api/v1',  # ~20 values
    status='200'         # ~10 values
)
```

### Bad Labels (High Cardinality)

```python
# AVOID: High cardinality
REQUEST_COUNT.labels(
    user_id='12345',          # Millions of values
    session_id='abc-xyz-123', # Unique per session
    timestamp='2024-01-01',   # Unique per request
    request_id='req-001'      # Unique per request
)
```

### Label Cardinality Impact

```
Low cardinality (< 100 values):  OK
Medium cardinality (100-1000):   Caution
High cardinality (> 10000):      Avoid
```

## 4. Common Exporters

### node_exporter

Collect system metrics từ Linux/Unix:

```bash
# Install
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz

# Run
./node_exporter --web.listen-address=":9100"
```

**Key metrics**:
```promql
node_cpu_seconds_total          # CPU time by mode
node_memory_MemAvailable_bytes  # Available memory
node_filesystem_avail_bytes     # Available disk space
node_network_receive_bytes_total # Network traffic
node_load1                      # 1-minute load average
```

### blackbox_exporter

Probe endpoints (HTTP, TCP, ICMP, DNS):

```yaml
# blackbox.yml
modules:
  http_2xx:
    prober: http
    http:
      valid_status_codes: []  # 2xx
  tcp_connect:
    prober: tcp
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: localhost:9115
```

**Key metrics**:
```promql
probe_success                    # 1 if probe succeeded
probe_http_status_code           # HTTP status code
probe_http_duration_seconds      # Response time
probe_ssl_earliest_cert_expiry   # SSL cert expiry
```

### postgres_exporter

Monitor PostgreSQL:

```bash
# Environment variable
export DATA_SOURCE_NAME="postgresql://user:pass@localhost:5432/db?sslmode=disable"
./postgres_exporter
```

**Key metrics**:
```promql
pg_up                           # PostgreSQL up
pg_stat_activity_count          # Active connections
pg_database_size_bytes          # Database size
pg_stat_database_xact_commit    # Committed transactions
```

### mysql_exporter

Monitor MySQL:

```bash
# .my.cnf
[client]
user=prometheus
password=secret

./mysqld_exporter --config.my-cnf=.my.cnf
```

## 5. Pushgateway

### Use Cases

Pushgateway phù hợp cho:
- Batch jobs (cron jobs, ETL pipelines)
- Short-lived processes
- Jobs behind firewall

### Không Nên Dùng Pushgateway Cho

- Long-running services (dùng pull model)
- Replacement cho service discovery
- Aggregation layer

### Usage

```bash
# Push metrics
echo "batch_job_duration_seconds 42.5" | curl --data-binary @- http://localhost:9091/metrics/job/batch_job

# Push với labels
cat <<EOF | curl --data-binary @- http://localhost:9091/metrics/job/batch_job/instance/server1
# HELP batch_job_duration_seconds Duration of batch job
# TYPE batch_job_duration_seconds gauge
batch_job_duration_seconds 42.5
EOF

# Delete metrics
curl -X DELETE http://localhost:9091/metrics/job/batch_job/instance/server1
```

### Prometheus Config

```yaml
scrape_configs:
  - job_name: 'pushgateway'
    honor_labels: true  # Important!
    static_configs:
      - targets: ['localhost:9091']
```

**Lưu ý**: `honor_labels: true` để giữ labels từ pushed metrics.

## 6. Custom Exporters

### Khi Nào Tạo Custom Exporter

- Third-party system không có exporter
- Internal metrics không expose qua standard format
- Custom business metrics

### Exporter Pattern

```python
from prometheus_client import Gauge, CollectorRegistry, generate_latest
from flask import Flask, Response

app = Flask(__name__)
registry = CollectorRegistry()

# Define metrics
db_connections = Gauge(
    'myapp_db_connections',
    'Database connections',
    registry=registry
)

@app.route('/metrics')
def metrics():
    # Update metrics
    db_connections.set(get_db_connection_count())
    return Response(generate_latest(registry), mimetype='text/plain')
```

## 7. Exposition Format

Prometheus text format:

```
# HELP metric_name Description of the metric
# TYPE metric_name counter|gauge|histogram|summary|untyped
metric_name{label1="value1",label2="value2"} value [timestamp]

# Example
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1234 1609459200000
http_requests_total{method="POST",status="201"} 56
```

## 8. Key Concepts Summary

| Concept | Key Points |
|---------|-----------|
| Client Libraries | Go, Python, Java, Ruby (official) |
| Naming | lowercase, underscores, base units, _total suffix |
| Labels | Low cardinality, meaningful, avoid user IDs |
| node_exporter | System metrics (CPU, memory, disk, network) |
| blackbox_exporter | Probe endpoints (HTTP, TCP, ICMP) |
| Pushgateway | Batch jobs, short-lived processes only |
| Custom Exporters | For systems without existing exporters |

## 9. Practice Questions

**Q1**: Tại sao counters nên kết thúc bằng `_total`?

<details>
<summary>Đáp án</summary>

Theo Prometheus naming conventions, counters nên kết thúc bằng `_total` để rõ ràng đây là cumulative counter. Điều này giúp người dùng biết cần dùng `rate()` hoặc `increase()` khi query, và phân biệt với gauges. Ví dụ: `http_requests_total` rõ ràng hơn `http_requests`.

</details>

**Q2**: Sự khác biệt giữa `honor_labels: true` và `false` trong Pushgateway config?

<details>
<summary>Đáp án</summary>

`honor_labels: true`: Giữ labels từ pushed metrics, không override bằng labels từ Prometheus config. Cần thiết cho Pushgateway để preserve job/instance labels từ batch jobs. `honor_labels: false` (default): Prometheus labels override pushed labels nếu conflict. Dùng `honor_labels: true` cho Pushgateway để tránh mất label information.

</details>

**Q3**: Khi nào nên tạo custom exporter thay vì dùng existing exporter?

<details>
<summary>Đáp án</summary>

Tạo custom exporter khi: (1) Không có existing exporter cho system cần monitor, (2) Cần expose internal business metrics không có trong standard exporters, (3) Existing exporter không đủ metrics hoặc không phù hợp với use case. Luôn check https://prometheus.io/docs/instrumenting/exporters/ trước khi tạo custom exporter.

</details>

## Tiếp Theo

- [Domain 5: Alerting](./07-domain-alerting.md)
- [Tài liệu tham khảo: Instrumentation](../04-instrumentation/)
