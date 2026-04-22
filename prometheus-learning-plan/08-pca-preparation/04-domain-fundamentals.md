# Domain 2: Prometheus Fundamentals (20%)

## Tổng Quan

Domain này chiếm 20% kỳ thi PCA. Tập trung vào kiến trúc Prometheus, data model, metric types, scrape configuration, service discovery, và storage.

## 1. Prometheus Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Prometheus Server                         │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │   Retrieval  │  │    TSDB      │  │   HTTP Server    │  │
│  │  (Scraping)  │  │  (Storage)   │  │   (Query API)    │  │
│  └──────┬───────┘  └──────────────┘  └──────────────────┘  │
│         │                                                    │
└─────────┼────────────────────────────────────────────────────┘
          │ scrape
          ▼
┌─────────────────────────────────────────────────────────────┐
│  Targets: /metrics endpoints                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ node_exporter│  │  Custom App  │  │  Other Exporters │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Components

| Component | Vai Trò |
|-----------|---------|
| Prometheus Server | Core: scraping, storage, querying |
| Exporters | Expose metrics từ third-party systems |
| Pushgateway | Nhận metrics từ short-lived jobs |
| Alertmanager | Xử lý và route alerts |
| Client Libraries | Instrument application code |

## 2. Data Model

### Time Series

Mỗi time series được xác định bởi metric name và labels:

```
<metric_name>{<label_name>=<label_value>, ...}

Ví dụ:
http_requests_total{method="GET", status="200", handler="/api"}
node_cpu_seconds_total{cpu="0", mode="idle"}
```

### Metric Name

- Phải match regex: `[a-zA-Z_:][a-zA-Z0-9_:]*`
- Convention: `<namespace>_<subsystem>_<name>_<unit>`
- Ví dụ: `http_requests_total`, `node_memory_MemTotal_bytes`

### Labels

- Key-value pairs để differentiate time series
- Label name: `[a-zA-Z_][a-zA-Z0-9_]*`
- Labels bắt đầu bằng `__` là internal (reserved)
- High cardinality labels → performance issues

### Samples

Mỗi sample gồm:
- Timestamp (milliseconds)
- Float64 value

```
{metric_name + labels} → [(timestamp, value), ...]
```

## 3. Metric Types

### Counter

- Chỉ tăng (hoặc reset về 0 khi restart)
- Dùng cho: requests, errors, bytes sent
- Luôn dùng `rate()` hoặc `increase()` khi query

```promql
# Requests per second
rate(http_requests_total[5m])

# Total requests in last hour
increase(http_requests_total[1h])
```

### Gauge

- Có thể tăng hoặc giảm
- Dùng cho: temperature, memory usage, queue size
- Query trực tiếp

```promql
# Current memory usage
node_memory_MemAvailable_bytes

# Memory usage percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

### Histogram

- Đo distribution của observations
- Tạo ra: `_bucket`, `_sum`, `_count`
- Dùng cho: latency, request size

```promql
# 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Average latency
rate(http_request_duration_seconds_sum[5m]) /
rate(http_request_duration_seconds_count[5m])
```

### Summary

- Tương tự histogram nhưng tính quantiles client-side
- Tạo ra: `_sum`, `_count`, và quantile time series
- Không thể aggregate across instances

```promql
# Pre-calculated quantile
http_request_duration_seconds{quantile="0.95"}
```

### So Sánh Histogram vs Summary

| | Histogram | Summary |
|--|-----------|---------|
| Quantile calculation | Server-side | Client-side |
| Aggregatable | ✅ Yes | ❌ No |
| Configurable buckets | ✅ Yes | ❌ No |
| Accurate quantiles | Approximate | Accurate |
| Use case | When aggregation needed | When accuracy critical |

## 4. Scrape Configuration

### Basic Config

```yaml
global:
  scrape_interval: 15s      # Default scrape interval
  evaluation_interval: 15s  # Rule evaluation interval
  scrape_timeout: 10s       # Timeout per scrape

scrape_configs:
  - job_name: 'my_app'
    scrape_interval: 30s    # Override global
    static_configs:
      - targets: ['localhost:8080']
        labels:
          env: production
```

### Scrape Process

1. Prometheus sends HTTP GET to `<scheme>://<target><metrics_path>`
2. Target responds với metrics in exposition format
3. Prometheus parses và stores samples
4. Adds `job` và `instance` labels automatically

### Exposition Format

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1234
http_requests_total{method="POST",status="201"} 56
```

## 5. Service Discovery

### Static Configuration

```yaml
scrape_configs:
  - job_name: 'static'
    static_configs:
      - targets: ['host1:9090', 'host2:9090']
```

### File-Based Service Discovery

```yaml
scrape_configs:
  - job_name: 'file_sd'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/*.json'
        refresh_interval: 30s
```

Target file format:
```json
[
  {
    "targets": ["host1:9090"],
    "labels": {"env": "prod", "team": "infra"}
  }
]
```

### Kubernetes Service Discovery

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### Consul Service Discovery

```yaml
scrape_configs:
  - job_name: 'consul'
    consul_sd_configs:
      - server: 'localhost:8500'
        services: ['web', 'api']
```

## 6. Relabeling

Relabeling cho phép modify labels trước khi scraping:

```yaml
relabel_configs:
  # Keep only production targets
  - source_labels: [env]
    regex: production
    action: keep

  # Drop dev targets
  - source_labels: [env]
    regex: dev
    action: drop

  # Rename label
  - source_labels: [__meta_kubernetes_pod_name]
    target_label: pod

  # Extract from address
  - source_labels: [__address__]
    regex: '([^:]+):.*'
    target_label: hostname
    replacement: '${1}'
```

### Relabel Actions

| Action | Mô Tả |
|--------|-------|
| `keep` | Giữ targets matching regex |
| `drop` | Bỏ targets matching regex |
| `replace` | Replace label value |
| `labelmap` | Map labels by regex |
| `labeldrop` | Drop labels matching regex |
| `labelkeep` | Keep only labels matching regex |

## 7. Storage

### Local Storage (TSDB)

```
/var/lib/prometheus/
├── chunks_head/     # Recent data in memory
├── wal/             # Write-ahead log
├── 01BKGV7JBM69T2G1BGBGM6KB12/  # Completed blocks
│   ├── chunks/
│   ├── index
│   ├── meta.json
│   └── tombstones
└── lock
```

### Retention

```yaml
# Command line flags
--storage.tsdb.retention.time=15d    # Default: 15 days
--storage.tsdb.retention.size=10GB   # Size-based retention
```

### Remote Storage

```yaml
remote_write:
  - url: "http://remote-storage:9201/write"

remote_read:
  - url: "http://remote-storage:9201/read"
```

## 8. Federation

Cho phép một Prometheus scrape metrics từ Prometheus khác:

```yaml
scrape_configs:
  - job_name: 'federate'
    honor_labels: true
    metrics_path: '/federate'
    params:
      match[]:
        - '{job="prometheus"}'
        - '{__name__=~"job:.*"}'
    static_configs:
      - targets:
          - 'source-prometheus:9090'
```

## 9. Key Concepts Summary

| Concept | Key Points |
|---------|-----------|
| Architecture | Pull-based, scrapes /metrics endpoints |
| Data Model | metric_name{labels} → time series |
| Counter | Only increases, use rate() |
| Gauge | Can go up/down, query directly |
| Histogram | Distribution, server-side quantiles |
| Summary | Distribution, client-side quantiles |
| Scrape | HTTP GET to /metrics every interval |
| Service Discovery | Static, file-based, Kubernetes, Consul |
| Storage | Local TSDB, 15d default retention |

## 10. Practice Questions

**Q1**: Sự khác biệt giữa Counter và Gauge là gì?

<details>
<summary>Đáp án</summary>

Counter chỉ tăng (hoặc reset về 0 khi restart), dùng cho requests, errors, bytes. Gauge có thể tăng hoặc giảm, dùng cho memory usage, temperature, queue size. Với Counter luôn dùng rate() hoặc increase() để query, với Gauge query trực tiếp.

</details>

**Q2**: Tại sao không nên dùng high cardinality labels?

<details>
<summary>Đáp án</summary>

High cardinality labels (như user_id, session_id) tạo ra quá nhiều unique time series. Điều này làm tăng memory usage, storage, và query time của Prometheus. Ví dụ: 1 triệu users × 10 endpoints = 10 triệu time series chỉ cho 1 metric.

</details>

**Q3**: Khi nào dùng Histogram vs Summary?

<details>
<summary>Đáp án</summary>

Dùng Histogram khi: cần aggregate across multiple instances, cần flexible quantile calculation, không biết trước distribution. Dùng Summary khi: cần accurate quantiles, chỉ có 1 instance, quantiles được define trước. Histogram thường được recommend hơn vì aggregatable.

</details>

## Tiếp Theo

- [Domain 3: PromQL](./05-domain-promql.md)
- [Tài liệu tham khảo: Prometheus Architecture](../01-fundamentals/04-prometheus-architecture.md)
