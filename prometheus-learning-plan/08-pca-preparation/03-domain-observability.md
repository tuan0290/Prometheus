# Domain 1: Observability Concepts (18%)

## Tổng Quan

Domain này chiếm 18% kỳ thi PCA. Tập trung vào các khái niệm nền tảng về observability, monitoring, và các metrics liên quan đến SLI/SLO/SLA.

## 1. Ba Trụ Cột Của Observability

### Metrics

Số liệu đo lường được theo thời gian.

- **Định nghĩa**: Giá trị số được thu thập định kỳ
- **Ví dụ**: CPU usage, request count, error rate
- **Ưu điểm**: Hiệu quả, dễ aggregate, phù hợp alerting
- **Nhược điểm**: Không có context chi tiết

### Logs

Bản ghi sự kiện theo thời gian.

- **Định nghĩa**: Text records của events xảy ra trong system
- **Ví dụ**: Application logs, access logs, error logs
- **Ưu điểm**: Chi tiết, dễ debug
- **Nhược điểm**: Volume lớn, khó aggregate

### Traces

Theo dõi request qua nhiều services.

- **Định nghĩa**: End-to-end tracking của request
- **Ví dụ**: Distributed tracing với Jaeger, Zipkin
- **Ưu điểm**: Hiểu latency, bottlenecks
- **Nhược điểm**: Overhead, phức tạp

### Events

Sự kiện rời rạc xảy ra trong system.

- **Định nghĩa**: Discrete occurrences (deployments, config changes)
- **Ví dụ**: Deployment events, scaling events
- **Ưu điểm**: Context cho anomalies
- **Nhược điểm**: Không continuous

```
Observability Pillars:
┌─────────────┬─────────────┬─────────────┬─────────────┐
│   Metrics   │    Logs     │   Traces    │   Events    │
│             │             │             │             │
│ Aggregated  │  Detailed   │  Request    │  Discrete   │
│ time-series │  text data  │  tracking   │  occurrences│
│             │             │             │             │
│ Prometheus  │    ELK      │   Jaeger    │  Alerting   │
└─────────────┴─────────────┴─────────────┴─────────────┘
```

## 2. Push vs Pull Model

### Pull Model (Prometheus)

Prometheus chủ động scrape metrics từ targets.

```
Pull Model:
┌─────────────┐     scrape     ┌─────────────┐
│  Prometheus │ ─────────────> │   Target    │
│   Server    │ <───────────── │  /metrics   │
└─────────────┘    metrics     └─────────────┘
```

**Ưu điểm**:
- Prometheus kiểm soát scrape frequency
- Dễ phát hiện target down (scrape fails)
- Không cần target biết về Prometheus
- Dễ debug (manual curl /metrics)

**Nhược điểm**:
- Targets phải expose HTTP endpoint
- Không phù hợp cho short-lived jobs
- Cần network access từ Prometheus đến targets

### Push Model (Pushgateway)

Targets chủ động push metrics đến Pushgateway.

```
Push Model:
┌─────────────┐     push      ┌─────────────┐     scrape    ┌─────────────┐
│  Batch Job  │ ─────────────>│ Pushgateway │ <─────────────│  Prometheus │
└─────────────┘               └─────────────┘               └─────────────┘
```

**Khi nào dùng Push**:
- Batch jobs (không tồn tại lâu)
- Jobs behind firewall
- Short-lived processes

**Lưu ý**: Pushgateway không phải giải pháp cho mọi trường hợp. Chỉ dùng khi pull không khả thi.

## 3. Service Level Indicators (SLIs)

SLI là metric đo lường một khía cạnh của service quality.

### Định Nghĩa

> SLI = Tỷ lệ các requests "tốt" trên tổng requests

### Ví Dụ SLIs

| Service | SLI | Công Thức |
|---------|-----|-----------|
| Web API | Availability | Successful requests / Total requests |
| Web API | Latency | % requests < 200ms |
| Database | Throughput | Queries per second |
| Storage | Durability | Data successfully stored / Data written |

### SLI trong PromQL

```promql
# Availability SLI
sum(rate(http_requests_total{status!~"5.."}[5m])) /
sum(rate(http_requests_total[5m]))

# Latency SLI (% requests < 200ms)
sum(rate(http_request_duration_seconds_bucket{le="0.2"}[5m])) /
sum(rate(http_request_duration_seconds_count[5m]))
```

## 4. Service Level Objectives (SLOs)

SLO là target value cho SLI.

### Định Nghĩa

> SLO = SLI target trong một time window

### Ví Dụ SLOs

| SLI | SLO |
|-----|-----|
| Availability | 99.9% over 30 days |
| Latency p95 | < 200ms over 7 days |
| Error rate | < 0.1% over 24 hours |

### Error Budget

```
Error Budget = 1 - SLO

Ví dụ:
SLO = 99.9% availability
Error Budget = 0.1% = 43.8 phút/tháng
```

### SLO Alerting

```yaml
# Alert khi error budget burned too fast
- alert: ErrorBudgetBurnRate
  expr: |
    (
      1 - sum(rate(http_requests_total{status!~"5.."}[1h])) /
          sum(rate(http_requests_total[1h]))
    ) > (1 - 0.999) * 14.4
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Error budget burning too fast"
```

## 5. Service Level Agreements (SLAs)

SLA là cam kết chính thức với khách hàng.

### Phân Biệt SLI, SLO, SLA

```
SLI → SLO → SLA

SLI: Measurement (what we measure)
SLO: Target (what we aim for)
SLA: Contract (what we promise)

Ví dụ:
SLI: Availability = 99.95%
SLO: Availability >= 99.9%
SLA: Availability >= 99.5% (với penalty nếu vi phạm)
```

### Relationship

- SLA thường ít strict hơn SLO (buffer)
- Vi phạm SLO = cảnh báo nội bộ
- Vi phạm SLA = penalty với khách hàng

## 6. Monitoring Best Practices

### The Four Golden Signals (Google SRE)

1. **Latency**: Thời gian xử lý request
2. **Traffic**: Số lượng requests
3. **Errors**: Tỷ lệ lỗi
4. **Saturation**: Mức độ sử dụng tài nguyên

```promql
# Latency (p95)
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Traffic (requests/sec)
sum(rate(http_requests_total[5m]))

# Errors (error rate)
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# Saturation (CPU)
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### USE Method (Brendan Gregg)

Cho infrastructure resources:

- **Utilization**: % time resource is busy
- **Saturation**: Amount of work queued
- **Errors**: Error count

### RED Method

Cho services:

- **Rate**: Requests per second
- **Errors**: Error rate
- **Duration**: Latency distribution

## 7. Key Concepts Summary

| Concept | Định Nghĩa | Ví Dụ |
|---------|-----------|-------|
| Metric | Số liệu đo lường | CPU usage = 45% |
| SLI | Metric đo service quality | Availability = 99.95% |
| SLO | Target cho SLI | Availability >= 99.9% |
| SLA | Cam kết với khách hàng | Availability >= 99.5% |
| Error Budget | 1 - SLO | 0.1% = 43.8 min/month |

## 8. Practice Questions

**Q1**: Sự khác biệt giữa metrics và logs là gì?

<details>
<summary>Đáp án</summary>

Metrics là số liệu aggregated theo thời gian (time series), phù hợp cho monitoring và alerting. Logs là text records chi tiết của events, phù hợp cho debugging. Metrics hiệu quả hơn về storage, logs cung cấp context chi tiết hơn.

</details>

**Q2**: Khi nào nên dùng Pushgateway thay vì pull model?

<details>
<summary>Đáp án</summary>

Pushgateway phù hợp cho: batch jobs (không tồn tại đủ lâu để scrape), jobs behind firewall không accessible từ Prometheus, short-lived processes. Không nên dùng Pushgateway như replacement cho pull model trong long-running services.

</details>

**Q3**: Error budget là gì và tại sao quan trọng?

<details>
<summary>Đáp án</summary>

Error budget = 1 - SLO. Ví dụ SLO 99.9% → error budget 0.1% (~43.8 phút/tháng). Error budget quan trọng vì: cho phép team cân bằng giữa reliability và feature development, khi budget cạn kiệt → focus vào reliability, khi budget còn nhiều → có thể ship features nhanh hơn.

</details>

## Tiếp Theo

- [Domain 2: Fundamentals](./04-domain-fundamentals.md)
- [Tài liệu tham khảo: Monitoring & Observability](../01-fundamentals/01-monitoring-observability.md)
