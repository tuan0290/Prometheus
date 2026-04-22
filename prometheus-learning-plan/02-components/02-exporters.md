# Exporters

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Kiến Trúc](#kiến-trúc)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Tổng Quan

Exporters là các chương trình chuyển đổi metrics từ các hệ thống không hỗ trợ Prometheus format sang Prometheus metrics format. Chúng expose metrics qua HTTP endpoint `/metrics` để Prometheus có thể scrape.

Có 3 loại exporters chính:
- **Official Exporters**: Do Prometheus team maintain (node_exporter, blackbox_exporter)
- **Third-party Exporters**: Do community maintain (mysql_exporter, postgres_exporter)
- **Custom Exporters**: Tự viết cho use cases đặc biệt

## Kiến Trúc

```
┌─────────────────────────────────────────────────────────┐
│                  Prometheus Server                      │
│                                                         │
│                    HTTP GET /metrics                    │
└────────────┬────────────┬────────────┬─────────────────┘
             │            │            │
             ▼            ▼            ▼
      ┌───────────┐ ┌───────────┐ ┌───────────┐
      │   Node    │ │ Blackbox  │ │  Custom   │
      │ Exporter  │ │ Exporter  │ │ Exporter  │
      └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
            │             │             │
            ▼             ▼             ▼
      ┌─────────┐   ┌─────────┐   ┌─────────┐
      │  Host   │   │ Network │   │  App    │
      │ Metrics │   │ Probes  │   │ Metrics │
      └─────────┘   └─────────┘   └─────────┘
```

### Exporter Pattern

```
┌──────────────────────────────────────┐
│           Exporter Process           │
│                                      │
│  ┌────────────┐    ┌──────────────┐ │
│  │  Collector │───▶│   Registry   │ │
│  │  (Gather)  │    │   (Store)    │ │
│  └──────┬─────┘    └──────┬───────┘ │
│         │                 │         │
│         ▼                 ▼         │
│  ┌────────────┐    ┌──────────────┐ │
│  │   Source   │    │ HTTP Handler │ │
│  │  (System)  │    │  /metrics    │ │
│  └────────────┘    └──────────────┘ │
└──────────────────────────────────────┘
```

## Cách Hoạt Động

### Quy Trình Exporting

1. **Collect**: Exporter thu thập metrics từ source system
2. **Transform**: Chuyển đổi sang Prometheus format
3. **Expose**: Expose metrics qua HTTP endpoint
4. **Scrape**: Prometheus scrape metrics từ endpoint

### Prometheus Exposition Format

Format chuẩn của metrics:

```
# HELP metric_name Description of the metric
# TYPE metric_name metric_type
metric_name{label1="value1",label2="value2"} metric_value timestamp

# Example
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1234 1609459200000
http_requests_total{method="POST",status="201"} 567 1609459200000
```

## Use Cases

### 1. Node Exporter - Hardware và OS Metrics

**Mục đích**: Thu thập metrics từ Linux/Unix systems

**Metrics chính**:
- CPU usage
- Memory usage
- Disk I/O
- Network statistics
- Filesystem usage

**Cài đặt**:
```bash
# Download
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xvfz node_exporter-*.tar.gz
cd node_exporter-*

# Run
./node_exporter
```

**Metrics example**:
```
node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67
node_memory_MemAvailable_bytes 8589934592
node_disk_io_time_seconds_total{device="sda"} 123.45
node_network_receive_bytes_total{device="eth0"} 1234567890
```

### 2. Blackbox Exporter - Network Probing

**Mục đích**: Probe endpoints qua HTTP, HTTPS, DNS, TCP, ICMP

**Use cases**:
- HTTP/HTTPS endpoint monitoring
- SSL certificate expiry
- DNS resolution time
- TCP port availability
- ICMP ping checks

**Probe types**:
- `http`: HTTP/HTTPS probes
- `tcp`: TCP connection probes
- `dns`: DNS lookup probes
- `icmp`: ICMP ping probes

**Metrics example**:
```
probe_success{instance="https://example.com"} 1
probe_duration_seconds{instance="https://example.com"} 0.123
probe_http_status_code{instance="https://example.com"} 200
probe_ssl_earliest_cert_expiry{instance="https://example.com"} 1735689600
```

### 3. MySQL Exporter - Database Metrics

**Mục đích**: Thu thập metrics từ MySQL/MariaDB

**Metrics chính**:
- Query performance
- Connection pool
- Replication status
- InnoDB metrics

**Cài đặt**:
```bash
# Download
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz
tar xvfz mysqld_exporter-*.tar.gz
cd mysqld_exporter-*

# Create MySQL user
mysql -u root -p << EOF
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'password';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
FLUSH PRIVILEGES;
EOF

# Run
export DATA_SOURCE_NAME='exporter:password@(localhost:3306)/'
./mysqld_exporter
```

### 4. Custom Exporters - Application-Specific Metrics

**Mục đích**: Export metrics từ applications hoặc services đặc biệt

**Ví dụ**: Export metrics từ log files

```python
from prometheus_client import start_http_server, Counter, Gauge
import time
import re

# Define metrics
error_count = Counter('log_errors_total', 'Total number of errors in logs')
warning_count = Counter('log_warnings_total', 'Total number of warnings in logs')
active_users = Gauge('active_users', 'Number of active users')

def parse_log_file(filename):
    """Parse log file and update metrics"""
    with open(filename, 'r') as f:
        for line in f:
            if 'ERROR' in line:
                error_count.inc()
            elif 'WARNING' in line:
                warning_count.inc()
            
            # Extract active users from log
            match = re.search(r'active_users=(\d+)', line)
            if match:
                active_users.set(int(match.group(1)))

if __name__ == '__main__':
    # Start HTTP server on port 8000
    start_http_server(8000)
    
    # Parse logs every 10 seconds
    while True:
        parse_log_file('/var/log/app.log')
        time.sleep(10)
```

## Cấu Hình

### Cấu Hình Node Exporter

**Prometheus configuration**:
```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets:
          - 'server1:9100'
          - 'server2:9100'
          - 'server3:9100'
        labels:
          env: 'production'
          region: 'us-east-1'
```

**Enable specific collectors**:
```bash
# Chỉ enable một số collectors
./node_exporter \
  --collector.disable-defaults \
  --collector.cpu \
  --collector.meminfo \
  --collector.diskstats \
  --collector.filesystem
```

### Cấu Hình Blackbox Exporter

**Blackbox exporter config** (`blackbox.yml`):
```yaml
modules:
  # HTTP probe
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: [200]
      method: GET
      preferred_ip_protocol: "ip4"
  
  # HTTPS probe with SSL verification
  https_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: [200]
      method: GET
      tls_config:
        insecure_skip_verify: false
  
  # TCP probe
  tcp_connect:
    prober: tcp
    timeout: 5s
  
  # ICMP probe
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: "ip4"
  
  # DNS probe
  dns_example:
    prober: dns
    timeout: 5s
    dns:
      query_name: "example.com"
      query_type: "A"
```

**Prometheus configuration**:
```yaml
scrape_configs:
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
          - https://api.example.com
          - https://www.example.com
    relabel_configs:
      # Pass target as parameter
      - source_labels: [__address__]
        target_label: __param_target
      # Set blackbox exporter address
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  - job_name: 'blackbox-tcp'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets:
          - database:3306
          - redis:6379
          - rabbitmq:5672
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

### Cấu Hình MySQL Exporter

**Prometheus configuration**:
```yaml
scrape_configs:
  - job_name: 'mysql'
    static_configs:
      - targets:
          - 'mysql-exporter:9104'
        labels:
          database: 'production'
          instance: 'mysql-master'
```

**MySQL exporter flags**:
```bash
./mysqld_exporter \
  --config.my-cnf=/etc/mysql/exporter.cnf \
  --collect.info_schema.tables \
  --collect.info_schema.innodb_metrics \
  --collect.perf_schema.tableiowaits \
  --collect.perf_schema.indexiowaits
```

### Cấu Hình Custom Exporter

**Prometheus configuration**:
```yaml
scrape_configs:
  - job_name: 'custom-app'
    static_configs:
      - targets:
          - 'app-exporter:8000'
        labels:
          app: 'my-application'
          env: 'production'
```

## Best Practices

### 1. Exporter Deployment

**Sidecar Pattern** (cho containers):
```yaml
# Kubernetes example
apiVersion: v1
kind: Pod
metadata:
  name: app-with-exporter
spec:
  containers:
    # Main application
    - name: app
      image: myapp:latest
      ports:
        - containerPort: 8080
    
    # Exporter sidecar
    - name: exporter
      image: custom-exporter:latest
      ports:
        - containerPort: 9090
```

**Separate Service** (cho infrastructure):
```
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Server1 │     │ Server2 │     │ Server3 │
│         │     │         │     │         │
│ Node    │     │ Node    │     │ Node    │
│Exporter │     │Exporter │     │Exporter │
│  :9100  │     │  :9100  │     │  :9100  │
└─────────┘     └─────────┘     └─────────┘
```

### 2. Metric Naming

Tuân theo naming conventions:

```
# Format: <namespace>_<subsystem>_<name>_<unit>

# ✅ Good
http_requests_total
http_request_duration_seconds
database_queries_total
cache_hits_total

# ❌ Bad
httpRequests
request_time
db_query_count
cacheHit
```

### 3. Label Usage

**✅ Good labels** (low cardinality):
```
http_requests_total{method="GET", status="200", endpoint="/api/users"}
```

**❌ Bad labels** (high cardinality):
```
http_requests_total{user_id="12345", request_id="abc-def-ghi"}
```

### 4. Exporter Performance

**Caching**: Cache expensive operations

```python
from prometheus_client import Gauge
import time

last_update = 0
cached_value = 0

def expensive_operation():
    global last_update, cached_value
    
    # Cache for 60 seconds
    if time.time() - last_update > 60:
        cached_value = perform_expensive_calculation()
        last_update = time.time()
    
    return cached_value

metric = Gauge('expensive_metric', 'Expensive metric with caching')
metric.set_function(expensive_operation)
```

**Timeout**: Set reasonable timeouts

```python
import requests
from prometheus_client import Counter

timeout_counter = Counter('scrape_timeouts_total', 'Total scrape timeouts')

def collect_metrics():
    try:
        response = requests.get('http://api/metrics', timeout=5)
        return response.json()
    except requests.Timeout:
        timeout_counter.inc()
        return {}
```

### 5. Error Handling

Expose error metrics:

```python
from prometheus_client import Counter, Gauge

scrape_errors = Counter('exporter_scrape_errors_total', 'Total scrape errors')
last_scrape_success = Gauge('exporter_last_scrape_success', 'Last scrape success (1=success, 0=failure)')

def collect():
    try:
        # Collect metrics
        data = fetch_data()
        last_scrape_success.set(1)
        return data
    except Exception as e:
        scrape_errors.inc()
        last_scrape_success.set(0)
        raise
```

### 6. Security

**Authentication**:
```yaml
scrape_configs:
  - job_name: 'secure-exporter'
    basic_auth:
      username: prometheus
      password: secret
    static_configs:
      - targets: ['exporter:9090']
```

**TLS**:
```yaml
scrape_configs:
  - job_name: 'secure-exporter'
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/ca.crt
      cert_file: /etc/prometheus/client.crt
      key_file: /etc/prometheus/client.key
    static_configs:
      - targets: ['exporter:9090']
```

### 7. Testing Exporters

Test exporter output:

```bash
# Check if exporter is running
curl http://localhost:9100/metrics

# Validate Prometheus format
promtool check metrics < metrics.txt

# Test specific metrics
curl -s http://localhost:9100/metrics | grep node_cpu_seconds_total
```

### 8. Monitoring Exporters

Monitor exporter health:

```yaml
# Alert on exporter down
groups:
  - name: exporter_alerts
    rules:
      - alert: ExporterDown
        expr: up{job="node"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Exporter {{ $labels.instance }} is down"
```

## Tài Liệu Liên Quan

- [Prometheus Server](./01-prometheus-server.md) - Cấu hình scraping
- [Instrumentation](../04-instrumentation/README.md) - Tạo custom metrics
- [Service Discovery](./05-service-discovery.md) - Tự động discover exporters
- [Lab 05 - Exporters](../07-labs/lab-05-exporters/README.md) - Bài tập thực hành

## Tài Liệu Tham Khảo

- [Prometheus Exporters](https://prometheus.io/docs/instrumenting/exporters/)
- [Node Exporter](https://github.com/prometheus/node_exporter)
- [Blackbox Exporter](https://github.com/prometheus/blackbox_exporter)
- [Writing Exporters](https://prometheus.io/docs/instrumenting/writing_exporters/)
