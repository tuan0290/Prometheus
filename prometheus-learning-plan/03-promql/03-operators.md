# Operators - Toán Tử Trong PromQL

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Arithmetic Operators](#arithmetic-operators)
- [Comparison Operators](#comparison-operators)
- [Logical Operators](#logical-operators)
- [Vector Matching](#vector-matching)
- [Operator Precedence](#operator-precedence)
- [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Giới Thiệu

PromQL cung cấp các operators để thực hiện phép tính, so sánh và logic trên metrics. Operators hoạt động trên instant vectors và scalars, cho phép bạn kết hợp và phân tích dữ liệu phức tạp.

**Các nhóm operators:**
- **Arithmetic** - Phép tính số học (+, -, *, /, %, ^)
- **Comparison** - So sánh (==, !=, >, <, >=, <=)
- **Logical** - Logic (and, or, unless)

## Arithmetic Operators

### Danh Sách Operators

| Operator | Mô Tả | Ví Dụ |
|---|---|---|
| `+` | Cộng | `a + b` |
| `-` | Trừ | `a - b` |
| `*` | Nhân | `a * b` |
| `/` | Chia | `a / b` |
| `%` | Modulo (chia lấy dư) | `a % b` |
| `^` | Lũy thừa | `a ^ b` |

### Scalar và Instant Vector

**Scalar với Scalar:**

```promql
5 + 3
```

**Kết quả:** `8`

**Instant Vector với Scalar:**

```promql
http_requests_total + 100
```

**Giải thích:** Cộng 100 vào mỗi giá trị trong instant vector.

**Kết quả mẫu:**
```
{instance="localhost:9090", method="GET"} 1234 → 1334
{instance="localhost:9091", method="GET"} 567 → 667
```

### Instant Vector với Instant Vector

**Cú pháp:**

```promql
vector1 + vector2
```

**Ví dụ:**

```promql
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

**Giải thích:** Tính memory đã sử dụng = Total - Available

**Kết quả mẫu:**
```
{instance="server1"} 8589934592  # 8GB used
{instance="server2"} 4294967296  # 4GB used
```

### Chuyển Đổi Đơn Vị

**Bytes to Megabytes:**

```promql
node_memory_MemAvailable_bytes / 1024 / 1024
```

**Bytes to Gigabytes:**

```promql
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```

**Seconds to Milliseconds:**

```promql
http_request_duration_seconds * 1000
```

### Tính Phần Trăm

**Memory Usage Percentage:**

```promql
(
  (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
  /
  node_memory_MemTotal_bytes
) * 100
```

**Giải thích:**
1. Tính memory used: `Total - Available`
2. Chia cho total: `used / total`
3. Nhân 100 để ra phần trăm

**Kết quả mẫu:**
```
{instance="server1"} 65.5  # 65.5% memory used
{instance="server2"} 42.3  # 42.3% memory used
```

## Comparison Operators

### Danh Sách Operators

| Operator | Mô Tả | Ví Dụ |
|---|---|---|
| `==` | Bằng | `a == b` |
| `!=` | Không bằng | `a != b` |
| `>` | Lớn hơn | `a > b` |
| `<` | Nhỏ hơn | `a < b` |
| `>=` | Lớn hơn hoặc bằng | `a >= b` |
| `<=` | Nhỏ hơn hoặc bằng | `a <= b` |

### Filtering Mode (Mặc Định)

Comparison operators mặc định hoạt động ở **filtering mode** - chỉ giữ lại các time-series thỏa mãn điều kiện.

**Ví dụ:**

```promql
http_requests_total > 1000
```

**Giải thích:** Chỉ trả về các time-series có giá trị > 1000.

**Input:**
```
{instance="server1", method="GET"} 1500
{instance="server2", method="GET"} 800
{instance="server3", method="GET"} 2000
```

**Output:**
```
{instance="server1", method="GET"} 1500
{instance="server3", method="GET"} 2000
```

### Boolean Mode

Sử dụng `bool` modifier để trả về 0 (false) hoặc 1 (true) thay vì filtering.

**Cú pháp:**

```promql
expression1 operator bool expression2
```

**Ví dụ:**

```promql
http_requests_total > bool 1000
```

**Output:**
```
{instance="server1", method="GET"} 1  # 1500 > 1000 = true
{instance="server2", method="GET"} 0  # 800 > 1000 = false
{instance="server3", method="GET"} 1  # 2000 > 1000 = true
```

**Use case:** Tạo binary metrics cho alerts hoặc dashboards.

### So Sánh Giữa Vectors

**Ví dụ: So sánh CPU usage giữa các cores**

```promql
rate(node_cpu_seconds_total{mode="user"}[5m]) > rate(node_cpu_seconds_total{mode="system"}[5m])
```

**Giải thích:** Tìm các cores có user CPU time > system CPU time.

## Logical Operators

### and - Giao (Intersection)

Trả về các time-series có mặt trong **cả hai** vectors.

**Cú pháp:**

```promql
vector1 and vector2
```

**Ví dụ:**

```promql
up{job="api"} and rate(http_requests_total[5m]) > 10
```

**Giải thích:** Tìm các instances đang up VÀ có request rate > 10/s.

**Use case:** Kết hợp nhiều điều kiện trong alerts.

### or - Hợp (Union)

Trả về các time-series có mặt trong **ít nhất một** trong hai vectors.

**Cú pháp:**

```promql
vector1 or vector2
```

**Ví dụ:**

```promql
up{job="api"} == 0 or rate(http_requests_total[5m]) < 1
```

**Giải thích:** Tìm các instances down HOẶC có request rate quá thấp.

**Use case:** Alert khi có bất kỳ vấn đề nào xảy ra.

### unless - Trừ (Complement)

Trả về các time-series trong vector1 **không có** trong vector2.

**Cú pháp:**

```promql
vector1 unless vector2
```

**Ví dụ:**

```promql
up{job="api"} unless on(instance) rate(http_requests_total[5m]) > 0
```

**Giải thích:** Tìm các instances đang up NHƯNG không có traffic.

**Use case:** Phát hiện services idle hoặc misconfigured.

### Ví Dụ Kết Hợp Logical Operators

**Mục đích:** Alert khi service down hoặc có quá nhiều errors

```promql
(up{job="api"} == 0)
or
(rate(http_requests_total{status=~"5.."}[5m]) > 10)
```

**Giải thích:**
- Điều kiện 1: Service down (`up == 0`)
- Điều kiện 2: Error rate > 10/s
- Alert nếu **bất kỳ** điều kiện nào đúng

## Vector Matching

### One-to-One Matching

Mặc định, operators matching các time-series dựa trên **tất cả labels**.

**Ví dụ:**

```promql
method_code:http_errors:rate5m{code="500"} / method:http_requests:rate5m
```

**Matching:** Chỉ kết hợp các time-series có **cùng labels** (method, instance, job, etc.)

### Ignoring Labels

Sử dụng `ignoring` để bỏ qua một số labels khi matching.

**Cú pháp:**

```promql
vector1 operator ignoring(label1, label2) vector2
```

**Ví dụ:**

```promql
method_code:http_errors:rate5m / ignoring(code) method:http_requests:rate5m
```

**Giải thích:** Match các time-series bỏ qua label `code`.

### On Labels

Sử dụng `on` để chỉ matching dựa trên một số labels cụ thể.

**Cú pháp:**

```promql
vector1 operator on(label1, label2) vector2
```

**Ví dụ:**

```promql
method_code:http_errors:rate5m / on(method) method:http_requests:rate5m
```

**Giải thích:** Chỉ match các time-series có cùng label `method`.

### Many-to-One / One-to-Many Matching

Sử dụng `group_left` hoặc `group_right` cho matching không đối xứng.

**Cú pháp:**

```promql
vector1 operator on(labels) group_left(labels) vector2
```

**Ví dụ:**

```promql
method_code:http_errors:rate5m
  / on(instance) group_left
method:http_requests:rate5m
```

**Giải thích:**
- Left side (errors): Nhiều time-series per instance (khác nhau về code)
- Right side (requests): Một time-series per instance
- `group_left`: Giữ labels từ left side

## Operator Precedence

Thứ tự ưu tiên từ cao đến thấp:

1. `^` (lũy thừa)
2. `*`, `/`, `%` (nhân, chia, modulo)
3. `+`, `-` (cộng, trừ)
4. `==`, `!=`, `>`, `<`, `>=`, `<=` (so sánh)
5. `and`, `unless` (logical and)
6. `or` (logical or)

**Ví dụ:**

```promql
2 + 3 * 4
```

**Kết quả:** `14` (không phải `20`)

**Giải thích:** `*` có precedence cao hơn `+`, nên tính `3 * 4 = 12` trước, sau đó `2 + 12 = 14`.

### Sử Dụng Dấu Ngoặc

Để thay đổi thứ tự tính toán:

```promql
(2 + 3) * 4
```

**Kết quả:** `20`

**Best practice:** Luôn dùng dấu ngoặc để làm rõ ý định, tránh nhầm lẫn.

## Ví Dụ Thực Tế

### Ví Dụ 1: CPU Usage Percentage

**Mục đích:** Tính phần trăm CPU usage

```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Giải thích:**
1. Tính idle CPU rate
2. Nhân 100 để ra phần trăm
3. Trừ từ 100 để ra usage percentage

**Kết quả mẫu:**
```
{instance="server1"} 35.5  # 35.5% CPU used
{instance="server2"} 62.3  # 62.3% CPU used
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
3. Chia để ra tỷ lệ
4. Nhân 100 để ra phần trăm

**Kết quả mẫu:**
```
2.5  # 2.5% error rate
```

### Ví Dụ 3: Disk Space Alert

**Mục đích:** Alert khi disk space < 10%

```promql
(
  node_filesystem_avail_bytes{fstype!="tmpfs"}
  /
  node_filesystem_size_bytes{fstype!="tmpfs"}
) * 100 < 10
```

**Giải thích:**
1. Tính available / total
2. Nhân 100 để ra phần trăm
3. Filter các filesystems có < 10% free

**Kết quả mẫu:**
```
{instance="server1", mountpoint="/"} 8.5  # 8.5% free - ALERT!
```

### Ví Dụ 4: Request Rate Comparison

**Mục đích:** So sánh request rate hiện tại với 1 giờ trước

```promql
rate(http_requests_total[5m])
/
rate(http_requests_total[5m] offset 1h)
```

**Giải thích:**
1. Tính rate hiện tại
2. Tính rate 1 giờ trước
3. Chia để ra tỷ lệ thay đổi

**Kết quả mẫu:**
```
{instance="server1"} 1.5  # Traffic tăng 50%
{instance="server2"} 0.8  # Traffic giảm 20%
```

### Ví Dụ 5: High CPU and High Memory

**Mục đích:** Tìm instances có cả CPU và memory cao

```promql
(
  100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
)
and
(
  (
    (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
    /
    node_memory_MemTotal_bytes
  ) * 100 > 80
)
```

**Giải thích:**
1. Tìm instances có CPU > 80%
2. Tìm instances có memory > 80%
3. Dùng `and` để lấy giao của hai tập hợp

**Use case:** Phát hiện instances bị overload nghiêm trọng.

### Ví Dụ 6: Service Health Score

**Mục đích:** Tính health score dựa trên nhiều metrics

```promql
(
  (up * 100)
  +
  ((1 - (rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]))) * 100)
  +
  ((1 - (rate(node_cpu_seconds_total{mode="idle"}[5m]))) * 100 < 80) * 100
) / 3
```

**Giải thích:**
1. Up status: 100 nếu up, 0 nếu down
2. Success rate: 100 - error percentage
3. CPU health: 100 nếu CPU < 80%, 0 nếu >= 80%
4. Trung bình 3 metrics

**Kết quả mẫu:**
```
{instance="server1"} 100  # Perfect health
{instance="server2"} 66.7  # Some issues
```

## Best Practices

### 1. Sử Dụng Dấu Ngoặc

**Tốt:**
```promql
(a + b) * c
```

**Không rõ ràng:**
```promql
a + b * c
```

### 2. Xử Lý Division by Zero

**Vấn đề:**
```promql
a / b  # Lỗi nếu b = 0
```

**Giải pháp:**
```promql
a / (b > 0)  # Chỉ chia khi b > 0
```

Hoặc:

```promql
a / (b + 1)  # Thêm 1 để tránh chia cho 0
```

### 3. Chọn Vector Matching Phù Hợp

- Dùng `on()` khi chỉ cần match một vài labels
- Dùng `ignoring()` khi cần match hầu hết labels
- Dùng `group_left/group_right` cho many-to-one matching

### 4. Tối Ưu Performance

**Chậm:**
```promql
(metric1 + metric2) * (metric3 + metric4)
```

**Nhanh hơn:**
```promql
metric1 * metric3 + metric1 * metric4 + metric2 * metric3 + metric2 * metric4
```

**Lý do:** Giảm số lượng intermediate vectors.

## Troubleshooting

### Vấn Đề: "Many-to-many matching not allowed"

**Triệu chứng:** Lỗi khi thực hiện phép tính giữa vectors

**Nguyên nhân:** Cả hai vectors đều có nhiều time-series với cùng labels

**Giải pháp:** Sử dụng `on()` hoặc `ignoring()` với `group_left/group_right`

```promql
vector1 / on(instance) group_left vector2
```

### Vấn Đề: Kết Quả Không Như Mong Đợi

**Triệu chứng:** Phép tính trả về empty result

**Nguyên nhân:** Labels không match giữa hai vectors

**Giải pháp:**
1. Kiểm tra labels của cả hai vectors
2. Sử dụng `ignoring()` hoặc `on()` để điều chỉnh matching

### Vấn Đề: Division by Zero

**Triệu chứng:** Kết quả là `+Inf` hoặc `NaN`

**Nguyên nhân:** Chia cho 0 hoặc 0/0

**Giải pháp:** Filter trước khi chia

```promql
numerator / (denominator > 0)
```

## Tài Liệu Liên Quan

- [Data Types](./02-data-types.md) - Hiểu các kiểu dữ liệu để sử dụng operators đúng
- [Functions](./04-functions.md) - Kết hợp operators với functions
- [Aggregations](./05-aggregations.md) - Aggregation operators
- [PromQL Basics](./01-promql-basics.md) - Cú pháp cơ bản

## Tài Liệu Tham Khảo

- [Official PromQL Operators Documentation](https://prometheus.io/docs/prometheus/latest/querying/operators/)
- [PromQL Quick Reference](../09-reference/01-promql-quick-reference.md)
