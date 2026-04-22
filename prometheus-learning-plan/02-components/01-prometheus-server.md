# Prometheus Server

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Kiến Trúc](#kiến-trúc)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Tổng Quan

Prometheus Server là thành phần trung tâm của hệ thống Prometheus. Nó chịu trách nhiệm thu thập (scrape) metrics từ các targets, lưu trữ dữ liệu time-series, và cung cấp HTTP API để truy vấn dữ liệu.

Prometheus Server bao gồm 3 phần chính:
- **Retrieval**: Thu thập metrics từ các targets
- **TSDB (Time Series Database)**: Lưu trữ dữ liệu time-series
- **HTTP Server**: Cung cấp API và Web UI

## Kiến Trúc

```
┌─────────────────────────────────────────────────────────┐
│              Prometheus Server                          │
│                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────┐ │
│  │  Retrieval   │───▶│     TSDB     │◀───│   HTTP   │ │
│  │   (Scrape)   │    │   (Storage)  │    │  Server  │ │
│  └──────┬───────┘    └──────────────┘    └────┬─────┘ │
│         │                                      │       │
└─────────┼──────────────────────────────────────┼───────┘
          │                                      │
          ▼                                      ▼
    ┌──────────┐                          ┌──────────┐
    │ Targets  │                          │  Clients │
    │(Exporters)│                         │(PromQL)  │
    └──────────┘                          └──────────┘
```

### Các Thành Phần Chi Tiết

**1. Retrieval Component**
- Pull metrics từ các targets theo scrape_interval
- Hỗ trợ service discovery để tự động phát hiện targets
- Xử lý relabeling và filtering

**2. TSDB Component**
- Lưu trữ dữ liệu time-series hiệu quả
- Nén dữ liệu để tiết kiệm không gian
- Hỗ trợ retention policies

**3. HTTP Server Component**
- Cung cấp PromQL query API
- Web UI để visualize và query
- API endpoints cho alerting và federation

## Cách Hoạt Động

### Quy Trình Scraping

1. **Service Discovery**: Prometheus phát hiện targets cần scrape
2. **Target Relabeling**: Áp dụng relabel_configs để modify targets
3. **HTTP GET Request**: Gửi request đến `/metrics` endpoint
4. **Parse Metrics**: Parse response theo Prometheus exposition format
5. **Metric Relabeling**: Áp dụng metric_relabel_configs
6. **Store to TSDB**: Lưu metrics vào time-series database

### Data Model

Mỗi time-series được định danh bởi:
- **Metric name**: Tên của metric (ví dụ: `http_requests_total`)
- **Labels**: Các cặp key-value (ví dụ: `method="GET", status="200"`)

```
http_requests_total{method="GET", status="200"} 1234
│                   │                            │
│                   │                            └─ Value
│                   └─ Labels
└─ Metric name
```

### Storage Engine

Prometheus sử dụng local time-series database với các đặc điểm:
- **Blocks**: Dữ liệu được tổ chức thành blocks (mặc định 2 giờ)
- **Compression**: Nén dữ liệu hiệu quả (trung bình 1.3 bytes/sample)
- **WAL (Write-Ahead Log)**: Đảm bảo durability
- **Compaction**: Tự động merge và compact blocks cũ

```
data/
├── 01ABCDEFGHIJKLMNOP/  # Block 1
│   ├── chunks/
│   ├── index
│   └── meta.json
├── 01QRSTUVWXYZ123456/  # Block 2
│   ├── chunks/
│   ├── index
│   └── meta.json
└── wal/                 # Write-Ahead Log
    ├── 00000000
    └── 00000001
```

## Use Cases

### 1. Monitoring Infrastructure

Sử dụng Prometheus Server để monitor servers, containers, và network devices:

```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```

### 2. Application Monitoring

Monitor application metrics thông qua client libraries:

```yaml
scrape_configs:
  - job_name: 'app'
    static_configs:
      - targets: ['app1:8080', 'app2:8080']
```

### 3. Service Discovery

Tự động phát hiện targets trong dynamic environments:

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
```

### 4. Federation

Thu thập metrics từ các Prometheus servers khác:

```yaml
scrape_configs:
  - job_name: 'federate'
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job="prometheus"}'
    static_configs:
      - targets: ['prometheus1:9090', 'prometheus2:9090']
```

## Cấu Hình

### Cấu Hình Cơ Bản

File `prometheus.yml` cơ bản:

```yaml
# Global configuration
global:
  scrape_interval: 15s      # Scrape targets mỗi 15 giây
  evaluation_interval: 15s  # Evaluate rules mỗi 15 giây
  external_labels:
    cluster: 'production'
    region: 'us-east-1'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']

# Rule files
rule_files:
  - 'alerts/*.yml'
  - 'rules/*.yml'

# Scrape configurations
scrape_configs:
  # Scrape Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Scrape node exporters
  - job_name: 'node'
    static_configs:
      - targets:
          - 'node1:9100'
          - 'node2:9100'
          - 'node3:9100'
```

### Cấu Hình Storage

Cấu hình retention và storage:

```yaml
# Command-line flags
--storage.tsdb.path=/var/lib/prometheus/data
--storage.tsdb.retention.time=15d
--storage.tsdb.retention.size=50GB
--storage.tsdb.wal-compression
```

### Cấu Hình Relabeling

Sử dụng relabeling để modify targets và metrics:

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    
    # Target relabeling
    relabel_configs:
      # Chỉ scrape pods có annotation prometheus.io/scrape=true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      
      # Sử dụng custom port từ annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: $1:${1}
    
    # Metric relabeling
    metric_relabel_configs:
      # Drop metrics có tên bắt đầu bằng go_
      - source_labels: [__name__]
        regex: 'go_.*'
        action: drop
```

### Cấu Hình Remote Storage

Cấu hình remote write để gửi metrics đến remote storage:

```yaml
remote_write:
  - url: "https://remote-storage.example.com/api/v1/write"
    basic_auth:
      username: prometheus
      password: secret
    queue_config:
      capacity: 10000
      max_shards: 5
      max_samples_per_send: 1000

remote_read:
  - url: "https://remote-storage.example.com/api/v1/read"
    basic_auth:
      username: prometheus
      password: secret
```

## Best Practices

### 1. Scrape Interval

- **Default**: 15s là giá trị hợp lý cho hầu hết use cases
- **High-frequency**: Sử dụng 5s-10s cho metrics cần độ chính xác cao
- **Low-frequency**: Sử dụng 30s-60s cho metrics ít thay đổi

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'critical-app'
    scrape_interval: 5s  # Override cho job cụ thể
    static_configs:
      - targets: ['app:8080']
```

### 2. Resource Planning

**CPU**: 
- ~1 core cho mỗi 1 triệu samples/second
- Tăng khi sử dụng complex queries

**Memory**:
- ~1-2 bytes per sample trong memory
- ~1.3 bytes per sample on disk (sau compression)
- Ước tính: `memory = samples * retention_time * bytes_per_sample`

**Disk**:
- Ước tính: `disk = samples_per_second * retention_seconds * 1.3 bytes`
- Ví dụ: 100k samples/s, 15 days retention = ~170GB

### 3. High Availability

Chạy multiple Prometheus servers với cùng configuration:

```
┌─────────────┐     ┌─────────────┐
│ Prometheus  │     │ Prometheus  │
│  Server 1   │     │  Server 2   │
└──────┬──────┘     └──────┬──────┘
       │                   │
       └───────┬───────────┘
               ▼
         ┌──────────┐
         │ Targets  │
         └──────────┘
```

Cả hai servers scrape cùng targets độc lập. Alertmanager sẽ deduplicate alerts.

### 4. Security

**Authentication**:
```yaml
# Sử dụng basic auth cho scrape targets
scrape_configs:
  - job_name: 'secure-app'
    basic_auth:
      username: prometheus
      password: secret
    static_configs:
      - targets: ['app:8080']
```

**TLS**:
```yaml
scrape_configs:
  - job_name: 'secure-app'
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/ca.crt
      cert_file: /etc/prometheus/client.crt
      key_file: /etc/prometheus/client.key
    static_configs:
      - targets: ['app:8443']
```

### 5. Monitoring Prometheus

Monitor Prometheus itself:

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

Key metrics để monitor:
- `prometheus_tsdb_head_samples`: Số samples trong memory
- `prometheus_tsdb_head_series`: Số time-series active
- `prometheus_target_scrapes_exceeded_sample_limit_total`: Targets vượt sample limit
- `prometheus_rule_evaluation_failures_total`: Rule evaluation failures

### 6. Cardinality Management

Tránh high cardinality labels:

**❌ Bad**:
```
http_requests_total{user_id="12345"}  # user_id có thể có hàng triệu values
```

**✅ Good**:
```
http_requests_total{endpoint="/api/users"}  # endpoint có số lượng values hữu hạn
```

### 7. Backup và Recovery

**Backup**:
```bash
# Snapshot API
curl -XPOST http://localhost:9090/api/v1/admin/tsdb/snapshot

# Backup snapshot directory
tar -czf prometheus-backup.tar.gz /var/lib/prometheus/data/snapshots/
```

**Recovery**:
```bash
# Extract backup
tar -xzf prometheus-backup.tar.gz -C /var/lib/prometheus/data/

# Restart Prometheus
systemctl restart prometheus
```

## Tài Liệu Liên Quan

- [Exporters](./02-exporters.md) - Các exporters để thu thập metrics
- [Service Discovery](./05-service-discovery.md) - Cấu hình service discovery
- [Storage and Retention](./06-storage-retention.md) - Chi tiết về storage
- [Prometheus Architecture](../01-fundamentals/04-prometheus-architecture.md) - Kiến trúc tổng thể

## Tài Liệu Tham Khảo

- [Prometheus Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Prometheus Storage](https://prometheus.io/docs/prometheus/latest/storage/)
- [Prometheus API](https://prometheus.io/docs/prometheus/latest/querying/api/)
