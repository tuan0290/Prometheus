# Monitoring và Observability

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Monitoring là gì?](#monitoring-là-gì)
- [Observability là gì?](#observability-là-gì)
- [Sự Khác Biệt Giữa Monitoring và Observability](#sự-khác-biệt-giữa-monitoring-và-observability)
- [Ba Trụ Cột của Observability](#ba-trụ-cột-của-observability)
- [Tại Sao Cần Monitoring và Observability?](#tại-sao-cần-monitoring-và-observability)
- [Monitoring trong Thực Tế](#monitoring-trong-thực-tế)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Giới Thiệu

Trong thế giới phần mềm hiện đại, hệ thống ngày càng phức tạp với nhiều dịch vụ phân tán, microservices, và infrastructure động. Để đảm bảo hệ thống hoạt động tốt và phát hiện vấn đề sớm, chúng ta cần **monitoring** và **observability**.

## Monitoring là gì?

**Monitoring** (giám sát) là quá trình thu thập, tổng hợp và phân tích dữ liệu về hệ thống để theo dõi trạng thái và hiệu suất của nó.

### Ví Dụ Thực Tế

Hãy tưởng tượng monitoring như việc theo dõi bảng điều khiển của một chiếc xe hơi:

```
┌─────────────────────────────────────┐
│     BẢNG ĐIỀU KHIỂN XE HƠI          │
├─────────────────────────────────────┤
│  Tốc độ: 80 km/h                    │
│  Nhiên liệu: ████████░░ 80%         │
│  Nhiệt độ động cơ: 90°C             │
│  Đèn cảnh báo: ✓ Tất cả bình thường │
└─────────────────────────────────────┘
```

Tương tự, monitoring hệ thống theo dõi:
- **CPU usage**: Mức sử dụng CPU (giống tốc độ xe)
- **Memory usage**: Bộ nhớ còn trống (giống nhiên liệu)
- **Disk I/O**: Hoạt động đọc/ghi đĩa
- **Network traffic**: Lưu lượng mạng
- **Application errors**: Lỗi ứng dụng (giống đèn cảnh báo)

### Đặc Điểm của Monitoring

- **Proactive**: Phát hiện vấn đề trước khi người dùng gặp phải
- **Continuous**: Chạy liên tục 24/7
- **Automated**: Tự động thu thập và cảnh báo
- **Historical**: Lưu trữ dữ liệu lịch sử để phân tích xu hướng

## Observability là gì?

**Observability** (khả năng quan sát) là khả năng hiểu được trạng thái bên trong của hệ thống dựa trên dữ liệu đầu ra mà nó tạo ra.

### Ví Dụ Thực Tế

Nếu monitoring là bảng điều khiển xe, thì observability là khả năng mở nắp ca-pô và kiểm tra chi tiết động cơ khi có vấn đề:

```
Monitoring: "Đèn cảnh báo động cơ sáng!"
            ↓
Observability: "Hãy xem chi tiết:
                - Bugi số 3 bị hỏng
                - Áp suất dầu thấp ở xi-lanh 2
                - Nhiệt độ tăng đột biến lúc 14:23"
```

### Đặc Điểm của Observability

- **Exploratory**: Cho phép khám phá và điều tra vấn đề
- **Context-rich**: Cung cấp ngữ cảnh chi tiết
- **Flexible**: Có thể trả lời các câu hỏi chưa biết trước
- **Debugging-focused**: Tập trung vào việc tìm nguyên nhân gốc rễ

## Sự Khác Biệt Giữa Monitoring và Observability

| Khía Cạnh | Monitoring | Observability |
|-----------|-----------|---------------|
| **Mục đích** | Phát hiện vấn đề đã biết | Điều tra vấn đề chưa biết |
| **Câu hỏi** | "Có vấn đề không?" | "Tại sao có vấn đề?" |
| **Phương pháp** | Theo dõi metrics định trước | Khám phá dữ liệu linh hoạt |
| **Dữ liệu** | Metrics tổng hợp | Metrics + Logs + Traces |
| **Cảnh báo** | Dựa trên ngưỡng | Dựa trên patterns và anomalies |

### Ví Dụ So Sánh

**Monitoring**:
```
Alert: CPU usage > 80% on server-01
```

**Observability**:
```
Investigation:
1. CPU spike started at 14:23:15
2. Caused by process "api-worker" (PID 1234)
3. Triggered by API endpoint /users/search
4. Request from IP 192.168.1.100
5. Query took 45 seconds (normal: 0.5s)
6. Database connection pool exhausted
```

## Ba Trụ Cột của Observability

Observability dựa trên ba loại dữ liệu chính:

### 1. Metrics (Chỉ Số)

Dữ liệu số đo lường theo thời gian.

```
┌─────────────────────────────────────┐
│  CPU Usage Over Time                │
│                                     │
│  100% ┤                             │
│   80% ┤     ╭─╮                     │
│   60% ┤   ╭─╯ ╰─╮                   │
│   40% ┤ ╭─╯     ╰─╮                 │
│   20% ┤─╯         ╰─────────        │
│    0% └─────────────────────────    │
│       10:00  11:00  12:00  13:00    │
└─────────────────────────────────────┘
```

**Ví dụ**: CPU usage, memory usage, request count, response time

**Ưu điểm**: Hiệu quả, dễ tổng hợp, phù hợp cho cảnh báo

### 2. Logs (Nhật Ký)

Bản ghi sự kiện rời rạc với thông tin chi tiết.

```
[2025-01-15 14:23:15] INFO  Request received: GET /api/users
[2025-01-15 14:23:15] DEBUG Database query: SELECT * FROM users WHERE id=123
[2025-01-15 14:23:16] ERROR Database timeout after 30s
[2025-01-15 14:23:16] WARN  Retrying request (attempt 2/3)
```

**Ưu điểm**: Chi tiết, có ngữ cảnh, tốt cho debugging

### 3. Traces (Dấu Vết)

Theo dõi một request qua nhiều services.

```
┌──────────────────────────────────────────────────┐
│  Request Trace: GET /api/orders/123              │
├──────────────────────────────────────────────────┤
│  API Gateway        [████░░░░░░] 45ms            │
│    ↓                                             │
│  Auth Service       [██░░░░░░░░] 20ms            │
│    ↓                                             │
│  Order Service      [████████░░] 80ms            │
│    ↓                                             │
│  Database           [██████████] 100ms ← SLOW!   │
│                                                  │
│  Total: 245ms                                    │
└──────────────────────────────────────────────────┘
```

**Ưu điểm**: Hiển thị luồng request, xác định bottleneck

## Tại Sao Cần Monitoring và Observability?

### 1. Phát Hiện Vấn Đề Sớm

Thay vì đợi người dùng báo lỗi, bạn biết trước khi họ gặp vấn đề.

### 2. Giảm Thời Gian Khắc Phục (MTTR)

**MTTR** (Mean Time To Resolution) là thời gian trung bình để khắc phục sự cố.

```
Không có Observability:
  Phát hiện → Điều tra → Tìm nguyên nhân → Sửa
  [30 phút]  [2 giờ]     [3 giờ]          [1 giờ]
  Total: 6.5 giờ

Có Observability:
  Phát hiện → Điều tra → Tìm nguyên nhân → Sửa
  [5 phút]   [15 phút]   [30 phút]        [1 giờ]
  Total: 1.8 giờ
```

### 3. Hiểu Hành Vi Hệ Thống

Biết hệ thống hoạt động như thế nào trong điều kiện bình thường và bất thường.

### 4. Capacity Planning

Dự đoán khi nào cần mở rộng hệ thống dựa trên xu hướng.

### 5. Cải Thiện Hiệu Suất

Xác định và tối ưu hóa các phần chậm của hệ thống.

## Monitoring trong Thực Tế

### Ví Dụ: Web Application

Một ứng dụng web cần monitor:

```
┌─────────────────────────────────────────────┐
│         WEB APPLICATION STACK               │
├─────────────────────────────────────────────┤
│  Load Balancer                              │
│    Metrics: requests/sec, response time     │
│         ↓                                   │
│  Web Servers (3 instances)                  │
│    Metrics: CPU, memory, active connections │
│         ↓                                   │
│  Application                                │
│    Metrics: errors, latency, throughput     │
│         ↓                                   │
│  Database                                   │
│    Metrics: queries/sec, connection pool    │
│         ↓                                   │
│  Disk Storage                               │
│    Metrics: disk usage, I/O operations      │
└─────────────────────────────────────────────┘
```

### Các Metrics Quan Trọng

**Golden Signals** (4 chỉ số vàng):

1. **Latency**: Thời gian phản hồi
   - Ví dụ: API response time = 250ms

2. **Traffic**: Lưu lượng truy cập
   - Ví dụ: 1000 requests/second

3. **Errors**: Tỷ lệ lỗi
   - Ví dụ: 0.5% requests failed

4. **Saturation**: Mức độ bão hòa tài nguyên
   - Ví dụ: CPU at 75% capacity

### Monitoring Workflow

```
┌──────────────┐
│  Collect     │  Thu thập metrics từ hệ thống
│  Metrics     │
└──────┬───────┘
       ↓
┌──────────────┐
│  Store       │  Lưu trữ trong time-series database
│  Data        │
└──────┬───────┘
       ↓
┌──────────────┐
│  Analyze     │  Phân tích và tạo cảnh báo
│  & Alert     │
└──────┬───────┘
       ↓
┌──────────────┐
│  Visualize   │  Hiển thị trên dashboard
│  & Report    │
└──────────────┘
```

## Tài Liệu Liên Quan

- [Time-Series Databases](./02-time-series-databases.md) - Tìm hiểu về cơ sở dữ liệu lưu trữ metrics
- [Metrics Types](./03-metrics-types.md) - Các loại metrics trong Prometheus
- [Prometheus Architecture](./04-prometheus-architecture.md) - Kiến trúc của Prometheus

## Tài Liệu Tham Khảo

- [Google SRE Book - Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Prometheus Documentation - Overview](https://prometheus.io/docs/introduction/overview/)
- [CNCF Observability Whitepaper](https://www.cncf.io/)
