# Các Loại Metrics

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Bốn Loại Metrics Cơ Bản](#bốn-loại-metrics-cơ-bản)
- [Counter](#counter)
- [Gauge](#gauge)
- [Histogram](#histogram)
- [Summary](#summary)
- [So Sánh Các Loại Metrics](#so-sánh-các-loại-metrics)
- [Khi Nào Dùng Loại Nào?](#khi-nào-dùng-loại-nào)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Giới Thiệu

Prometheus định nghĩa **4 loại metrics cơ bản**: Counter, Gauge, Histogram, và Summary. Mỗi loại phù hợp cho các trường hợp sử dụng khác nhau. Hiểu rõ từng loại giúp bạn chọn đúng metric cho monitoring.

## Bốn Loại Metrics Cơ Bản

```
┌────────────────────────────────────────┐
│  Prometheus Metric Types               │
├────────────────────────────────────────┤
│  1. Counter   → Đếm tăng dần           │
│  2. Gauge     → Giá trị lên xuống      │
│  3. Histogram → Phân phối giá trị      │
│  4. Summary   → Quantiles và tổng      │
└────────────────────────────────────────┘
```

## Counter

### Định Nghĩa

**Counter** là metric chỉ tăng (hoặc reset về 0). Dùng để đếm số lần một sự kiện xảy ra.

### Đặc Điểm

```
✓ Chỉ tăng (monotonically increasing)
✓ Có thể reset về 0 (khi restart)
✗ KHÔNG bao giờ giảm
```

### Ví Dụ Thực Tế

**Đồng hồ đo quãng đường xe hơi (Odometer)**:

```
Thời gian    Quãng đường
─────────────────────────
08:00        10,000 km
09:00        10,050 km  ← Tăng 50km
10:00        10,120 km  ← Tăng 70km
11:00        10,180 km  ← Tăng 60km
```

Odometer chỉ tăng, không bao giờ giảm (trừ khi reset).

### Use Cases

```
┌────────────────────────────────────────┐
│  Counter Use Cases                     │
├────────────────────────────────────────┤
│  • http_requests_total                 │
│    Tổng số HTTP requests               │
│                                        │
│  • errors_total                        │
│    Tổng số lỗi                         │
│                                        │
│  • bytes_sent_total                    │
│    Tổng số bytes gửi đi                │
│                                        │
│  • tasks_completed_total               │
│    Tổng số tasks hoàn thành            │
└────────────────────────────────────────┘
```

### Ví Dụ Code

```python
from prometheus_client import Counter

# Tạo counter
http_requests = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint']
)

# Tăng counter
http_requests.labels(method='GET', endpoint='/api').inc()
http_requests.labels(method='POST', endpoint='/api').inc(5)
```

### Visualization

```
http_requests_total
─────────────────────────────────────
Value
1000 ┤                         ╭────
 800 ┤                   ╭─────╯
 600 ┤             ╭─────╯
 400 ┤       ╭─────╯
 200 ┤ ╭─────╯
   0 ┤─╯
     └────────────────────────────────
     10:00  11:00  12:00  13:00  14:00
```

### PromQL với Counter

```promql
# Tốc độ tăng per second (rate)
rate(http_requests_total[5m])

# Tổng số requests trong 1 giờ
increase(http_requests_total[1h])

# Tổng requests của tất cả instances
sum(rate(http_requests_total[5m]))
```

## Gauge

### Định Nghĩa

**Gauge** là metric có thể tăng hoặc giảm. Dùng để đo giá trị hiện tại của một thứ gì đó.

### Đặc Điểm

```
✓ Có thể tăng
✓ Có thể giảm
✓ Đại diện giá trị tức thời
```

### Ví Dụ Thực Tế

**Nhiệt kế đo nhiệt độ**:

```
Thời gian    Nhiệt độ
─────────────────────
08:00        20°C
09:00        22°C  ← Tăng
10:00        25°C  ← Tăng
11:00        23°C  ← Giảm
12:00        24°C  ← Tăng
```

Nhiệt độ có thể lên xuống tự do.

### Use Cases

```
┌────────────────────────────────────────┐
│  Gauge Use Cases                       │
├────────────────────────────────────────┤
│  • cpu_usage_percent                   │
│    Mức sử dụng CPU (0-100%)            │
│                                        │
│  • memory_available_bytes              │
│    Bộ nhớ còn trống                    │
│                                        │
│  • active_connections                  │
│    Số connections đang active          │
│                                        │
│  • queue_size                          │
│    Số items trong queue                │
│                                        │
│  • temperature_celsius                 │
│    Nhiệt độ hiện tại                   │
└────────────────────────────────────────┘
```

### Ví Dụ Code

```python
from prometheus_client import Gauge

# Tạo gauge
cpu_usage = Gauge(
    'cpu_usage_percent',
    'CPU usage percentage',
    ['core']
)

# Set giá trị
cpu_usage.labels(core='0').set(75.5)

# Tăng/giảm
active_users = Gauge('active_users', 'Active users')
active_users.inc()    # Tăng 1
active_users.dec()    # Giảm 1
active_users.inc(10)  # Tăng 10
```

### Visualization

```
cpu_usage_percent
─────────────────────────────────────
Value
100% ┤     ╭─╮
 80% ┤   ╭─╯ ╰─╮
 60% ┤ ╭─╯     ╰─╮
 40% ┤─╯         ╰─╮
 20% ┤             ╰───
  0% ┤
     └────────────────────────────────
     10:00  11:00  12:00  13:00  14:00
```

### PromQL với Gauge

```promql
# Giá trị hiện tại
cpu_usage_percent

# Trung bình trong 5 phút
avg_over_time(cpu_usage_percent[5m])

# Giá trị max trong 1 giờ
max_over_time(cpu_usage_percent[1h])
```

## Histogram

### Định Nghĩa

**Histogram** đo phân phối của các giá trị trong các buckets (nhóm) định trước. Dùng để phân tích latency, request sizes, etc.

### Đặc Điểm

```
✓ Chia giá trị vào buckets
✓ Tính được quantiles (xấp xỉ)
✓ Hiệu quả cho aggregation
✓ Buckets được định nghĩa trước
```

### Ví Dụ Thực Tế

**Phân loại học sinh theo điểm số**:

```
Điểm      Số học sinh
─────────────────────────
0-50      ████ 4 người
51-70     ████████ 8 người
71-85     ████████████ 12 người
86-100    ██████ 6 người
```

### Use Cases

```
┌────────────────────────────────────────┐
│  Histogram Use Cases                   │
├────────────────────────────────────────┤
│  • http_request_duration_seconds       │
│    Thời gian xử lý request             │
│                                        │
│  • response_size_bytes                 │
│    Kích thước response                 │
│                                        │
│  • db_query_duration_seconds           │
│    Thời gian query database            │
└────────────────────────────────────────┘
```

### Cấu Trúc Histogram

Một histogram tạo ra **3 metrics**:

```
http_request_duration_seconds_bucket{le="0.1"}   100
http_request_duration_seconds_bucket{le="0.5"}   250
http_request_duration_seconds_bucket{le="1.0"}   400
http_request_duration_seconds_bucket{le="+Inf"}  450

http_request_duration_seconds_sum                180.5
http_request_duration_seconds_count              450
```

**Giải thích**:
- `_bucket{le="0.1"}`: 100 requests ≤ 0.1s
- `_bucket{le="0.5"}`: 250 requests ≤ 0.5s
- `_sum`: Tổng thời gian = 180.5s
- `_count`: Tổng số requests = 450

### Ví Dụ Code

```python
from prometheus_client import Histogram

# Tạo histogram với buckets
request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint'],
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0]
)

# Observe giá trị
request_duration.labels(method='GET', endpoint='/api').observe(0.25)
request_duration.labels(method='POST', endpoint='/api').observe(1.5)
```

### Visualization

```
Request Duration Distribution
─────────────────────────────────────
Count
120 ┤     ████
100 ┤     ████
 80 ┤     ████  ████
 60 ┤     ████  ████
 40 ┤ ████████  ████  ████
 20 ┤ ████████  ████  ████  ██
  0 ┤─────────────────────────────
     0.1s  0.5s  1.0s  2.0s  5.0s
```

### PromQL với Histogram

```promql
# Tính p95 latency (95th percentile)
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket[5m])
)

# Tính p50 (median)
histogram_quantile(0.5, 
  rate(http_request_duration_seconds_bucket[5m])
)

# Average latency
rate(http_request_duration_seconds_sum[5m]) /
rate(http_request_duration_seconds_count[5m])
```

## Summary

### Định Nghĩa

**Summary** tương tự histogram nhưng tính quantiles chính xác ở client-side. Dùng khi cần quantiles chính xác.

### Đặc Điểm

```
✓ Quantiles chính xác
✓ Tính toán ở client
✗ KHÔNG thể aggregate giữa nhiều instances
✗ Tốn tài nguyên hơn histogram
```

### Cấu Trúc Summary

Một summary tạo ra metrics:

```
http_request_duration_seconds{quantile="0.5"}    0.25
http_request_duration_seconds{quantile="0.9"}    0.8
http_request_duration_seconds{quantile="0.99"}   1.5

http_request_duration_seconds_sum                180.5
http_request_duration_seconds_count              450
```

### Ví Dụ Code

```python
from prometheus_client import Summary

# Tạo summary
request_duration = Summary(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint']
)

# Observe giá trị
request_duration.labels(method='GET', endpoint='/api').observe(0.25)
```

### Histogram vs Summary

```
┌────────────────────────────────────────────────┐
│  Histogram vs Summary                          │
├────────────────────────────────────────────────┤
│  Histogram:                                    │
│  ✓ Có thể aggregate                            │
│  ✓ Quantiles xấp xỉ                            │
│  ✓ Hiệu quả hơn                                │
│  ✗ Cần định nghĩa buckets trước                │
│                                                │
│  Summary:                                      │
│  ✓ Quantiles chính xác                         │
│  ✗ KHÔNG thể aggregate                         │
│  ✗ Tốn tài nguyên hơn                          │
│  ✗ Quantiles cố định                           │
└────────────────────────────────────────────────┘
```

**Khuyến nghị**: Dùng **Histogram** trong hầu hết trường hợp.

## So Sánh Các Loại Metrics

### Bảng So Sánh

| Đặc Điểm | Counter | Gauge | Histogram | Summary |
|----------|---------|-------|-----------|---------|
| **Tăng/Giảm** | Chỉ tăng | Cả hai | Chỉ tăng | Chỉ tăng |
| **Reset** | Có | Không | Có | Có |
| **Aggregation** | Có | Có | Có | Không |
| **Use case** | Đếm events | Giá trị hiện tại | Phân phối | Quantiles chính xác |

### Ví Dụ Tổng Hợp

```
Application Metrics:
─────────────────────────────────────────────────

Counter:
  http_requests_total{status="200"}      1,234,567
  http_requests_total{status="404"}      12,345
  http_requests_total{status="500"}      123

Gauge:
  active_connections                     42
  memory_usage_bytes                     1,073,741,824
  cpu_usage_percent                      65.5

Histogram:
  http_request_duration_seconds_bucket{le="0.1"}   1000
  http_request_duration_seconds_bucket{le="0.5"}   2500
  http_request_duration_seconds_sum                1250.5
  http_request_duration_seconds_count              3000

Summary:
  api_response_time_seconds{quantile="0.5"}        0.25
  api_response_time_seconds{quantile="0.99"}       1.5
  api_response_time_seconds_sum                    500.0
  api_response_time_seconds_count                  2000
```

## Khi Nào Dùng Loại Nào?

### Decision Tree

```
Bạn muốn đo gì?
│
├─ Đếm số lần xảy ra?
│  → Counter
│  Ví dụ: requests, errors, tasks completed
│
├─ Giá trị hiện tại?
│  → Gauge
│  Ví dụ: CPU, memory, connections, temperature
│
├─ Phân phối giá trị?
│  │
│  ├─ Cần aggregate nhiều instances?
│  │  → Histogram
│  │  Ví dụ: request latency, response size
│  │
│  └─ Cần quantiles chính xác cho 1 instance?
│     → Summary
│     Ví dụ: SLA monitoring
```

### Best Practices

```
┌────────────────────────────────────────────────┐
│  Metric Type Best Practices                    │
├────────────────────────────────────────────────┤
│  Counter:                                      │
│  • Đặt tên kết thúc bằng _total                │
│  • Dùng rate() để tính tốc độ                  │
│  • Đừng dùng cho giá trị có thể giảm           │
│                                                │
│  Gauge:                                        │
│  • Dùng cho giá trị snapshot                   │
│  • Đừng dùng cho giá trị chỉ tăng              │
│                                                │
│  Histogram:                                    │
│  • Chọn buckets phù hợp với use case           │
│  • Dùng cho latency và sizes                   │
│  • Prefer histogram over summary               │
│                                                │
│  Summary:                                      │
│  • Chỉ dùng khi thực sự cần quantiles chính xác│
│  • Không dùng cho distributed systems          │
└────────────────────────────────────────────────┘
```

## Tài Liệu Liên Quan

- [Monitoring and Observability](./01-monitoring-observability.md) - Tại sao cần metrics
- [Time-Series Databases](./02-time-series-databases.md) - Cách metrics được lưu trữ
- [Prometheus Architecture](./04-prometheus-architecture.md) - Cách Prometheus thu thập metrics

## Tài Liệu Tham Khảo

- [Prometheus Metric Types](https://prometheus.io/docs/concepts/metric_types/)
- [Prometheus Best Practices - Metric Naming](https://prometheus.io/docs/practices/naming/)
- [Histograms and Summaries](https://prometheus.io/docs/practices/histograms/)
