# PCA Exam Tips

## Chiến Lược Làm Bài

### 1. Quản Lý Thời Gian

- **Tổng thời gian**: 90 phút, ~60 câu
- **Thời gian/câu**: ~90 giây
- **Chiến lược**:
  - Lần 1: Làm tất cả câu dễ (~45 phút)
  - Lần 2: Quay lại câu đã flag (~30 phút)
  - Lần 3: Review toàn bộ (~15 phút)

### 2. Đọc Câu Hỏi Cẩn Thận

Chú ý các từ khóa:

| Từ Khóa | Ý Nghĩa |
|---------|---------|
| "always" | Tuyệt đối, không có exception |
| "never" | Không bao giờ |
| "best practice" | Cách được recommend nhất |
| "most appropriate" | Phù hợp nhất trong context |
| "NOT" | Tìm đáp án sai |
| "EXCEPT" | Tất cả đúng trừ một |

### 3. Loại Trừ Đáp Án

Thường có 1-2 đáp án rõ ràng sai:
1. Loại bỏ đáp án sai rõ ràng
2. So sánh các đáp án còn lại
3. Chọn đáp án phù hợp nhất

### 4. Flag và Skip

- Flag câu không chắc chắn
- Không dừng quá lâu ở một câu
- Quay lại sau khi làm xong

---

## Key Concepts Cần Nhớ

### PromQL (Domain 3 - 28%)

```promql
# Counter → luôn dùng rate()
rate(counter_metric[5m])

# Histogram quantile
histogram_quantile(0.95, rate(metric_bucket[5m]))

# Aggregation
sum by (label) (metric)
sum without (label) (metric)

# TopK
topk(5, sum by (job) (metric))

# Offset
metric offset 1h

# Absent
absent(up{job="service"})
```

### Metric Types

| Type | Chỉ Tăng | Dùng rate() | Aggregatable |
|------|----------|-------------|--------------|
| Counter | ✅ | ✅ Required | ✅ |
| Gauge | ❌ | ❌ | ✅ |
| Histogram | N/A | ✅ for buckets | ✅ |
| Summary | N/A | ✅ for sum/count | ❌ |

### Ports Cần Nhớ

| Service | Port |
|---------|------|
| Prometheus | 9090 |
| Pushgateway | 9091 |
| Alertmanager | 9093 |
| node_exporter | 9100 |
| blackbox_exporter | 9115 |
| postgres_exporter | 9187 |

### Alert States

```
Inactive → Pending → Firing → Resolved
```

- **Inactive**: Condition false
- **Pending**: Condition true, waiting for `for` duration
- **Firing**: Condition true for `for` duration
- **Resolved**: Condition false again

### Alertmanager Timing

| Parameter | Mô Tả |
|-----------|-------|
| `group_wait` | Chờ trước khi gửi notification đầu tiên |
| `group_interval` | Chờ trước khi gửi alerts mới trong group |
| `repeat_interval` | Chờ trước khi resend notification |

---

## Common Traps

### Trap 1: Counter vs Gauge

```promql
# WRONG: Query counter directly
http_requests_total

# CORRECT: Use rate()
rate(http_requests_total[5m])
```

### Trap 2: Histogram Quantile

```promql
# WRONG: Missing rate()
histogram_quantile(0.95, http_request_duration_seconds_bucket)

# CORRECT: With rate()
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

### Trap 3: Summary vs Histogram

- **Summary**: Quantiles tính client-side, KHÔNG aggregate được
- **Histogram**: Quantiles tính server-side, CÓ aggregate được
- Exam thường hỏi về aggregation → Histogram

### Trap 4: Pushgateway

- Pushgateway KHÔNG phải replacement cho pull model
- Chỉ dùng cho batch jobs và short-lived processes
- Cần `honor_labels: true` khi scrape

### Trap 5: Label Cardinality

- High cardinality labels (user_id, session_id) → AVOID
- Low cardinality labels (status, method, env) → OK
- Cardinality = number of unique time series

### Trap 6: `by` vs `without`

```promql
# by: KEEP specified labels
sum by (job) (metric)  # Only job label remains

# without: REMOVE specified labels
sum without (instance) (metric)  # All labels except instance
```

---

## Domain-Specific Tips

### Domain 1: Observability (18%)

- Nhớ 3 pillars: Metrics, Logs, Traces
- Push vs Pull: Prometheus dùng Pull
- SLI → SLO → SLA (từ measurement đến contract)
- Error budget = 1 - SLO

### Domain 2: Fundamentals (20%)

- Architecture: Prometheus scrapes /metrics endpoints
- 4 metric types và khi nào dùng mỗi loại
- Labels: job và instance được thêm tự động
- Default retention: 15 days
- Histogram vs Summary: Histogram aggregatable

### Domain 3: PromQL (28%) - QUAN TRỌNG NHẤT

- `rate()` cho counters, `delta()` cho gauges
- `histogram_quantile()` cần `rate()` với buckets
- `by` giữ labels, `without` loại bỏ labels
- `topk(k, expr)` trả về k highest values
- `absent()` trả về 1 nếu metric missing
- `offset` shift backward in time

### Domain 4: Instrumentation (20%)

- Naming: lowercase, underscores, base units, `_total` suffix
- node_exporter: port 9100, system metrics
- blackbox_exporter: probe HTTP/TCP/ICMP
- Pushgateway: batch jobs only, `honor_labels: true`
- High cardinality labels → avoid

### Domain 5: Alerting (14%)

- Alert states: inactive → pending → firing → resolved
- `for` duration: prevent flapping
- Alertmanager: grouping, inhibition, silences
- `group_wait` vs `group_interval` vs `repeat_interval`
- Inhibition: suppress warnings when critical firing

---

## Ngày Thi

### Trước Khi Thi

- [ ] Test webcam và microphone
- [ ] Đảm bảo internet ổn định
- [ ] Dọn dẹp bàn làm việc (không có tài liệu)
- [ ] Chuẩn bị ID hợp lệ
- [ ] Vào phòng thi 30 phút trước

### Trong Khi Thi

- [ ] Đọc kỹ từng câu hỏi
- [ ] Flag câu không chắc
- [ ] Không dừng quá lâu ở một câu
- [ ] Review flagged questions
- [ ] Kiểm tra lại trước khi submit

### Sau Khi Thi

- Kết quả thường có ngay sau khi submit
- Nếu không đậu: review weak areas và retake sau 2 tuần
- Nếu đậu: chứng chỉ valid 2 năm

---

## Quick Reference Card

### PromQL Cheat Sheet

```promql
# Rate (counters)
rate(metric[5m])

# Increase (counters)
increase(metric[1h])

# Percentile (histograms)
histogram_quantile(0.95, rate(metric_bucket[5m]))

# Aggregations
sum by (label) (metric)
avg without (instance) (metric)
topk(5, metric)
count(up)

# Comparisons
metric > 100
metric{label="value"}

# Time
metric offset 1h
avg_over_time(metric[1h])

# Utility
absent(metric)
predict_linear(metric[1h], 3600)
```

### Alerting Cheat Sheet

```yaml
# Alert rule
- alert: AlertName
  expr: condition > threshold
  for: 5m
  labels:
    severity: warning|critical
  annotations:
    summary: "Short description"
    description: "Detailed: {{ $value }}"

# Alertmanager routing
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: default
```

---

## Chúc Bạn Thi Tốt!

Nếu bạn đã:
- ✅ Hoàn thành tất cả 6 labs
- ✅ Đạt 80%+ practice questions
- ✅ Nắm vững PromQL
- ✅ Hiểu alerting workflow

Bạn đã sẵn sàng cho kỳ thi PCA!

**Đăng ký thi**: [https://training.linuxfoundation.org/certification/prometheus-certified-associate/](https://training.linuxfoundation.org/certification/prometheus-certified-associate/)
