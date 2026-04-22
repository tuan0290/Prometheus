# Alerting Rules

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Cú Pháp Alert Rule](#cú-pháp-alert-rule)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Alert States](#alert-states)
- [Viết Alert Rules Hiệu Quả](#viết-alert-rules-hiệu-quả)
- [Các Pattern Alert Phổ Biến](#các-pattern-alert-phổ-biến)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)
- [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

## Tổng Quan

Alerting rules trong Prometheus cho phép bạn định nghĩa các điều kiện khi nào cần gửi cảnh báo. Khi một biểu thức PromQL trả về kết quả, Prometheus sẽ tạo ra alert và gửi đến Alertmanager để xử lý và phân phối thông báo.

**Lợi ích của Prometheus alerting:**
- Định nghĩa alerts bằng PromQL mạnh mẽ
- Hỗ trợ alert states với thời gian chờ (for duration)
- Labels và annotations linh hoạt
- Tích hợp chặt chẽ với Alertmanager
- Dễ dàng version control với YAML files

## Cú Pháp Alert Rule

### Cấu Trúc Cơ Bản

Alert rules được định nghĩa trong rule files và tham chiếu từ `prometheus.yml`:

```yaml
# prometheus.yml
rule_files:
  - "rules/*.yml"
  - "alerts.yml"
```

### Cấu Trúc Rule File

```yaml
groups:
  - name: <group_name>
    interval: <evaluation_interval>  # Tùy chọn, ghi đè global
    rules:
      - alert: <alert_name>
        expr: <promql_expression>
        for: <duration>
        labels:
          <label_key>: <label_value>
        annotations:
          <annotation_key>: <annotation_value>
```

### Các Trường Quan Trọng

| Trường | Bắt Buộc | Mô Tả |
|--------|----------|-------|
| `alert` | Có | Tên alert, phải là identifier hợp lệ |
| `expr` | Có | Biểu thức PromQL, trả về true khi alert cần kích hoạt |
| `for` | Không | Thời gian biểu thức phải đúng trước khi firing |
| `labels` | Không | Labels bổ sung gắn vào alert |
| `annotations` | Không | Metadata mô tả alert (summary, description) |

### Ví Dụ Alert Rule Đơn Giản

```yaml
groups:
  - name: example
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU usage cao trên {{ $labels.instance }}"
          description: "CPU usage là {{ $value | printf \"%.2f\" }}% trên instance {{ $labels.instance }}"
```

## Cách Hoạt Động

### Vòng Đời Alert

```
┌─────────────────────────────────────────────────────────────┐
│                    Prometheus Server                         │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │  Rule Files  │───▶│  Evaluation  │───▶│ Alert State  │  │
│  │  (YAML)      │    │  Engine      │    │  Manager     │  │
│  └──────────────┘    └──────────────┘    └──────┬───────┘  │
│                                                  │          │
└──────────────────────────────────────────────────┼──────────┘
                                                   │
                                                   ▼
                                          ┌──────────────────┐
                                          │  Alertmanager    │
                                          └──────────────────┘
```

### Quá Trình Evaluation

1. **Load Rules**: Prometheus đọc rule files khi khởi động
2. **Periodic Evaluation**: Mỗi `evaluation_interval` (mặc định 1m), Prometheus chạy tất cả expressions
3. **State Tracking**: Theo dõi kết quả qua thời gian
4. **Alert Firing**: Khi điều kiện đúng đủ thời gian `for`, alert chuyển sang firing
5. **Send to Alertmanager**: Prometheus gửi alerts đến Alertmanager qua HTTP

## Alert States

Mỗi alert có thể ở một trong ba trạng thái:

```
                    expr = false
                    ┌─────────────────────────────────┐
                    │                                 │
                    ▼                                 │
┌──────────────┐  expr = true  ┌──────────────┐      │
│   INACTIVE   │──────────────▶│   PENDING    │──────┘
│              │               │              │
│ Alert không  │               │ Đang chờ     │  for duration
│ kích hoạt    │               │ đủ thời gian │  elapsed
└──────────────┘               └──────┬───────┘
        ▲                             │
        │                             ▼
        │                      ┌──────────────┐
        │    expr = false       │   FIRING     │
        └───────────────────────│              │
                                │ Alert đang   │
                                │ được gửi đi  │
                                └──────────────┘
```

### Chi Tiết Từng State

**INACTIVE**
- Biểu thức PromQL trả về không có kết quả hoặc false
- Alert không được gửi đến Alertmanager
- Trạng thái bình thường khi hệ thống hoạt động tốt

**PENDING**
- Biểu thức PromQL trả về kết quả (true)
- Đang chờ đủ thời gian `for` duration
- Chưa gửi đến Alertmanager
- Giúp tránh false positives từ spike ngắn

**FIRING**
- Biểu thức đúng liên tục đủ thời gian `for`
- Alert được gửi đến Alertmanager
- Tiếp tục gửi cho đến khi biểu thức trở về false

### Xem Alert States

Truy cập Prometheus UI tại `http://localhost:9090/alerts` để xem trạng thái tất cả alerts.

```bash
# Kiểm tra alerts qua API
curl http://localhost:9090/api/v1/alerts

# Kiểm tra rules
curl http://localhost:9090/api/v1/rules
```

## Viết Alert Rules Hiệu Quả

### Sử Dụng `for` Duration

Luôn sử dụng `for` để tránh false positives:

```yaml
# Không tốt - có thể false positive từ spike ngắn
- alert: HighMemory
  expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.1

# Tốt hơn - chờ 5 phút để xác nhận
- alert: HighMemory
  expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.1
  for: 5m
```

### Labels Severity

Sử dụng labels để phân loại mức độ nghiêm trọng:

```yaml
labels:
  severity: critical   # critical, warning, info
  team: platform       # Team chịu trách nhiệm
  service: api         # Service liên quan
```

### Annotations Hữu Ích

```yaml
annotations:
  summary: "Mô tả ngắn gọn về alert"
  description: "Mô tả chi tiết với giá trị: {{ $value }}"
  runbook_url: "https://wiki.example.com/runbooks/high-cpu"
  dashboard_url: "https://grafana.example.com/d/abc123"
```

### Template Variables

Sử dụng Go template trong annotations:

```yaml
annotations:
  summary: "Instance {{ $labels.instance }} down"
  description: >
    {{ $labels.job }} trên {{ $labels.instance }} đã down
    trong hơn 5 phút. Giá trị hiện tại: {{ $value }}.
```

## Các Pattern Alert Phổ Biến

### 1. CPU Usage Cao

```yaml
groups:
  - name: cpu_alerts
    rules:
      - alert: HighCPUUsage
        expr: |
          100 - (avg by(instance) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          ) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU usage cao trên {{ $labels.instance }}"
          description: "CPU usage là {{ $value | printf \"%.1f\" }}% (ngưỡng: 80%)"

      - alert: CriticalCPUUsage
        expr: |
          100 - (avg by(instance) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          ) * 100) > 95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "CPU usage nghiêm trọng trên {{ $labels.instance }}"
          description: "CPU usage là {{ $value | printf \"%.1f\" }}% (ngưỡng: 95%)"
```

### 2. Disk Space Thấp

```yaml
groups:
  - name: disk_alerts
    rules:
      - alert: LowDiskSpace
        expr: |
          (node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"}
          / node_filesystem_size_bytes{fstype!~"tmpfs|fuse.lxcfs"}) * 100 < 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space thấp trên {{ $labels.instance }}"
          description: >
            Filesystem {{ $labels.mountpoint }} trên {{ $labels.instance }}
            chỉ còn {{ $value | printf \"%.1f\" }}% dung lượng trống.

      - alert: CriticalDiskSpace
        expr: |
          (node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"}
          / node_filesystem_size_bytes{fstype!~"tmpfs|fuse.lxcfs"}) * 100 < 5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Disk space nghiêm trọng trên {{ $labels.instance }}"
          description: >
            Filesystem {{ $labels.mountpoint }} trên {{ $labels.instance }}
            chỉ còn {{ $value | printf \"%.1f\" }}% dung lượng trống.
```

### 3. Service Down

```yaml
groups:
  - name: service_alerts
    rules:
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} down"
          description: >
            Instance {{ $labels.instance }} của job {{ $labels.job }}
            đã không thể scrape trong hơn 1 phút.

      - alert: PrometheusTargetMissing
        expr: up == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Prometheus target mất kết nối"
          description: "Target {{ $labels.instance }} không thể kết nối."
```

### 4. Memory Usage Cao

```yaml
groups:
  - name: memory_alerts
    rules:
      - alert: HighMemoryUsage
        expr: |
          (1 - (node_memory_MemAvailable_bytes
          / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage cao trên {{ $labels.instance }}"
          description: "Memory usage là {{ $value | printf \"%.1f\" }}%"

      - alert: CriticalMemoryUsage
        expr: |
          (1 - (node_memory_MemAvailable_bytes
          / node_memory_MemTotal_bytes)) * 100 > 95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Memory usage nghiêm trọng trên {{ $labels.instance }}"
          description: "Memory usage là {{ $value | printf \"%.1f\" }}%"
```

### 5. HTTP Error Rate Cao

```yaml
groups:
  - name: http_alerts
    rules:
      - alert: HighHTTPErrorRate
        expr: |
          sum by(job, instance) (
            rate(http_requests_total{status=~"5.."}[5m])
          )
          /
          sum by(job, instance) (
            rate(http_requests_total[5m])
          ) * 100 > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "HTTP error rate cao cho {{ $labels.job }}"
          description: "Error rate là {{ $value | printf \"%.2f\" }}%"
```

### 6. Latency Cao

```yaml
groups:
  - name: latency_alerts
    rules:
      - alert: HighRequestLatency
        expr: |
          histogram_quantile(0.99,
            sum by(job, le) (
              rate(http_request_duration_seconds_bucket[5m])
            )
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Latency p99 cao cho {{ $labels.job }}"
          description: "p99 latency là {{ $value | printf \"%.3f\" }}s"
```

## Use Cases

### Khi Nào Dùng Alert Rules

- **Infrastructure monitoring**: CPU, memory, disk, network
- **Application monitoring**: Error rates, latency, throughput
- **Business metrics**: Số đơn hàng, doanh thu, user activity
- **SLO monitoring**: Theo dõi Service Level Objectives
- **Capacity planning**: Cảnh báo trước khi hết tài nguyên

### Khi Nào Không Dùng Alert Rules

- Thông tin chỉ cần xem trên dashboard (không cần action)
- Metrics thay đổi liên tục và bình thường (noise)
- Khi chưa có runbook để xử lý alert

## Cấu Hình

### Cấu Hình Prometheus Để Load Rules

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s  # Tần suất evaluate rules

rule_files:
  - "rules/node_alerts.yml"
  - "rules/app_alerts.yml"
  - "rules/*.yml"  # Wildcard pattern

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
```

### Cấu Trúc Thư Mục Rules

```
prometheus/
├── prometheus.yml
└── rules/
    ├── node_alerts.yml      # Infrastructure alerts
    ├── app_alerts.yml       # Application alerts
    ├── database_alerts.yml  # Database alerts
    └── slo_alerts.yml       # SLO alerts
```

### Reload Rules Không Restart

```bash
# Gửi SIGHUP để reload
kill -HUP $(pidof prometheus)

# Hoặc dùng HTTP API
curl -X POST http://localhost:9090/-/reload
```

### Validate Rule Files

```bash
# Dùng promtool để validate
promtool check rules rules/node_alerts.yml

# Validate tất cả files
promtool check rules rules/*.yml
```

## Best Practices

### 1. Đặt Tên Alert Rõ Ràng

```yaml
# Không tốt
- alert: Alert1
- alert: HighValue

# Tốt
- alert: NodeCPUUsageHigh
- alert: KubernetesPodCrashLooping
- alert: DatabaseConnectionPoolExhausted
```

### 2. Phân Cấp Severity

```yaml
# Sử dụng 3 cấp độ nhất quán
labels:
  severity: info      # Thông tin, không cần action ngay
  severity: warning   # Cần chú ý, có thể cần action
  severity: critical  # Cần action ngay lập tức
```

### 3. Tránh Alert Fatigue

- Chỉ alert khi cần action thực sự
- Sử dụng `for` duration phù hợp
- Nhóm alerts liên quan
- Có runbook cho mỗi alert

### 4. Sử Dụng Recording Rules

Tạo recording rules cho expressions phức tạp dùng nhiều lần:

```yaml
groups:
  - name: recording_rules
    rules:
      # Recording rule
      - record: job:http_requests_total:rate5m
        expr: sum by(job) (rate(http_requests_total[5m]))

  - name: alert_rules
    rules:
      # Dùng recording rule trong alert
      - alert: HighRequestRate
        expr: job:http_requests_total:rate5m > 1000
        for: 5m
```

### 5. Test Alert Rules

```bash
# Tạo test file
cat > test_rules.yml << 'EOF'
rule_files:
  - rules/node_alerts.yml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      - series: 'node_cpu_seconds_total{mode="idle", instance="host1"}'
        values: '0+0x10'  # CPU 100% busy
    alert_rule_test:
      - eval_time: 10m
        alertname: HighCPUUsage
        exp_alerts:
          - exp_labels:
              severity: warning
              instance: host1
EOF

promtool test rules test_rules.yml
```

### 6. Tổ Chức Rules Theo Nhóm

```yaml
groups:
  - name: node.rules
    interval: 30s
    rules:
      - alert: NodeDown
        # ...

  - name: application.rules
    interval: 1m
    rules:
      - alert: AppHighLatency
        # ...
```

## Tài Liệu Liên Quan

- [Alertmanager Config](./02-alertmanager-config.md) - Cấu hình routing và grouping
- [Notification Channels](./03-notification-channels.md) - Email, Slack, PagerDuty
- [Alertmanager Component](../02-components/03-alertmanager.md) - Tổng quan Alertmanager
- [PromQL Basics](../03-promql/01-promql-basics.md) - Cú pháp PromQL cho alert expressions
- [Lab 04 - Alertmanager](../07-labs/lab-04-alertmanager/README.md) - Bài tập thực hành

## Tài Liệu Tham Khảo

- [Prometheus Alerting Rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/) - Official documentation
- [Alerting Best Practices](https://prometheus.io/docs/practices/alerting/) - Prometheus best practices
- [promtool](https://prometheus.io/docs/prometheus/latest/command-line/promtool/) - CLI tool để validate rules
- [Awesome Prometheus Alerts](https://awesome-prometheus-alerts.grep.to/) - Thư viện alert rules cộng đồng
