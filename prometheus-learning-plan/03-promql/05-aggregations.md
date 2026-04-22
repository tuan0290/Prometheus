# Aggregations - Tổng Hợp Dữ Liệu Trong PromQL

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Aggregation Operators](#aggregation-operators)
- [Grouping với by](#grouping-với-by)
- [Grouping với without](#grouping-với-without)
- [topk và bottomk](#topk-và-bottomk)
- [quantile](#quantile)
- [count_values](#count_values)
- [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Giới Thiệu

Aggregation operators cho phép bạn tổng hợp nhiều time-series thành ít time-series hơn hoặc thành một giá trị duy nhất. Đây là công cụ quan trọng để:
- Tính tổng metrics từ nhiều instances
- Tìm giá trị trung bình, max, min
- Group metrics theo dimensions
- Phân tích distribution của data

**Cú pháp chung:**

```promql
<aggregation_operator>([parameter,] <instant_vector>) [by|without (<label_list>)]
```

## Aggregation Operators

### sum() - Tổng

Tính tổng các giá trị.

**Ví dụ cơ bản:**

```promql
sum(http_requests_total)
```

**Giải thích:** Tổng tất cả requests từ tất cả instances, methods, status codes.

**Input:**
```
{instance="server1", method="GET", status="200"} 1000
{instance="server1", method="POST", status="200"} 500
{instance="server2", method="GET", status="200"} 800
{instance="server2", method="POST", status="200"} 300
```

**Output:**
```
{} 2600
```

### avg() - Trung Bình

Tính giá trị trung bình.

**Ví dụ:**

```promql
avg(node_cpu_usage)
```

**Giải thích:** CPU usage trung bình của tất cả nodes.

**Use case:** Monitoring average response time, average load.

### min() và max() - Nhỏ Nhất và Lớn Nhất

Tìm giá trị nhỏ nhất hoặc lớn nhất.

**Ví dụ:**

```promql
# CPU cao nhất
max(node_cpu_usage)

# Memory thấp nhất
min(node_memory_available_bytes)
```

**Use case:** Tìm bottleneck, identify outliers.

### count() - Đếm

Đếm số lượng time-series.

**Ví dụ:**

```promql
count(up{job="api"})
```

**Giải thích:** Đếm số instances của job "api".

**Use case:** Monitoring cluster size, counting active services.

### stddev() và stdvar() - Độ Lệch Chuẩn

Tính độ lệch chuẩn và phương sai.

**Ví dụ:**

```promql
# Độ lệch chuẩn
stddev(http_request_duration_seconds)

# Phương sai
stdvar(http_request_duration_seconds)
```

**Use case:** Phát hiện inconsistency, monitoring variability.

## Grouping với by

`by` clause cho phép bạn group time-series theo một hoặc nhiều labels.

### Cú Pháp

```promql
<aggregation> by (<label1>, <label2>, ...) (<instant_vector>)
```

Hoặc:

```promql
<aggregation>(<instant_vector>) by (<label1>, <label2>, ...)
```

### Ví Dụ 1: Group By Single Label

**Query:**

```promql
sum by(method) (http_requests_total)
```

**Input:**
```
{instance="server1", method="GET", status="200"} 1000
{instance="server1", method="GET", status="404"} 50
{instance="server1", method="POST", status="200"} 500
{instance="server2", method="GET", status="200"} 800
{instance="server2", method="POST", status="200"} 300
```

**Output:**
```
{method="GET"} 1850   # 1000 + 50 + 800
{method="POST"} 800   # 500 + 300
```

**Giải thích:**
- Group tất cả time-series theo `method`
- Bỏ qua các labels khác (instance, status)
- Tính tổng cho mỗi group

### Ví Dụ 2: Group By Multiple Labels

**Query:**

```promql
sum by(method, status) (http_requests_total)
```

**Output:**
```
{method="GET", status="200"} 1800
{method="GET", status="404"} 50
{method="POST", status="200"} 800
```

**Giải thích:** Group theo cả `method` VÀ `status`, giữ lại cả hai labels.

### Ví Dụ 3: Average By Instance

**Query:**

```promql
avg by(instance) (rate(http_request_duration_seconds[5m]))
```

**Giải thích:** Average response time cho mỗi instance.

**Use case:** So sánh performance giữa các instances.

### Ví Dụ 4: Max By Datacenter

**Query:**

```promql
max by(datacenter) (node_cpu_usage)
```

**Giải thích:** CPU usage cao nhất trong mỗi datacenter.

**Use case:** Identify hotspots per datacenter.

## Grouping với without

`without` clause giữ lại tất cả labels **ngoại trừ** những labels được chỉ định.

### Cú Pháp

```promql
<aggregation> without (<label1>, <label2>, ...) (<instant_vector>)
```

### Ví Dụ 1: Without Single Label

**Query:**

```promql
sum without(instance) (http_requests_total)
```

**Input:**
```
{instance="server1", method="GET", status="200"} 1000
{instance="server2", method="GET", status="200"} 800
{instance="server1", method="POST", status="200"} 500
{instance="server2", method="POST", status="200"} 300
```

**Output:**
```
{method="GET", status="200"} 1800   # server1 + server2
{method="POST", status="200"} 800   # server1 + server2
```

**Giải thích:**
- Bỏ label `instance`
- Giữ lại tất cả labels khác (method, status)
- Sum các instances lại với nhau

### Ví Dụ 2: Without Multiple Labels

**Query:**

```promql
sum without(instance, pod) (http_requests_total)
```

**Giải thích:** Aggregate across instances và pods, giữ lại các labels khác.

**Use case:** Aggregate metrics từ multiple replicas.

### So Sánh by vs without

**Scenario:** Có metrics với labels: `instance`, `job`, `method`, `status`

**Với by:**

```promql
sum by(method, status) (http_requests_total)
```

**Kết quả:** Chỉ giữ `method` và `status`

**Với without:**

```promql
sum without(instance, job) (http_requests_total)
```

**Kết quả:** Giữ `method` và `status` (bỏ `instance` và `job`)

**Khi nào dùng gì?**

| Tình Huống | Dùng | Lý Do |
|---|---|---|
| Biết chính xác labels cần giữ | `by` | Explicit, rõ ràng |
| Có nhiều labels, chỉ muốn bỏ vài cái | `without` | Ngắn gọn hơn |
| Labels có thể thay đổi | `without` | Linh hoạt hơn |

## topk và bottomk

### topk() - Top K Elements

Trả về k time-series có giá trị lớn nhất.

**Cú pháp:**

```promql
topk(k, <instant_vector>)
```

**Ví dụ:**

```promql
topk(5, sum by(path) (rate(http_requests_total[5m])))
```

**Giải thích:** Top 5 endpoints có traffic cao nhất.

**Kết quả mẫu:**
```
{path="/api/users"} 45.2
{path="/api/orders"} 38.7
{path="/api/products"} 32.1
{path="/api/search"} 28.5
{path="/api/cart"} 25.3
```

**Use cases:**
- Top endpoints by traffic
- Top instances by CPU usage
- Top services by error rate

### bottomk() - Bottom K Elements

Trả về k time-series có giá trị nhỏ nhất.

**Ví dụ:**

```promql
bottomk(3, node_memory_available_bytes)
```

**Giải thích:** 3 instances có ít memory available nhất.

**Use case:** Identify instances cần thêm resources.

### Kết Hợp topk với Aggregation

**Ví dụ: Top 10 services by error rate**

```promql
topk(10,
  sum by(service) (rate(http_requests_total{status=~"5.."}[5m]))
)
```

**Giải thích:**
1. Tính error rate cho mỗi service
2. Sum across all instances
3. Lấy top 10

## quantile

Tính quantile (percentile) của distribution.

**Cú pháp:**

```promql
quantile(φ, <instant_vector>)
```

Trong đó `φ` là quantile (0-1).

**Ví dụ:**

```promql
# P50 (median)
quantile(0.5, http_request_duration_seconds)

# P90
quantile(0.9, http_request_duration_seconds)

# P95
quantile(0.95, http_request_duration_seconds)

# P99
quantile(0.99, http_request_duration_seconds)
```

**Với grouping:**

```promql
quantile by(service) (0.95, http_request_duration_seconds)
```

**Giải thích:** P95 latency cho mỗi service.

**Lưu ý:** `quantile()` hoạt động trên instant vector values, khác với `histogram_quantile()` hoạt động trên histogram buckets.

**So sánh:**

| Function | Input | Use Case |
|---|---|---|
| `quantile()` | Instant vector values | Tính quantile của current values |
| `histogram_quantile()` | Histogram buckets | Tính quantile từ histogram data |

## count_values

Đếm số lượng time-series có cùng giá trị.

**Cú pháp:**

```promql
count_values("output_label", <instant_vector>)
```

**Ví dụ:**

```promql
count_values("version", prometheus_build_info)
```

**Kết quả mẫu:**
```
{version="2.30.0"} 5  # 5 instances chạy version 2.30.0
{version="2.31.0"} 3  # 3 instances chạy version 2.31.0
{version="2.32.0"} 2  # 2 instances chạy version 2.32.0
```

**Use cases:**
- Version distribution trong cluster
- Status code distribution
- Configuration drift detection

**Ví dụ: HTTP status code distribution**

```promql
count_values("status", http_response_status)
```

**Kết quả mẫu:**
```
{status="200"} 850
{status="404"} 45
{status="500"} 12
{status="503"} 3
```

## Ví Dụ Thực Tế

### Ví Dụ 1: Total Request Rate Per Service

**Mục đích:** Tính tổng request rate cho mỗi service

```promql
sum by(service) (rate(http_requests_total[5m]))
```

**Giải thích:**
1. Tính rate cho mỗi instance
2. Group by service
3. Sum tất cả instances của mỗi service

**Kết quả mẫu:**
```
{service="api"} 125.5
{service="auth"} 45.2
{service="database"} 78.3
```

### Ví Dụ 2: Average Response Time By Endpoint

**Mục đích:** Tính average response time cho mỗi endpoint

```promql
avg by(path) (rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m]))
```

**Giải thích:**
1. Tính average duration: sum / count
2. Group by path
3. Average across all instances

**Kết quả mẫu:**
```
{path="/api/users"} 0.15      # 150ms
{path="/api/orders"} 0.25     # 250ms
{path="/api/products"} 0.12   # 120ms
```

### Ví Dụ 3: Error Rate Per Service

**Mục đích:** Tính error rate percentage cho mỗi service

```promql
(
  sum by(service) (rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum by(service) (rate(http_requests_total[5m]))
) * 100
```

**Giải thích:**
1. Sum error requests per service
2. Sum total requests per service
3. Chia và nhân 100

**Kết quả mẫu:**
```
{service="api"} 2.5      # 2.5% error rate
{service="auth"} 0.8     # 0.8% error rate
{service="database"} 5.2 # 5.2% error rate
```

### Ví Dụ 4: Top 5 Instances By CPU Usage

**Mục đích:** Tìm 5 instances có CPU cao nhất

```promql
topk(5,
  100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
)
```

**Giải thích:**
1. Tính CPU usage percentage
2. Average across all CPU cores per instance
3. Lấy top 5

**Kết quả mẫu:**
```
{instance="server1"} 85.5
{instance="server5"} 78.3
{instance="server3"} 72.1
{instance="server8"} 68.9
{instance="server2"} 65.4
```

### Ví Dụ 5: Memory Usage By Datacenter

**Mục đích:** Tính total memory used trong mỗi datacenter

```promql
sum by(datacenter) (
  node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
) / 1024 / 1024 / 1024
```

**Giải thích:**
1. Tính memory used per instance
2. Sum per datacenter
3. Chuyển đổi sang GB

**Kết quả mẫu:**
```
{datacenter="us-east-1"} 512.5  # 512.5 GB
{datacenter="us-west-2"} 384.2  # 384.2 GB
{datacenter="eu-west-1"} 256.8  # 256.8 GB
```

### Ví Dụ 6: Request Distribution By Status Code

**Mục đích:** Phân tích distribution của HTTP status codes

```promql
sum by(status) (rate(http_requests_total[5m]))
```

**Kết quả mẫu:**
```
{status="200"} 125.5  # Success
{status="201"} 15.2   # Created
{status="400"} 5.3    # Bad Request
{status="404"} 8.7    # Not Found
{status="500"} 2.1    # Server Error
```

**Visualization:** Pie chart hoặc bar chart.

### Ví Dụ 7: P95 Latency Across All Services

**Mục đích:** Tính P95 latency tổng thể

```promql
quantile(0.95, rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m]))
```

**Giải thích:**
1. Tính average duration cho mỗi time-series
2. Tính P95 của tất cả values

**Kết quả mẫu:**
```
{} 0.35  # P95 = 350ms
```

### Ví Dụ 8: Instances Below Memory Threshold

**Mục đích:** Đếm số instances có < 10% memory available

```promql
count(
  (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 < 10
)
```

**Giải thích:**
1. Tính memory available percentage
2. Filter < 10%
3. Đếm số instances

**Kết quả mẫu:**
```
{} 3  # 3 instances có low memory
```

### Ví Dụ 9: Average CPU By Environment

**Mục đích:** So sánh CPU usage giữa các environments

```promql
avg by(environment) (
  100 - (avg by(instance, environment) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
)
```

**Giải thích:**
1. Tính CPU usage per instance
2. Average per environment

**Kết quả mẫu:**
```
{environment="production"} 45.5
{environment="staging"} 25.3
{environment="development"} 15.8
```

### Ví Dụ 10: Kubernetes Pod Count By Namespace

**Mục đích:** Đếm số pods trong mỗi namespace

```promql
count by(namespace) (kube_pod_info)
```

**Kết quả mẫu:**
```
{namespace="default"} 15
{namespace="kube-system"} 8
{namespace="monitoring"} 5
{namespace="app"} 25
```

## Best Practices

### 1. Chọn by vs without Phù Hợp

**Khi có ít labels cần giữ:**

```promql
sum by(service, environment) (metric)
```

**Khi có nhiều labels, chỉ muốn bỏ vài cái:**

```promql
sum without(instance, pod) (metric)
```

### 2. Aggregation Trước Khi Tính Toán

**Tốt:**

```promql
sum by(service) (rate(http_requests_total[5m]))
```

**Không tốt:**

```promql
sum(rate(http_requests_total[5m])) by(service)
```

**Lý do:** Cả hai đều đúng, nhưng cú pháp đầu tiên rõ ràng hơn.

### 3. Sử Dụng topk Để Giảm Cardinality

**Thay vì:**

```promql
sum by(path) (rate(http_requests_total[5m]))
```

**Dùng:**

```promql
topk(20, sum by(path) (rate(http_requests_total[5m])))
```

**Lý do:** Giảm số lượng time-series trong kết quả, tốt cho performance.

### 4. Kết Hợp Aggregations

**Ví dụ: Average of maximums**

```promql
avg(max by(instance) (node_cpu_usage))
```

**Giải thích:**
1. Tìm max CPU per instance
2. Tính average của các max values

### 5. Sử Dụng without Cho Flexibility

**Tốt:**

```promql
sum without(instance, pod, container) (metric)
```

**Lý do:** Nếu thêm labels mới, query vẫn hoạt động đúng.

## Troubleshooting

### Vấn Đề: Aggregation Mất Labels Quan Trọng

**Triệu chứng:** Kết quả không có labels cần thiết

**Nguyên nhân:** Không dùng `by` hoặc `without`

**Giải pháp:**

```promql
# Thay vì
sum(metric)

# Dùng
sum by(service, environment) (metric)
```

### Vấn Đề: topk Không Trả Về Đủ Results

**Triệu chứng:** topk trả về ít hơn k elements

**Nguyên nhân:** Không đủ time-series trong input

**Giải pháp:** Kiểm tra input vector có đủ elements không

### Vấn Đề: Aggregation Với Labels Không Khớp

**Triệu chứng:** Kết quả không như mong đợi

**Nguyên nhân:** Labels không consistent giữa các time-series

**Giải pháp:** Normalize labels trước khi aggregate

```promql
sum by(service) (label_replace(metric, "service", "$1", "app", "(.*)"))
```

## Tài Liệu Liên Quan

- [Functions](./04-functions.md) - Các functions khác trong PromQL
- [Operators](./03-operators.md) - Kết hợp aggregations với operators
- [Data Types](./02-data-types.md) - Hiểu input/output types
- [Advanced Queries](./06-advanced-queries.md) - Queries phức tạp với aggregations

## Tài Liệu Tham Khảo

- [Official PromQL Aggregation Documentation](https://prometheus.io/docs/prometheus/latest/querying/operators/#aggregation-operators)
- [PromQL Quick Reference](../09-reference/01-promql-quick-reference.md)
