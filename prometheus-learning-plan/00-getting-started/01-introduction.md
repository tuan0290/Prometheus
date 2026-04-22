# Giới Thiệu về Prometheus

## Mục Lục

- [Prometheus là gì?](#prometheus-là-gì)
- [Tại sao cần Monitoring?](#tại-sao-cần-monitoring)
- [Tại sao nên học Prometheus?](#tại-sao-nên-học-prometheus)
- [Prometheus trong Hệ Sinh Thái Cloud Native](#prometheus-trong-hệ-sinh-thái-cloud-native)
- [Các Use Cases Phổ Biến](#các-use-cases-phổ-biến)
- [Điều gì làm Prometheus khác biệt?](#điều-gì-làm-prometheus-khác-biệt)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Prometheus là gì?

**Prometheus** là một hệ thống monitoring và alerting mã nguồn mở được thiết kế đặc biệt cho môi trường cloud native và microservices. Được phát triển ban đầu tại SoundCloud vào năm 2012, Prometheus hiện là dự án chính thức của Cloud Native Computing Foundation (CNCF) và đã trở thành tiêu chuẩn de facto cho monitoring trong Kubernetes và các hệ thống phân tán.

Prometheus thu thập và lưu trữ metrics dưới dạng time-series data - tức là các giá trị số được ghi lại theo thời gian, kèm theo các labels (nhãn) để phân loại và truy vấn dữ liệu một cách linh hoạt.

### Ví dụ Thực Tế

Hãy tưởng tượng Prometheus như một **bác sĩ theo dõi sức khỏe** của hệ thống máy tính:

- **Metrics** giống như các chỉ số sinh tồn: nhịp tim (CPU usage), huyết áp (memory usage), nhiệt độ cơ thể (disk I/O)
- **Time-series data** giống như bảng theo dõi sức khỏe theo thời gian: đo mỗi 15 giây để thấy xu hướng
- **Alerts** giống như chuông báo động: khi nhịp tim quá nhanh (CPU > 90%), bác sĩ được thông báo ngay
- **Labels** giống như thông tin bệnh nhân: tên, tuổi, phòng ban - giúp phân loại và tìm kiếm nhanh

## Tại sao cần Monitoring?

Trong thế giới phần mềm hiện đại, ứng dụng không chỉ chạy trên một máy chủ đơn lẻ mà thường được triển khai trên hàng chục, hàng trăm, thậm chí hàng nghìn containers và servers. Monitoring giúp bạn:

### 1. Phát Hiện Vấn Đề Sớm

Thay vì đợi người dùng phàn nàn "website chậm", monitoring cho phép bạn phát hiện vấn đề trước khi nó ảnh hưởng đến người dùng.

```
Không có monitoring:
User: "Website bị lỗi!" → Dev: "Để tôi kiểm tra..." → 30 phút sau mới tìm ra nguyên nhân

Có monitoring:
Alert: "API response time > 2s" → Dev: "Đã biết, đang xử lý" → 5 phút sau đã fix
```

### 2. Hiểu Hành Vi Hệ Thống

Monitoring giúp bạn trả lời các câu hỏi quan trọng:
- Có bao nhiêu requests mỗi giây?
- API nào được gọi nhiều nhất?
- Database query nào chậm nhất?
- Hệ thống có đủ tài nguyên không?

### 3. Đưa Ra Quyết Định Dựa Trên Dữ Liệu

Thay vì đoán mò "có lẽ cần thêm server", monitoring cung cấp dữ liệu cụ thể:
- CPU usage trung bình 85% trong giờ cao điểm → Cần scale up
- 95% requests hoàn thành dưới 100ms → Performance tốt
- Error rate tăng 300% sau deploy → Cần rollback ngay

### 4. Tuân Thủ SLA và SLO

Nếu bạn cam kết 99.9% uptime với khách hàng, monitoring là cách duy nhất để chứng minh bạn đang đáp ứng cam kết đó.

## Tại sao nên học Prometheus?

### 1. Tiêu Chuẩn Ngành

Prometheus đã trở thành tiêu chuẩn monitoring cho:
- **Kubernetes**: Tích hợp sẵn, hầu hết Kubernetes clusters đều dùng Prometheus
- **Cloud Native**: Được CNCF chứng nhận là graduated project (cùng cấp với Kubernetes)
- **Microservices**: Thiết kế đặc biệt cho kiến trúc phân tán

### 2. Nhu Cầu Thị Trường Cao

Kỹ năng Prometheus được yêu cầu trong nhiều vị trí:
- DevOps Engineer
- Site Reliability Engineer (SRE)
- Platform Engineer
- Cloud Engineer
- Backend Developer (trong các công ty hiện đại)

### 3. Hệ Sinh Thái Phong Phú

Prometheus không đứng một mình:
- **Grafana**: Visualization và dashboards đẹp mắt
- **Alertmanager**: Quản lý alerts thông minh
- **Exporters**: Hàng trăm exporters sẵn có cho mọi công nghệ
- **Client Libraries**: Hỗ trợ mọi ngôn ngữ lập trình phổ biến

### 4. Mã Nguồn Mở và Miễn Phí

- Không có licensing costs
- Community lớn và active
- Tài liệu phong phú
- Có thể tùy chỉnh theo nhu cầu

### 5. Kiến Thức Nền Tảng cho Observability

Học Prometheus giúp bạn hiểu:
- Time-series databases
- Metrics-based monitoring
- Pull vs Push models
- Service discovery
- Distributed systems monitoring

Những kiến thức này áp dụng được cho nhiều công cụ khác trong lĩnh vực observability.

## Prometheus trong Hệ Sinh Thái Cloud Native

Prometheus là một phần quan trọng trong bộ công cụ cloud native:

```
┌─────────────────────────────────────────────────────────┐
│              Cloud Native Observability Stack            │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐         │
│  │  Metrics │    │   Logs   │    │  Traces  │         │
│  │          │    │          │    │          │         │
│  │Prometheus│    │   Loki   │    │  Jaeger  │         │
│  │          │    │  ELK     │    │  Tempo   │         │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘         │
│       │               │               │                │
│       └───────────────┼───────────────┘                │
│                       │                                │
│                  ┌────▼─────┐                          │
│                  │ Grafana  │                          │
│                  │(Unified  │                          │
│                  │Dashboard)│                          │
│                  └──────────┘                          │
└─────────────────────────────────────────────────────────┘
```

**Vai trò của Prometheus**:
- Thu thập metrics từ applications, containers, infrastructure
- Lưu trữ time-series data hiệu quả
- Cung cấp PromQL để query và analyze
- Trigger alerts khi có vấn đề
- Tích hợp với Grafana để visualization

## Các Use Cases Phổ Biến

### 1. Infrastructure Monitoring

Theo dõi servers, containers, và network:
- CPU, memory, disk usage
- Network traffic và latency
- Container health và resource consumption

**Ví dụ**: Một công ty có 100 servers cần biết server nào đang quá tải để scale hoặc redistribute workload.

### 2. Application Performance Monitoring (APM)

Theo dõi performance của ứng dụng:
- Request rate và response time
- Error rates và success rates
- Database query performance
- API endpoint latency

**Ví dụ**: E-commerce website cần biết API "add to cart" có response time bao nhiêu, có bao nhiêu % requests bị lỗi.

### 3. Business Metrics

Theo dõi metrics liên quan đến business:
- Số lượng đơn hàng mới
- Revenue per minute
- Active users
- Conversion rates

**Ví dụ**: Startup cần dashboard real-time hiển thị số user đăng ký mỗi giờ để đánh giá hiệu quả marketing campaign.

### 4. Kubernetes Monitoring

Theo dõi Kubernetes clusters:
- Pod status và restarts
- Node resources
- Deployment health
- Service discovery

**Ví dụ**: DevOps team cần biết có bao nhiêu pods đang running, có pod nào bị crash và restart liên tục không.

### 5. Alerting và On-Call

Tự động thông báo khi có vấn đề:
- CPU > 90% trong 5 phút → Alert
- Error rate > 5% → Page on-call engineer
- Disk space < 10% → Warning notification

**Ví dụ**: SRE team được alert qua Slack/PagerDuty khi production có vấn đề, ngay cả lúc 3 giờ sáng.

## Điều gì làm Prometheus khác biệt?

### 1. Pull-Based Model

Khác với nhiều hệ thống monitoring khác (push model), Prometheus chủ động "kéo" (pull) metrics từ targets:

```
Push Model (traditional):
Application → Push metrics → Monitoring System

Pull Model (Prometheus):
Prometheus → Pull metrics ← Application (exposes /metrics endpoint)
```

**Ưu điểm**:
- Prometheus kiểm soát tần suất scrape
- Dễ phát hiện targets bị down (không nhận được metrics)
- Không cần configure application để biết monitoring server ở đâu

### 2. Powerful Query Language (PromQL)

PromQL cho phép query và aggregate metrics một cách linh hoạt:

```promql
# Tính request rate trong 5 phút
rate(http_requests_total[5m])

# Tìm top 5 endpoints chậm nhất
topk(5, http_request_duration_seconds)

# Alert khi error rate > 5%
rate(http_requests_total{status="500"}[5m]) / rate(http_requests_total[5m]) > 0.05
```

### 3. Multi-Dimensional Data Model

Metrics có thể có nhiều labels để phân loại:

```promql
http_requests_total{
  method="GET",
  endpoint="/api/users",
  status="200",
  instance="server-1"
}
```

Điều này cho phép query linh hoạt:
- Tất cả requests từ endpoint `/api/users`
- Tất cả GET requests
- Tất cả 500 errors từ server-1

### 4. Service Discovery

Prometheus tự động phát hiện targets cần monitor:
- Kubernetes pods và services
- Consul services
- AWS EC2 instances
- File-based configuration

Không cần manually add/remove targets khi scale up/down.

### 5. No External Dependencies

Prometheus là single binary, không cần:
- External database
- Message queue
- Complex setup

Chỉ cần download và chạy là có thể bắt đầu monitoring.

## Tài Liệu Liên Quan

- [Learning Path Overview](./02-learning-path-overview.md) - Xem lộ trình học tập chi tiết
- [Prerequisites](./03-prerequisites.md) - Chuẩn bị kiến thức và công cụ cần thiết
- [Monitoring and Observability](../01-fundamentals/01-monitoring-observability.md) - Tìm hiểu sâu hơn về monitoring concepts
- [Prometheus Architecture](../01-fundamentals/04-prometheus-architecture.md) - Kiến trúc chi tiết của Prometheus

## Tài Liệu Tham Khảo

- [Prometheus Official Website](https://prometheus.io/) - Trang chủ chính thức
- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/) - Tài liệu chính thức
- [CNCF Prometheus Project](https://www.cncf.io/projects/prometheus/) - Thông tin về Prometheus trong CNCF
