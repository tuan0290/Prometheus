# Data Types - Các Kiểu Dữ Liệu Trong PromQL

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Scalar](#scalar)
- [Instant Vector](#instant-vector)
- [Range Vector](#range-vector)
- [String](#string)
- [Chuyển Đổi Giữa Các Kiểu](#chuyển-đổi-giữa-các-kiểu)
- [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Giới Thiệu

PromQL có 4 kiểu dữ liệu chính mà bạn cần hiểu để viết queries hiệu quả:

1. **Scalar** - Giá trị số đơn lẻ
2. **Instant Vector** - Tập hợp time-series tại một thời điểm
3. **Range Vector** - Tập hợp time-series trong một khoảng thời gian
4. **String** - Chuỗi ký tự (ít được sử dụng)

Hiểu rõ các kiểu dữ liệu này là chìa khóa để viết PromQL queries chính xác.

## Scalar

### Định Nghĩa

Scalar là một giá trị số thực (floating-point) đơn lẻ, không có labels và không gắn với time-series cụ thể.

### Cú Pháp

```promql
42
3.14
-10.5
```

### Ví Dụ Sử Dụng

**Ví dụ 1: Số literal**

```promql
100
```

**Kết quả:** Giá trị scalar `100`

**Ví dụ 2: Phép tính với scalar**

```promql
5 + 3
```

**Kết quả:** Giá trị scalar `8`

**Ví dụ 3: So sánh với threshold**

```promql
http_requests_total > 1000
```

**Giải thích:** So sánh instant vector với scalar 1000, trả về các time-series có giá trị > 1000.

### Use Cases

- Định nghĩa thresholds cho alerts
- Tính toán với constants
- Chuyển đổi đơn vị (ví dụ: bytes to megabytes)

**Ví dụ chuyển đổi đơn vị:**

```promql
node_memory_MemAvailable_bytes / 1024 / 1024
```

**Giải thích:** Chuyển đổi bytes sang megabytes bằng cách chia cho scalar 1024 hai lần.

## Instant Vector

### Định Nghĩa

Instant Vector là một tập hợp các time-series, mỗi series có một giá trị duy nhất tại một thời điểm cụ thể (thường là thời điểm hiện tại).

### Cú Pháp

```promql
metric_name{label="value"}
```

### Cấu Trúc Dữ Liệu

Mỗi element trong instant vector bao gồm:
- **Metric name** - Tên của metric
- **Labels** - Tập hợp key-value pairs
- **Sample value** - Giá trị tại thời điểm query
- **Timestamp** - Thời điểm của sample

### Ví Dụ

**Query:**

```promql
http_requests_total{method="GET"}
```

**Kết quả mẫu:**

```
http_requests_total{instance="localhost:9090", job="api", method="GET", status="200"} 15234 @1609459200
http_requests_total{instance="localhost:9091", job="api", method="GET", status="200"} 8956 @1609459200
http_requests_total{instance="localhost:9090", job="api", method="GET", status="404"} 127 @1609459200
```

**Giải thích:**
- 3 time-series khác nhau (khác nhau về instance và status)
- Mỗi series có 1 giá trị tại timestamp 1609459200
- Đây là instant vector với 3 elements

### Đặc Điểm

- Mỗi time-series chỉ có **một giá trị duy nhất**
- Tất cả giá trị đều tại **cùng một timestamp**
- Có thể áp dụng **operators và functions**
- Kết quả của hầu hết PromQL queries là instant vectors

### Use Cases

- Hiển thị giá trị hiện tại của metrics
- Tính toán giữa các metrics
- Aggregation (sum, avg, max, min)
- Filtering và grouping

## Range Vector

### Định Nghĩa

Range Vector là một tập hợp các time-series, mỗi series chứa **nhiều giá trị** trong một khoảng thời gian.

### Cú Pháp

```promql
metric_name{label="value"}[duration]
```

**Duration format:**
- `[5m]` - 5 minutes
- `[1h]` - 1 hour
- `[24h]` - 24 hours
- `[7d]` - 7 days

### Ví Dụ

**Query:**

```promql
http_requests_total{method="GET"}[5m]
```

**Kết quả mẫu:**

```
http_requests_total{instance="localhost:9090", job="api", method="GET", status="200"}
  15000 @1609459080
  15100 @1609459095
  15200 @1609459110
  15234 @1609459125

http_requests_total{instance="localhost:9091", job="api", method="GET", status="200"}
  8900 @1609459080
  8920 @1609459095
  8945 @1609459110
  8956 @1609459125
```

**Giải thích:**
- 2 time-series
- Mỗi series có **nhiều giá trị** trong 5 phút
- Mỗi giá trị có timestamp riêng

### Đặc Điểm

- Mỗi time-series có **nhiều samples** (data points)
- Samples nằm trong **time range** được chỉ định
- **Không thể** sử dụng trực tiếp với operators
- **Phải** sử dụng với functions như `rate()`, `increase()`, `avg_over_time()`

### Lỗi Thường Gặp

**Sai:**

```promql
http_requests_total[5m] + 10
```

**Lỗi:** `Error: invalid expression type "range vector" for operator "+"`

**Đúng:**

```promql
rate(http_requests_total[5m]) + 10
```

**Giải thích:** Phải dùng function `rate()` để chuyển range vector thành instant vector trước khi cộng.

### Use Cases

Range vectors được sử dụng với các functions:

**1. rate() - Tính tốc độ thay đổi**

```promql
rate(http_requests_total[5m])
```

**2. increase() - Tính tổng tăng**

```promql
increase(http_requests_total[1h])
```

**3. avg_over_time() - Tính trung bình**

```promql
avg_over_time(cpu_usage[10m])
```

**4. max_over_time() - Tìm giá trị max**

```promql
max_over_time(response_time_seconds[1h])
```

## String

### Định Nghĩa

String là chuỗi ký tự, hiện tại chỉ được sử dụng trong một số contexts hạn chế.

### Cú Pháp

```promql
"hello world"
'single quotes'
`backticks`
```

### Use Cases

Strings chủ yếu được dùng trong:
- Label values trong queries
- Annotations trong Grafana
- Recording rule names

**Ví dụ:**

```promql
http_requests_total{method="GET"}
```

Ở đây `"GET"` là string được dùng làm label value.

## Chuyển Đổi Giữa Các Kiểu

### Range Vector → Instant Vector

Sử dụng functions để chuyển đổi:

```promql
# Range vector
http_requests_total[5m]

# Chuyển thành instant vector bằng rate()
rate(http_requests_total[5m])
```

### Instant Vector → Scalar

Sử dụng aggregation functions:

```promql
# Instant vector
http_requests_total

# Chuyển thành scalar bằng sum()
sum(http_requests_total)
```

**Kết quả:** Một giá trị scalar duy nhất (tổng của tất cả time-series).

### Scalar → Instant Vector

Không thể chuyển trực tiếp, nhưng có thể dùng trong phép tính:

```promql
# Scalar 100 được áp dụng cho mỗi element trong instant vector
http_requests_total + 100
```

### Bảng Tóm Tắt Chuyển Đổi

| Từ | Đến | Cách Thức | Ví Dụ |
|---|---|---|---|
| Range Vector | Instant Vector | Functions (rate, increase, avg_over_time) | `rate(metric[5m])` |
| Instant Vector | Scalar | Aggregation (sum, avg, count) | `sum(metric)` |
| Scalar | Instant Vector | Phép tính với instant vector | `metric + 10` |
| Instant Vector | Range Vector | Không thể | N/A |

## Ví Dụ Thực Tế

### Ví Dụ 1: Monitoring Request Rate

**Mục đích:** Tính số requests per second

```promql
rate(http_requests_total[5m])
```

**Kiểu dữ liệu:**
- Input: Range vector `http_requests_total[5m]`
- Output: Instant vector (requests/second cho mỗi time-series)

**Kết quả mẫu:**
```
{instance="localhost:9090", method="GET"} 12.5
{instance="localhost:9091", method="GET"} 8.3
```

### Ví Dụ 2: Total Requests Across All Instances

**Mục đích:** Tính tổng requests từ tất cả instances

```promql
sum(http_requests_total)
```

**Kiểu dữ liệu:**
- Input: Instant vector `http_requests_total`
- Output: Scalar (một giá trị duy nhất)

**Kết quả mẫu:**
```
45678
```

### Ví Dụ 3: Average CPU Usage Over Time

**Mục đích:** Tính CPU usage trung bình trong 1 giờ

```promql
avg_over_time(node_cpu_usage[1h])
```

**Kiểu dữ liệu:**
- Input: Range vector `node_cpu_usage[1h]`
- Output: Instant vector (giá trị trung bình cho mỗi time-series)

**Kết quả mẫu:**
```
{instance="server1"} 0.45
{instance="server2"} 0.62
```

### Ví Dụ 4: Memory Available in GB

**Mục đích:** Chuyển đổi memory từ bytes sang GB

```promql
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```

**Kiểu dữ liệu:**
- Input: Instant vector `node_memory_MemAvailable_bytes`
- Scalars: `1024` (3 lần)
- Output: Instant vector (giá trị tính bằng GB)

**Kết quả mẫu:**
```
{instance="server1"} 8.5
{instance="server2"} 12.3
```

### Ví Dụ 5: Combining Data Types

**Mục đích:** Tính phần trăm requests lỗi

```promql
(
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
) * 100
```

**Kiểu dữ liệu:**
- Range vectors: `http_requests_total[5m]`
- Instant vectors: `rate(...)` results
- Scalars: `sum(...)` results và `100`
- Output: Scalar (phần trăm lỗi)

**Kết quả mẫu:**
```
2.5
```

**Giải thích:** 2.5% requests có lỗi 5xx.

## Best Practices

### 1. Hiểu Kiểu Dữ Liệu Của Query

Trước khi viết query, xác định:
- Input data type cần gì?
- Output data type mong muốn là gì?
- Cần functions nào để chuyển đổi?

### 2. Sử Dụng Functions Phù Hợp

**Với Range Vectors:**
- `rate()` - cho counters
- `increase()` - cho counters
- `avg_over_time()` - cho gauges
- `max_over_time()`, `min_over_time()` - cho gauges

**Với Instant Vectors:**
- `sum()`, `avg()`, `max()`, `min()` - aggregations
- `+`, `-`, `*`, `/` - arithmetic operations

### 3. Tránh Nhầm Lẫn Range Vector và Instant Vector

**Sai:**
```promql
http_requests_total[5m] > 1000
```

**Đúng:**
```promql
rate(http_requests_total[5m]) > 1000
```

### 4. Chọn Time Range Phù Hợp

- **Quá ngắn** (< 2 scrape intervals): Không đủ data points
- **Quá dài**: Query chậm, kết quả không real-time
- **Khuyến nghị**: 5m cho rate calculations

## Tài Liệu Liên Quan

- [PromQL Basics](./01-promql-basics.md) - Cú pháp cơ bản và selectors
- [Operators](./03-operators.md) - Sử dụng operators với các kiểu dữ liệu
- [Functions](./04-functions.md) - Functions để chuyển đổi và xử lý data types
- [Aggregations](./05-aggregations.md) - Aggregation operations

## Tài Liệu Tham Khảo

- [Official PromQL Data Types Documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/#expression-language-data-types)
- [PromQL Quick Reference](../09-reference/01-promql-quick-reference.md)
