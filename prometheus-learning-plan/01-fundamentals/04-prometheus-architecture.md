# Kiến Trúc Prometheus

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Tổng Quan Kiến Trúc](#tổng-quan-kiến-trúc)
- [Các Thành Phần Chính](#các-thành-phần-chính)
- [Pull-Based Model](#pull-based-model)
- [Data Flow](#data-flow)
- [Service Discovery](#service-discovery)
- [Storage](#storage)
- [Querying và Visualization](#querying-và-visualization)
- [Alerting](#alerting)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Giới Thiệu

Prometheus là một hệ thống monitoring và alerting mã nguồn mở được thiết kế cho reliability và scalability. Kiến trúc của Prometheus dựa trên mô hình **pull-based**, khác với nhiều hệ thống monitoring truyền thống sử dụng push-based.

## Tổng Quan Kiến Trúc

### Sơ Đồ Tổng Thể

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROMETHEUS ECOSYSTEM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐         ┌──────────────────────────┐         │
│  │  Prometheus  │         │   Service Discovery      │         │
│  │    Server    │◄────────┤  (Kubernetes, Consul,    │         │
│  │              │         │   File, Static, etc.)    │         │
│  └───────┬──────┘         └──────────────────────────┘         │
│          │                                                      │
│          │ Pull metrics (HTTP)                                 │
│          ↓                                                      │
│  ┌──────────────────────────────────────────────────┐          │
│  │              Targets (Exporters)                 │          │
│  ├──────────────────────────────────────────────────┤          │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐       │          │
│  │  │   Node   │  │Blackbox  │  │  Custom  │       │          │
│  │  │ Exporter │  │ Exporter │  │ Exporter │  ...  │          │
│  │  └──────────┘  └──────────┘  └──────────┘       │          │
│  │                                                  │          │
│  │  ┌──────────────────────────────────┐           │          │
│  │  │  Application with Client Library │           │          │
│  │  │  (Go, Python, Java, etc.)        │           │          │
│  │  └──────────────────────────────────┘           │          │
│  └──────────────────────────────────────────────────┘          │
│                                                                 │
│  ┌──────────────┐         ┌──────────────┐                     │
│  │  Prometheus  │────────►│ Alertmanager │                     │
│  │    Server    │  Alerts │              │                     │
│  └───────┬──────┘         └──────┬───────┘                     │
│          │                       │                             │
│          │ PromQL                │ Notifications               │
│          ↓                       ↓                             │
│  ┌──────────────┐         ┌──────────────┐                     │
│  │   Grafana    │         │    Email     │                     │
│  │  Dashboard   │         │    Slack     │                     │
│  └──────────────┘         │  PagerDuty   │                     │
│                           └──────────────┘                     │
│                                                                 │
│  ┌──────────────────────────────────────┐                      │
│  │  Pushgateway (for short-lived jobs)  │                      │
│  └──────────────────────────────────────┘                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Các Thành Phần Chính

```
┌────────────────────────────────────────────────┐
│  Core Components                               │
├────────────────────────────────────────────────┤
│  1. Prometheus Server                          │
│     • Retrieval (scraping)                     │
│     • TSDB (storage)                           │
│     • HTTP Server (API & UI)                   │
│                                                │
│  2. Client Libraries                           │
│     • Instrument application code              │
│                                                │
│  3. Exporters                                  │
│     • Expose metrics from 3rd party systems    │
│                                                │
│  4. Pushgateway                                │
│     • For short-lived jobs                     │
│                                                │
│  5. Alertmanager                               │
│     • Handle alerts                            │
│                                                │
│  6. Service Discovery                          │
│     • Discover targets dynamically             │
└────────────────────────────────────────────────┘
```

## Các Thành Phần Chính

### 1. Prometheus Server

Trái tim của hệ thống, gồm 3 phần chính:

```
┌─────────────────────────────────────────┐
│       PROMETHEUS SERVER                 │
├─────────────────────────────────────────┤
│                                         │
│  ┌───────────────────────────────┐     │
│  │  Retrieval (Scraper)          │     │
│  │  • Pull metrics from targets  │     │
│  │  • Every 15s (configurable)   │     │
│  └────────────┬──────────────────┘     │
│               ↓                         │
│  ┌───────────────────────────────┐     │
│  │  TSDB (Time-Series Database)  │     │
│  │  • Store metrics locally      │     │
│  │  • Compression                │     │
│  │  • Retention policies         │     │
│  └────────────┬──────────────────┘     │
│               ↓                         │
│  ┌───────────────────────────────┐     │
│  │  HTTP Server                  │     │
│  │  • PromQL API                 │     │
│  │  • Web UI                     │     │
│  │  • Federation API             │     │
│  └───────────────────────────────┘     │
│                                         │
└─────────────────────────────────────────┘
```

**Chức năng**:
- **Retrieval**: Scrape (pull) metrics từ targets theo interval
- **TSDB**: Lưu trữ time-series data hiệu quả
- **HTTP Server**: Cung cấp API và Web UI

### 2. Exporters

Exporters expose metrics từ hệ thống không hỗ trợ Prometheus native.

```
┌────────────────────────────────────────┐
│  Common Exporters                      │
├────────────────────────────────────────┤
│  • Node Exporter                       │
│    → System metrics (CPU, memory, disk)│
│                                        │
│  • Blackbox Exporter                   │
│    → Probe endpoints (HTTP, TCP, ICMP) │
│                                        │
│  • MySQL Exporter                      │
│    → MySQL database metrics            │
│                                        │
│  • PostgreSQL Exporter                 │
│    → PostgreSQL metrics                │
│                                        │
│  • Custom Exporters                    │
│    → Application-specific metrics      │
└────────────────────────────────────────┘
```

**Workflow**:

```
┌──────────────┐      HTTP GET      ┌──────────────┐
│  Prometheus  │ ──────────────────► │   Exporter   │
│    Server    │                     │              │
└──────────────┘                     └──────┬───────┘
                                            │
                                            │ Collect
                                            ↓
                                     ┌──────────────┐
                                     │  Target      │
                                     │  System      │
                                     │  (MySQL, OS) │
                                     └──────────────┘
```

### 3. Client Libraries

Thư viện để instrument application code.

```
┌────────────────────────────────────────┐
│  Official Client Libraries             │
├────────────────────────────────────────┤
│  • Go                                  │
│  • Python                              │
│  • Java / Scala                        │
│  • Ruby                                │
│  • .NET / C#                           │
└────────────────────────────────────────┘
```

**Ví dụ Python**:

```python
from prometheus_client import Counter, start_http_server

# Define metric
requests = Counter('http_requests_total', 'Total requests')

# Increment metric
requests.inc()

# Expose metrics on :8000/metrics
start_http_server(8000)
```

### 4. Pushgateway

Cho phép short-lived jobs push metrics.

```
┌──────────────┐      Push       ┌──────────────┐
│  Batch Job   │ ───────────────► │ Pushgateway  │
│ (short-lived)│                  │              │
└──────────────┘                  └──────┬───────┘
                                         │
                                         │ Pull
                                         ↓
                                  ┌──────────────┐
                                  │  Prometheus  │
                                  │    Server    │
                                  └──────────────┘
```

**Use case**: Cron jobs, batch processing

### 5. Alertmanager

Xử lý alerts từ Prometheus.

```
┌──────────────┐      Alerts     ┌──────────────┐
│  Prometheus  │ ───────────────► │ Alertmanager │
│    Server    │                  │              │
└──────────────┘                  └──────┬───────┘
                                         │
                                         │ Route & Notify
                                         ↓
                          ┌──────────────────────────┐
                          │  • Email                 │
                          │  • Slack                 │
                          │  • PagerDuty             │
                          │  • Webhook               │
                          └──────────────────────────┘
```

**Chức năng**:
- **Grouping**: Gom alerts liên quan
- **Inhibition**: Tắt alerts phụ thuộc
- **Silencing**: Tạm tắt alerts
- **Routing**: Gửi đến đúng receiver

## Pull-Based Model

### Pull vs Push

```
┌────────────────────────────────────────────────┐
│  PULL-BASED (Prometheus)                       │
├────────────────────────────────────────────────┤
│                                                │
│  ┌──────────────┐         ┌──────────────┐    │
│  │  Prometheus  │ ──────► │   Target     │    │
│  │    Server    │  Pull   │  (Exporter)  │    │
│  └──────────────┘         └──────────────┘    │
│                                                │
│  Ưu điểm:                                      │
│  ✓ Prometheus kiểm soát scrape rate           │
│  ✓ Dễ phát hiện target down                   │
│  ✓ Targets không cần biết Prometheus          │
│  ✓ Có thể scrape nhiều lần từ nhiều Prometheus│
│                                                │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│  PUSH-BASED (Traditional)                      │
├────────────────────────────────────────────────┤
│                                                │
│  ┌──────────────┐         ┌──────────────┐    │
│  │   Target     │ ──────► │  Monitoring  │    │
│  │  (Agent)     │  Push   │    Server    │    │
│  └──────────────┘         └──────────────┘    │
│                                                │
│  Nhược điểm:                                   │
│  ✗ Targets phải biết monitoring server         │
│  ✗ Khó scale với nhiều targets                 │
│  ✗ Network issues ảnh hưởng data collection    │
│                                                │
└────────────────────────────────────────────────┘
```

### Scrape Process

```
1. Prometheus đọc config
   ↓
2. Service discovery tìm targets
   ↓
3. Mỗi scrape_interval (default 15s):
   ↓
4. HTTP GET /metrics từ mỗi target
   ↓
5. Parse metrics (Prometheus format)
   ↓
6. Lưu vào TSDB với timestamp
```

**Ví dụ scrape config**:

```yaml
scrape_configs:
  - job_name: 'node'
    scrape_interval: 15s
    static_configs:
      - targets:
        - 'localhost:9100'
        - 'server1:9100'
        - 'server2:9100'
```

## Data Flow

### End-to-End Flow

```
┌─────────────────────────────────────────────────────────┐
│  1. APPLICATION                                         │
│     ┌──────────────────────────────────┐               │
│     │  app.py                          │               │
│     │  requests.inc()  # Increment     │               │
│     └──────────────────────────────────┘               │
│                    ↓                                    │
├─────────────────────────────────────────────────────────┤
│  2. CLIENT LIBRARY                                      │
│     ┌──────────────────────────────────┐               │
│     │  Store in memory                 │               │
│     │  http_requests_total = 1234      │               │
│     └──────────────────────────────────┘               │
│                    ↓                                    │
├─────────────────────────────────────────────────────────┤
│  3. METRICS ENDPOINT                                    │
│     ┌──────────────────────────────────┐               │
│     │  GET /metrics                    │               │
│     │  http_requests_total 1234        │               │
│     └──────────────────────────────────┘               │
│                    ↓                                    │
├─────────────────────────────────────────────────────────┤
│  4. PROMETHEUS SCRAPE                                   │
│     ┌──────────────────────────────────┐               │
│     │  Every 15s: HTTP GET /metrics    │               │
│     └──────────────────────────────────┘               │
│                    ↓                                    │
├─────────────────────────────────────────────────────────┤
│  5. TSDB STORAGE                                        │
│     ┌──────────────────────────────────┐               │
│     │  timestamp: 1610000000           │               │
│     │  metric: http_requests_total     │               │
│     │  value: 1234                     │               │
│     └──────────────────────────────────┘               │
│                    ↓                                    │
├─────────────────────────────────────────────────────────┤
│  6. QUERY (PromQL)                                      │
│     ┌──────────────────────────────────┐               │
│     │  rate(http_requests_total[5m])   │               │
│     └──────────────────────────────────┘               │
│                    ↓                                    │
├─────────────────────────────────────────────────────────┤
│  7. VISUALIZATION                                       │
│     ┌──────────────────────────────────┐               │
│     │  Grafana Dashboard               │               │
│     │  [Graph showing request rate]    │               │
│     └──────────────────────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

## Service Discovery

Prometheus hỗ trợ nhiều cơ chế service discovery:

```
┌────────────────────────────────────────┐
│  Service Discovery Mechanisms          │
├────────────────────────────────────────┤
│  • Static Config                       │
│    → Hardcoded targets                 │
│                                        │
│  • File-based                          │
│    → Read from JSON/YAML files         │
│                                        │
│  • Kubernetes                          │
│    → Auto-discover pods/services       │
│                                        │
│  • Consul                              │
│    → Service registry integration      │
│                                        │
│  • EC2, Azure, GCE                     │
│    → Cloud provider APIs               │
│                                        │
│  • DNS                                 │
│    → DNS SRV records                   │
└────────────────────────────────────────┘
```

**Ví dụ Kubernetes SD**:

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

## Storage

### Local Storage

```
prometheus/
└── data/
    ├── wal/                    # Write-Ahead Log
    │   ├── 00000001
    │   └── 00000002
    ├── 01HQXXX/                # Block (2h window)
    │   ├── chunks/
    │   │   └── 000001
    │   ├── index
    │   ├── meta.json
    │   └── tombstones
    └── 01HQYYY/                # Another block
        ├── chunks/
        ├── index
        └── meta.json
```

### Retention

```yaml
# prometheus.yml
storage:
  tsdb:
    retention.time: 15d      # Keep 15 days
    retention.size: 50GB     # Or max 50GB
```

### Remote Storage

```
┌──────────────┐      Remote Write    ┌──────────────┐
│  Prometheus  │ ───────────────────► │   Remote     │
│    Server    │                      │   Storage    │
│              │ ◄─────────────────── │ (Thanos, M3) │
└──────────────┘      Remote Read     └──────────────┘
```

## Querying và Visualization

### PromQL

```
┌──────────────┐      PromQL Query    ┌──────────────┐
│   Grafana    │ ───────────────────► │  Prometheus  │
│  Dashboard   │                      │    Server    │
│              │ ◄─────────────────── │              │
└──────────────┘      JSON Response   └──────────────┘
```

**Ví dụ queries**:

```promql
# Instant query
http_requests_total

# Range query
rate(http_requests_total[5m])

# Aggregation
sum(rate(http_requests_total[5m])) by (status)
```

### Web UI

Prometheus có built-in Web UI:

```
http://localhost:9090

Pages:
  /graph       → Query và visualize
  /alerts      → Active alerts
  /targets     → Scrape targets status
  /config      → Current configuration
  /rules       → Recording/alerting rules
```

## Alerting

### Alert Flow

```
┌──────────────────────────────────────────────────┐
│  1. ALERTING RULES (prometheus.yml)              │
│     ┌──────────────────────────────────┐         │
│     │  alert: HighErrorRate            │         │
│     │  expr: rate(errors[5m]) > 0.05   │         │
│     └──────────────────────────────────┘         │
│                    ↓                              │
├──────────────────────────────────────────────────┤
│  2. PROMETHEUS EVALUATION                        │
│     ┌──────────────────────────────────┐         │
│     │  Every evaluation_interval       │         │
│     │  Check if condition is true      │         │
│     └──────────────────────────────────┘         │
│                    ↓                              │
├──────────────────────────────────────────────────┤
│  3. ALERT STATE                                  │
│     ┌──────────────────────────────────┐         │
│     │  Inactive → Pending → Firing     │         │
│     └──────────────────────────────────┘         │
│                    ↓                              │
├──────────────────────────────────────────────────┤
│  4. SEND TO ALERTMANAGER                         │
│     ┌──────────────────────────────────┐         │
│     │  POST /api/v1/alerts             │         │
│     └──────────────────────────────────┘         │
│                    ↓                              │
├──────────────────────────────────────────────────┤
│  5. ALERTMANAGER PROCESSING                      │
│     ┌──────────────────────────────────┐         │
│     │  • Group similar alerts          │         │
│     │  • Apply inhibition rules        │         │
│     │  • Route to receivers            │         │
│     └──────────────────────────────────┘         │
│                    ↓                              │
├──────────────────────────────────────────────────┤
│  6. NOTIFICATION                                 │
│     ┌──────────────────────────────────┐         │
│     │  Email, Slack, PagerDuty, etc.   │         │
│     └──────────────────────────────────┘         │
└──────────────────────────────────────────────────┘
```

## Tài Liệu Liên Quan

- [Monitoring and Observability](./01-monitoring-observability.md) - Tại sao cần monitoring
- [Time-Series Databases](./02-time-series-databases.md) - Cách Prometheus lưu trữ data
- [Metrics Types](./03-metrics-types.md) - Các loại metrics Prometheus hỗ trợ
- [Glossary](./05-glossary.md) - Thuật ngữ Prometheus

## Tài Liệu Tham Khảo

- [Prometheus Architecture Overview](https://prometheus.io/docs/introduction/overview/)
- [Prometheus Components](https://prometheus.io/docs/introduction/overview/#components)
- [Prometheus Storage](https://prometheus.io/docs/prometheus/latest/storage/)
