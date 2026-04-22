# Functions - Hàm Trong PromQL

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Rate và Increase](#rate-và-increase)
- [Aggregation Functions](#aggregation-functions)
- [Math Functions](#math-functions)
- [Time Functions](#time-functions)
- [Histogram Functions](#histogram-functions)
- [Label Manipulation](#label-manipulation)
- [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Giới Thiệu

PromQL cung cấp nhiều built-in functions để xử lý và phân tích time-series data. Functions giúp bạn:
- Tính toán rates và increases
- Thực hiện aggregations
- Xử lý time-series data
- Làm việc với histograms
- Manipulate labels

## Rate và Increase

### rate()

Tính tốc độ thay đổi trung bình per-second của counter trong time range.

**Cú pháp:**

```promql
rate(range_vector)
```

**Ví dụ:**

```promql
rate(http_requests_total[5m])
```

**Giải thích:**
- Tính số requests per second trong 5 phút gần nhất
- Tự động xử lý counter resets
- Kết quả là instant vector

**Kết quả mẫu:**
```
{instance="server1", method="GET"} 12.5  # 12.5 requests/second
{instance="server2", method="GET"} 8.3   # 8.3 requests/second
```

**Use cases:**
- Tính request rate
- Tính throughput
- Monitoring traffic patterns

**Best practice:** Sử dụng time range ít nhất 4x scrape interval (ví dụ: nếu scrape interval là 15s, dùng `[1m]` trở lên).

### irate()

Tính tốc độ thay đổi tức thời (instantaneous rate) dựa trên 2 data points cuối cùng.

**Cú pháp:**

```promql
irate(range_vector)
```

**Ví dụ:**

```promql
irate(http_requests_total[5m])
```

**Giải thích:**
- Chỉ sử dụng 2 samples cuối cùng trong range
- Phản ứng nhanh hơn với thay đổi
- Nhạy cảm hơn với spikes

**So sánh rate() vs irate():**

| Đặc điểm | rate() | irate() |
|---|---|---|
| Tính toán | Trung bình trên toàn bộ range | Chỉ 2 points cuối |
| Độ mượt | Smooth, ổn định | Spiky, nhạy cảm |
| Use case | Alerts, long-term trends | Graphs, debugging |

**Ví dụ so sánh:**

```promql
# Smooth, tốt cho alerts
rate(http_requests_total[5m])

# Responsive, tốt cho graphs
irate(http_requests_total[5m])
```

### increase()

Tính tổng số tăng của counter trong time range.

**Cú pháp:**

```promql
increase(range_vector)
```

**Ví dụ:**

```promql
increase(http_requests_total[1h])
```

**Giải thích:**
- Tính tổng số requests trong 1 giờ
- Tương đương `rate() * seconds_in_range`
- Tự động xử lý counter resets

**Kết quả mẫu:**
```
{instance="server1", method="GET"} 45000  # 45k requests trong 1h
{instance="server2", method="GET"} 30000  # 30k requests trong 1h
```

**Use cases:**
- Đếm tổng số events trong khoảng thời gian
- Tính total requests, errors, transactions

**Quan hệ với rate():**

```promql
increase(metric[5m]) == rate(metric[5m]) * 300
```

(300 seconds = 5 minutes)

## Aggregation Functions

### sum()

Tính tổng các giá trị.

**Cú pháp:**

```promql
sum(instant_vector)
```

**Ví dụ:**

```promql
sum(http_requests_total)
```

**Giải thích:** Tổng tất cả requests từ tất cả instances.

**Kết quả mẫu:**
```
{} 123456  # Scalar - tổng từ tất cả time-series
```

**Với grouping:**

```promql
sum by(method) (http_requests_total)
```

**Kết quả mẫu:**
```
{method="GET"} 80000
{method="POST"} 30000
{method="DELETE"} 5000
```

### avg()

Tính trung bình các giá trị.

**Cú pháp:**

```promql
avg(instant_vector)
```

**Ví dụ:**

```promql
avg(node_cpu_usage)
```

**Giải thích:** CPU usage trung bình của tất cả nodes.

**Use cases:**
- Average response time
- Average CPU/memory usage
- Average queue length

### min() và max()

Tìm giá trị nhỏ nhất và lớn nhất.

**Ví dụ:**

```promql
# Tìm instance có CPU cao nhất
max(node_cpu_usage)

# Tìm instance có memory thấp nhất
min(node_memory_available_bytes)
```

**Với grouping:**

```promql
max by(datacenter) (node_cpu_usage)
```

**Giải thích:** CPU cao nhất trong mỗi datacenter.

### count()

Đếm số lượng time-series.

**Ví dụ:**

```promql
count(up{job="api"})
```

**Giải thích:** Đếm số instances của job "api".

**Use cases:**
- Đếm số instances đang chạy
- Đếm số services
- Đếm số alerts firing

### count_values()

Đếm số lượng time-series có cùng giá trị.

**Cú pháp:**

```promql
count_values("label_name", instant_vector)
```

**Ví dụ:**

```promql
count_values("version", prometheus_build_info)
```

**Kết quả mẫu:**
```
{version="2.30.0"} 5  # 5 instances chạy version 2.30.0
{version="2.31.0"} 3  # 3 instances chạy version 2.31.0
```

**Use case:** Phát hiện version drift trong cluster.

## Math Functions

### abs()

Trả về giá trị tuyệt đối.

**Ví dụ:**

```promql
abs(delta(cpu_temp_celsius[5m]))
```

**Giải thích:** Độ thay đổi tuyệt đối của nhiệt độ CPU.

### ceil(), floor(), round()

Làm tròn số.

**Ví dụ:**

```promql
# Làm tròn lên
ceil(node_memory_available_bytes / 1024 / 1024 / 1024)

# Làm tròn xuống
floor(node_memory_available_bytes / 1024 / 1024 / 1024)

# Làm tròn gần nhất
round(node_memory_available_bytes / 1024 / 1024 / 1024)
```

### clamp_min() và clamp_max()

Giới hạn giá trị trong khoảng.

**Ví dụ:**

```promql
# Đảm bảo giá trị >= 0
clamp_min(rate(http_requests_total[5m]), 0)

# Đảm bảo giá trị <= 100
clamp_max(cpu_usage_percent, 100)
```

**Use case:** Xử lý outliers hoặc invalid values.

### sqrt(), exp(), ln(), log2(), log10()

Các hàm toán học.

**Ví dụ:**

```promql
# Căn bậc hai
sqrt(metric)

# Exponential
exp(metric)

# Logarithm tự nhiên
ln(metric)

# Log base 2
log2(metric)

# Log base 10
log10(metric)
```

## Time Functions

### time()

Trả về Unix timestamp hiện tại.

**Ví dụ:**

```promql
time()
```

**Kết quả:** `1609459200` (Unix timestamp)

**Use case:** Tính thời gian từ một event.

```promql
time() - process_start_time_seconds
```

**Giải thích:** Uptime của process (seconds).

### day_of_week(), day_of_month(), hour(), minute()

Trích xuất thành phần thời gian.

**Ví dụ:**

```promql
# Ngày trong tuần (0 = Sunday, 6 = Saturday)
day_of_week()

# Ngày trong tháng (1-31)
day_of_month()

# Giờ (0-23)
hour()

# Phút (0-59)
minute()
```

**Use case:** Alerts chỉ trong business hours.

```promql
# Alert chỉ từ 9am-5pm, Monday-Friday
ALERT HighErrorRate
IF rate(http_errors[5m]) > 10
  AND hour() >= 9
  AND hour() < 17
  AND day_of_week() >= 1
  AND day_of_week() <= 5
```

### timestamp()

Trả về timestamp của sample.

**Ví dụ:**

```promql
timestamp(metric)
```

**Use case:** Kiểm tra staleness của data.

```promql
time() - timestamp(metric) > 300
```

**Giải thích:** Metric không được update trong 5 phút.

## Histogram Functions

### histogram_quantile()

Tính quantile từ histogram.

**Cú pháp:**

```promql
histogram_quantile(φ, histogram)
```

Trong đó `φ` là quantile (0-1).

**Ví dụ: P95 latency**

```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

**Giải thích:**
- `0.95` = P95 (95th percentile)
- Tính từ histogram buckets
- Kết quả là latency mà 95% requests nhanh hơn

**Kết quả mẫu:**
```
{instance="server1", method="GET"} 0.25  # P95 = 250ms
{instance="server2", method="GET"} 0.18  # P95 = 180ms
```

**Các quantiles phổ biến:**

```promql
# P50 (median)
histogram_quantile(0.5, rate(http_request_duration_seconds_bucket[5m]))

# P90
histogram_quantile(0.9, rate(http_request_duration_seconds_bucket[5m]))

# P95
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# P99
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

**Best practice:** Luôn dùng `rate()` hoặc `increase()` với histogram buckets.

**Với grouping:**

```promql
histogram_quantile(0.95,
  sum by(le, method) (rate(http_request_duration_seconds_bucket[5m]))
)
```

**Giải thích:** P95 latency cho mỗi HTTP method.

## Label Manipulation

### label_replace()

Thêm hoặc thay đổi label dựa trên regex.

**Cú pháp:**

```promql
label_replace(v, "dst_label", "replacement", "src_label", "regex")
```

**Ví dụ:**

```promql
label_replace(
  http_requests_total,
  "environment",
  "$1",
  "instance",
  ".*-(.*)\.example\.com:.*"
)
```

**Giải thích:**
- Extract environment từ instance name
- `instance="api-prod.example.com:8080"` → `environment="prod"`

**Use case:** Normalize labels từ các sources khác nhau.

### label_join()

Kết hợp nhiều labels thành một.

**Cú pháp:**

```promql
label_join(v, "dst_label", "separator", "src_label1", "src_label2", ...)
```

**Ví dụ:**

```promql
label_join(
  http_requests_total,
  "endpoint",
  ":",
  "method",
  "path"
)
```

**Giải thích:**
- Kết hợp `method` và `path` thành `endpoint`
- `method="GET"`, `path="/api/users"` → `endpoint="GET:/api/users"`

**Use case:** Tạo composite labels cho grouping.

## Ví Dụ Thực Tế

### Ví Dụ 1: Request Rate Per Endpoint

**Mục đích:** Tính request rate cho mỗi endpoint

```promql
sum by(method, path) (rate(http_requests_total[5m]))
```

**Giải thích:**
1. Tính rate cho mỗi time-series
2. Group by method và path
3. Sum các instances

**Kết quả mẫu:**
```
{method="GET", path="/api/users"} 45.2
{method="POST", path="/api/users"} 12.8
{method="GET", path="/api/orders"} 23.5
```

### Ví Dụ 2: Error Rate Percentage

**Mục đích:** Tính phần trăm requests lỗi

```promql
(
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
) * 100
```

**Giải thích:**
1. Tính rate của 5xx errors
2. Tính rate của tất cả requests
3. Chia và nhân 100

**Kết quả mẫu:**
```
{} 2.5  # 2.5% error rate
```

### Ví Dụ 3: P95 Latency By Service

**Mục đích:** Tính P95 latency cho mỗi service

```promql
histogram_quantile(0.95,
  sum by(le, service) (rate(http_request_duration_seconds_bucket[5m]))
)
```

**Giải thích:**
1. Tính rate cho mỗi bucket
2. Group by service (và le cho histogram)
3. Tính P95 quantile

**Kết quả mẫu:**
```
{service="api"} 0.25      # 250ms
{service="auth"} 0.15     # 150ms
{service="database"} 0.45 # 450ms
```

### Ví Dụ 4: Top 5 Endpoints By Traffic

**Mục đích:** Tìm 5 endpoints có traffic cao nhất

```promql
topk(5, sum by(path) (rate(http_requests_total[5m])))
```

**Giải thích:**
1. Tính rate cho mỗi path
2. Sum across all instances
3. Lấy top 5

**Kết quả mẫu:**
```
{path="/api/users"} 45.2
{path="/api/orders"} 38.7
{path="/api/products"} 32.1
{path="/api/search"} 28.5
{path="/api/cart"} 25.3
```

### Ví Dụ 5: Memory Usage Prediction

**Mục đích:** Dự đoán khi nào hết memory

```promql
predict_linear(node_memory_MemAvailable_bytes[1h], 3600)
```

**Giải thích:**
- Dựa trên trend 1 giờ qua
- Dự đoán giá trị sau 3600 seconds (1 giờ nữa)
- Nếu kết quả < 0, sẽ hết memory trong 1 giờ

**Use case:** Proactive alerts trước khi hết resources.

### Ví Dụ 6: Request Rate Change

**Mục đích:** So sánh request rate với 1 giờ trước

```promql
(
  sum(rate(http_requests_total[5m]))
  /
  sum(rate(http_requests_total[5m] offset 1h))
) - 1
```

**Giải thích:**
1. Tính rate hiện tại
2. Tính rate 1 giờ trước
3. Chia và trừ 1 để ra % thay đổi

**Kết quả mẫu:**
```
{} 0.25  # Tăng 25%
{} -0.15 # Giảm 15%
```

### Ví Dụ 7: Uptime Percentage

**Mục đích:** Tính uptime % trong 24 giờ

```promql
avg_over_time(up[24h]) * 100
```

**Giải thích:**
- `up` = 1 khi service up, 0 khi down
- `avg_over_time` tính trung bình
- Nhân 100 để ra phần trăm

**Kết quả mẫu:**
```
{instance="server1"} 99.8  # 99.8% uptime
{instance="server2"} 95.2  # 95.2% uptime
```

## Best Practices

### 1. Chọn Time Range Phù Hợp

**Với rate():**
- Minimum: 4x scrape interval
- Recommended: 5m cho real-time monitoring
- Longer ranges: Smoother but less responsive

**Với increase():**
- Phụ thuộc vào use case
- 1h cho hourly totals
- 24h cho daily totals

### 2. Sử Dụng rate() Thay Vì irate() Cho Alerts

**Tốt:**
```promql
rate(http_requests_total[5m]) > 100
```

**Không tốt:**
```promql
irate(http_requests_total[5m]) > 100
```

**Lý do:** `rate()` ổn định hơn, tránh false positives.

### 3. Luôn Dùng rate() Với Histogram Buckets

**Tốt:**
```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

**Sai:**
```promql
histogram_quantile(0.95, http_request_duration_seconds_bucket)
```

### 4. Group By Trước Khi Tính Quantile

**Tốt:**
```promql
histogram_quantile(0.95,
  sum by(le, service) (rate(http_request_duration_seconds_bucket[5m]))
)
```

**Không tốt:**
```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

**Lý do:** Quantile của aggregated data chính xác hơn.

## Troubleshooting

### Vấn Đề: rate() Trả Về Giá Trị Âm

**Triệu chứng:** `rate()` cho kết quả âm

**Nguyên nhân:** Counter reset không được xử lý đúng

**Giải pháp:** Đảm bảo metric là counter, không phải gauge

### Vấn Đề: histogram_quantile() Trả Về NaN

**Triệu chứng:** Quantile calculation trả về NaN

**Nguyên nhân:**
- Thiếu `le` label
- Không dùng `rate()` hoặc `increase()`
- Buckets không đầy đủ

**Giải pháp:**
```promql
histogram_quantile(0.95,
  sum by(le) (rate(metric_bucket[5m]))
)
```

### Vấn Đề: Aggregation Mất Labels

**Triệu chứng:** Labels quan trọng bị mất sau aggregation

**Giải pháp:** Sử dụng `by` hoặc `without` để giữ labels

```promql
sum by(service, environment) (metric)
```

## Tài Liệu Liên Quan

- [Data Types](./02-data-types.md) - Hiểu input/output types của functions
- [Operators](./03-operators.md) - Kết hợp functions với operators
- [Aggregations](./05-aggregations.md) - Aggregation operators chi tiết
- [Metrics Types](../01-fundamentals/03-metrics-types.md) - Counter, Gauge, Histogram

## Tài Liệu Tham Khảo

- [Official PromQL Functions Documentation](https://prometheus.io/docs/prometheus/latest/querying/functions/)
- [PromQL Quick Reference](../09-reference/01-promql-quick-reference.md)
