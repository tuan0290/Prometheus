# PromQL Basics - Cơ Bản Về PromQL

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Cú Pháp Cơ Bản](#cú-pháp-cơ-bản)
- [Metric Selectors](#metric-selectors)
- [Label Matchers](#label-matchers)
- [Time Ranges](#time-ranges)
- [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Giới Thiệu

PromQL (Prometheus Query Language) là ngôn ngữ truy vấn mạnh mẽ được thiết kế đặc biệt cho dữ liệu time-series. PromQL cho phép bạn chọn lọc, tổng hợp và phân tích metrics theo thời gian thực.

**Đặc điểm chính của PromQL:**
- Ngôn ngữ biểu thức (expression language) chứ không phải SQL
- Hoạt động trên time-series data
- Hỗ trợ tính toán và aggregation theo thời gian
- Tích hợp sẵn với Prometheus và Grafana

## Cú Pháp Cơ Bản

### Instant Vector Selector

Cách đơn giản nhất để truy vấn metric là sử dụng tên metric:

```promql
http_requests_total
```

Query này trả về tất cả time-series có tên `http_requests_total` tại thời điểm hiện tại.

**Kết quả mẫu:**
```
http_requests_total{instance="localhost:9090", job="prometheus", method="GET", status="200"} 1234
http_requests_total{instance="localhost:9090", job="prometheus", method="POST", status="200"} 567
http_requests_total{instance="localhost:9090", job="prometheus", method="GET", status="404"} 89
```

### Range Vector Selector

Để lấy dữ liệu trong một khoảng thời gian, thêm duration vào trong dấu ngoặc vuông:

```promql
http_requests_total[5m]
```

Query này trả về tất cả giá trị của `http_requests_total` trong 5 phút gần nhất.

**Duration units:**
- `s` - seconds (giây)
- `m` - minutes (phút)
- `h` - hours (giờ)
- `d` - days (ngày)
- `w` - weeks (tuần)
- `y` - years (năm)

## Metric Selectors

### Chọn Metric Theo Tên

Cách cơ bản nhất là sử dụng tên metric:

```promql
node_cpu_seconds_total
```

### Chọn Tất Cả Metrics

Sử dụng `{__name__=~".*"}` để chọn tất cả metrics (không khuyến nghị trong production):

```promql
{__name__=~"node.*"}
```

Query này chọn tất cả metrics có tên bắt đầu bằng "node".

## Label Matchers

Label matchers cho phép lọc time-series dựa trên giá trị của labels.

### Equality Matcher (=)

Chọn time-series có label khớp chính xác:

```promql
http_requests_total{method="GET"}
```

**Giải thích:** Chỉ lấy requests với method là GET.

### Inequality Matcher (!=)

Chọn time-series có label không khớp:

```promql
http_requests_total{status!="200"}
```

**Giải thích:** Lấy tất cả requests không có status 200 (lỗi).

### Regular Expression Matcher (=~)

Chọn time-series có label khớp với regex:

```promql
http_requests_total{method=~"GET|POST"}
```

**Giải thích:** Lấy requests với method là GET hoặc POST.

### Negative Regular Expression Matcher (!~)

Chọn time-series có label không khớp với regex:

```promql
http_requests_total{status!~"2.."}
```

**Giải thích:** Lấy requests không có status code 2xx (thành công).

### Kết Hợp Nhiều Matchers

Bạn có thể kết hợp nhiều label matchers:

```promql
http_requests_total{method="GET", status="200", instance="localhost:9090"}
```

**Giải thích:** Lấy GET requests thành công từ instance cụ thể.

## Time Ranges

### Offset Modifier

Sử dụng `offset` để truy vấn dữ liệu từ quá khứ:

```promql
http_requests_total offset 5m
```

**Giải thích:** Lấy giá trị của metric từ 5 phút trước.

**Use case:** So sánh giá trị hiện tại với quá khứ:

```promql
http_requests_total - http_requests_total offset 1h
```

Tính số requests tăng thêm trong 1 giờ qua.

### @ Modifier

Sử dụng `@` để truy vấn tại một timestamp cụ thể:

```promql
http_requests_total @ 1609459200
```

**Giải thích:** Lấy giá trị tại Unix timestamp 1609459200 (2021-01-01 00:00:00 UTC).

**Kết hợp với offset:**

```promql
http_requests_total @ 1609459200 offset 5m
```

Lấy giá trị tại 5 phút trước timestamp đã chỉ định.

## Ví Dụ Thực Tế

### Ví Dụ 1: Monitoring HTTP Errors

**Mục đích:** Tìm tất cả HTTP requests có lỗi (status 5xx)

```promql
http_requests_total{status=~"5.."}
```

**Use case:** Alert khi có quá nhiều lỗi server.

### Ví Dụ 2: Monitoring Specific Service

**Mục đích:** Theo dõi requests của một service cụ thể

```promql
http_requests_total{job="api-server", instance="10.0.1.5:8080"}
```

**Use case:** Debug performance issues của một instance cụ thể.

### Ví Dụ 3: Excluding Test Environments

**Mục đích:** Loại trừ dữ liệu từ môi trường test

```promql
http_requests_total{environment!="test"}
```

**Use case:** Chỉ monitor production traffic.

### Ví Dụ 4: Multiple Services Pattern

**Mục đích:** Monitor nhiều services có tên tương tự

```promql
http_requests_total{job=~"api-.*"}
```

**Use case:** Aggregate metrics từ tất cả API services (api-users, api-orders, api-payments).

### Ví Dụ 5: Time Range Analysis

**Mục đích:** Lấy dữ liệu trong 1 giờ qua để tính rate

```promql
http_requests_total{method="GET"}[1h]
```

**Use case:** Chuẩn bị dữ liệu cho các functions như `rate()` hoặc `increase()`.

## Best Practices

### 1. Sử Dụng Label Matchers Hiệu Quả

**Tốt:**
```promql
http_requests_total{job="api", method="GET"}
```

**Không tốt:**
```promql
{__name__="http_requests_total", job="api", method="GET"}
```

**Lý do:** Cú pháp đầu tiên ngắn gọn và dễ đọc hơn.

### 2. Tránh Queries Quá Rộng

**Tốt:**
```promql
http_requests_total{job="api"}
```

**Không tốt:**
```promql
http_requests_total
```

**Lý do:** Query cụ thể hơn giúp giảm tải cho Prometheus và trả về kết quả nhanh hơn.

### 3. Sử Dụng Regex Cẩn Thận

**Tốt:**
```promql
http_requests_total{status=~"2..|3.."}
```

**Không tốt:**
```promql
http_requests_total{status=~".*"}
```

**Lý do:** Regex quá rộng có thể làm chậm query và trả về quá nhiều dữ liệu không cần thiết.

### 4. Đặt Tên Labels Có Ý Nghĩa

Khi tạo custom metrics, sử dụng label names rõ ràng:

```promql
api_requests_total{endpoint="/users", http_method="GET", status_code="200"}
```

## Troubleshooting

### Vấn Đề: Query Không Trả Về Kết Quả

**Triệu chứng:** Query trả về "No data"

**Nguyên nhân có thể:**
1. Metric name sai chính tả
2. Label matcher không khớp với dữ liệu thực tế
3. Time range không có dữ liệu

**Giải pháp:**
1. Kiểm tra metric name trong Prometheus UI (Status > Targets)
2. Thử query đơn giản hơn: chỉ dùng metric name
3. Kiểm tra labels có sẵn bằng cách query không có matcher

### Vấn Đề: Query Quá Chậm

**Triệu chứng:** Query mất nhiều thời gian để thực thi

**Nguyên nhân:**
- Query quá rộng, trả về quá nhiều time-series
- Time range quá lớn

**Giải pháp:**
1. Thêm label matchers để thu hẹp kết quả
2. Giảm time range
3. Sử dụng recording rules cho queries phức tạp thường xuyên

## Tài Liệu Liên Quan

- [Data Types](./02-data-types.md) - Hiểu về scalar, instant vector, range vector
- [Operators](./03-operators.md) - Sử dụng operators để kết hợp queries
- [Functions](./04-functions.md) - Áp dụng functions lên queries
- [Prometheus Architecture](../01-fundamentals/04-prometheus-architecture.md) - Hiểu cách Prometheus lưu trữ metrics

## Tài Liệu Tham Khảo

- [Official PromQL Documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [PromQL Cheat Sheet](../09-reference/01-promql-quick-reference.md)
