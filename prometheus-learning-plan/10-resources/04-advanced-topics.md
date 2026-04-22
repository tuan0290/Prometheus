# Advanced Topics

## Federation

Prometheus federation cho phép một Prometheus scrape metrics từ Prometheus khác, phù hợp cho large-scale deployments.

### Hierarchical Federation

```
Global Prometheus
├── DC1 Prometheus (scrapes local targets)
├── DC2 Prometheus (scrapes local targets)
└── DC3 Prometheus (scrapes local targets)
```

### Configuration

```yaml
# Global Prometheus scrapes from DC Prometheus instances
scrape_configs:
  - job_name: 'federate'
    honor_labels: true
    metrics_path: '/federate'
    params:
      match[]:
        - '{job="prometheus"}'
        - '{__name__=~"job:.*"}'  # Recording rules only
    static_configs:
      - targets:
          - 'dc1-prometheus:9090'
          - 'dc2-prometheus:9090'
```

**Tài liệu**: https://prometheus.io/docs/prometheus/latest/federation/

---

## Remote Storage

Cho long-term storage và horizontal scaling.

### Remote Write

```yaml
remote_write:
  - url: "http://thanos-receive:19291/api/v1/receive"
    queue_config:
      max_samples_per_send: 10000
      max_shards: 30
      capacity: 100000
```

### Remote Read

```yaml
remote_read:
  - url: "http://thanos-query:10902/api/v1/read"
    read_recent: false  # Only read historical data
```

### Popular Remote Storage Solutions

| Solution | URL | Mô Tả |
|----------|-----|-------|
| Thanos | https://thanos.io/ | HA + long-term storage |
| Cortex | https://cortexmetrics.io/ | Horizontally scalable |
| VictoriaMetrics | https://victoriametrics.com/ | High performance |
| Grafana Mimir | https://grafana.com/oss/mimir/ | Scalable Prometheus |
| InfluxDB | https://www.influxdata.com/ | Time series DB |

---

## High Availability (HA)

### Prometheus HA Setup

Chạy 2 Prometheus instances với cùng config:

```
Load Balancer
├── Prometheus 1 (scrapes all targets)
└── Prometheus 2 (scrapes all targets)
```

**Lưu ý**: Cả 2 instances scrape độc lập, data không sync. Dùng Thanos hoặc Cortex để deduplicate.

### Thanos HA

```
Prometheus 1 → Thanos Sidecar → Object Storage (S3/GCS)
Prometheus 2 → Thanos Sidecar ↗

Thanos Query → Thanos Store → Object Storage
            → Thanos Sidecar 1
            → Thanos Sidecar 2
```

**Tài liệu**: https://thanos.io/tip/thanos/quick-tutorial.md/

---

## Kubernetes Monitoring

### kube-state-metrics

```bash
# Install
kubectl apply -f https://github.com/kubernetes/kube-state-metrics/releases/latest/download/kube-state-metrics.yaml
```

### Prometheus Operator

```bash
# Install via Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack
```

### Key Kubernetes Metrics

```promql
# Pod restart count
kube_pod_container_status_restarts_total

# Deployment availability
kube_deployment_status_replicas_available / kube_deployment_spec_replicas

# Node CPU
sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (node)

# Container memory
container_memory_usage_bytes{container!=""}
```

---

## Cardinality Management

High cardinality là một trong những vấn đề phổ biến nhất với Prometheus.

### Detect High Cardinality

```promql
# Top metrics by series count
topk(10, count by (__name__) ({__name__=~".+"}))

# Total series count
prometheus_tsdb_head_series

# Series per job
count by (job) (up)
```

### Solutions

1. **Drop high-cardinality labels**:
```yaml
metric_relabel_configs:
  - regex: 'user_id|session_id|request_id'
    action: labeldrop
```

2. **Drop unnecessary metrics**:
```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'go_.*'
    action: drop
```

3. **Use recording rules** to pre-aggregate

---

## Security

### TLS Configuration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'secure_target'
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/ca.pem
      cert_file: /etc/prometheus/cert.pem
      key_file: /etc/prometheus/key.pem
```

### Basic Auth

```yaml
scrape_configs:
  - job_name: 'auth_target'
    basic_auth:
      username: 'prometheus'
      password_file: /etc/prometheus/password
```

### Reverse Proxy with nginx

```nginx
server {
    listen 443 ssl;
    server_name prometheus.example.com;

    ssl_certificate /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;

    auth_basic "Prometheus";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://localhost:9090;
    }
}
```

---

## Performance Tuning

### Scrape Optimization

```yaml
global:
  scrape_interval: 30s      # Increase if not needed every 15s
  scrape_timeout: 10s

scrape_configs:
  - job_name: 'high_frequency'
    scrape_interval: 5s     # Only for critical metrics
  
  - job_name: 'low_frequency'
    scrape_interval: 60s    # For slow-changing metrics
```

### Query Optimization

```yaml
# Use recording rules for expensive queries
groups:
  - name: performance
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
```

### Storage Optimization

```bash
# Prometheus 3.x: dùng config file thay vì CLI flags (deprecated)
# Trong prometheus.yml:
# storage:
#   tsdb:
#     retention:
#       time: 15d
#       size: 50GB

# CLI (deprecated nhưng vẫn hoạt động)
prometheus \
  --storage.tsdb.retention.time=15d \
  --storage.tsdb.retention.size=50GB \
  --storage.tsdb.wal-compression
```

---

## Useful Links

| Topic | URL |
|-------|-----|
| Thanos | https://thanos.io/ |
| Cortex | https://cortexmetrics.io/ |
| Grafana Mimir | https://grafana.com/oss/mimir/ |
| Prometheus Operator | https://github.com/prometheus-operator/prometheus-operator |
| kube-prometheus | https://github.com/prometheus-operator/kube-prometheus |
| Awesome Prometheus | https://github.com/roaldnefs/awesome-prometheus |
