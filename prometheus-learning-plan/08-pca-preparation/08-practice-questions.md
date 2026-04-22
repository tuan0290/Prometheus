# PCA Practice Questions

## Hướng Dẫn

- Đọc kỹ câu hỏi trước khi chọn đáp án
- Click vào "Đáp án" để xem giải thích
- Mục tiêu: đạt 80%+ trước khi thi

---

## Domain 1: Observability Concepts (18%)

**Q1.1**: Đâu là sự khác biệt chính giữa metrics và logs?

A) Metrics là text, logs là numbers  
B) Metrics là aggregated time-series, logs là detailed event records  
C) Metrics chỉ dùng cho alerting, logs chỉ dùng cho debugging  
D) Không có sự khác biệt

<details>
<summary>Đáp án: B</summary>

Metrics là aggregated numerical data theo thời gian (time-series), phù hợp cho monitoring và alerting. Logs là detailed text records của events, phù hợp cho debugging. Cả hai đều có thể dùng cho alerting và debugging, nhưng với mục đích khác nhau.

</details>

---

**Q1.2**: Prometheus sử dụng mô hình nào để thu thập metrics?

A) Push model  
B) Pull model  
C) Cả push và pull  
D) Event-driven model

<details>
<summary>Đáp án: B</summary>

Prometheus sử dụng pull model: Prometheus server chủ động scrape metrics từ targets bằng cách gửi HTTP GET request đến `/metrics` endpoint. Pushgateway là exception cho batch jobs, nhưng Prometheus vẫn pull từ Pushgateway.

</details>

---

**Q1.3**: SLO 99.9% availability có nghĩa là gì?

A) Service phải up 99.9% thời gian  
B) 99.9% requests phải thành công  
C) Error budget là 0.1% (~43.8 phút/tháng)  
D) A và C đều đúng

<details>
<summary>Đáp án: D</summary>

SLO 99.9% availability có thể có nghĩa là service phải up 99.9% thời gian, VÀ error budget là 1 - 0.999 = 0.001 = 0.1%, tương đương ~43.8 phút downtime mỗi tháng.

</details>

---

**Q1.4**: Khi nào nên dùng Pushgateway?

A) Cho tất cả services  
B) Cho long-running services  
C) Cho batch jobs và short-lived processes  
D) Khi Prometheus không thể scrape trực tiếp

<details>
<summary>Đáp án: C</summary>

Pushgateway phù hợp cho batch jobs và short-lived processes không tồn tại đủ lâu để Prometheus scrape. Không nên dùng cho long-running services vì pull model hiệu quả hơn và dễ detect khi service down.

</details>

---

**Q1.5**: Four Golden Signals của Google SRE là gì?

A) CPU, Memory, Disk, Network  
B) Latency, Traffic, Errors, Saturation  
C) Availability, Reliability, Performance, Security  
D) Metrics, Logs, Traces, Events

<details>
<summary>Đáp án: B</summary>

Four Golden Signals: Latency (thời gian xử lý), Traffic (số requests), Errors (tỷ lệ lỗi), Saturation (mức độ sử dụng tài nguyên). Đây là framework của Google SRE để monitor services.

</details>

---

## Domain 2: Prometheus Fundamentals (20%)

**Q2.1**: Metric type nào chỉ có thể tăng?

A) Gauge  
B) Counter  
C) Histogram  
D) Summary

<details>
<summary>Đáp án: B</summary>

Counter chỉ có thể tăng (hoặc reset về 0 khi process restart). Gauge có thể tăng hoặc giảm. Histogram và Summary đo distributions.

</details>

---

**Q2.2**: Đâu là cách đúng để query counter metric?

A) `http_requests_total`  
B) `rate(http_requests_total[5m])`  
C) `delta(http_requests_total[5m])`  
D) `avg(http_requests_total)`

<details>
<summary>Đáp án: B</summary>

Counters luôn tăng, nên query trực tiếp không có ý nghĩa. `rate()` tính per-second rate, là cách đúng để query counters. `delta()` dùng cho gauges, không phải counters.

</details>

---

**Q2.3**: Labels nào được Prometheus tự động thêm vào khi scrape?

A) `env` và `region`  
B) `job` và `instance`  
C) `host` và `port`  
D) `service` và `version`

<details>
<summary>Đáp án: B</summary>

Prometheus tự động thêm `job` (từ job_name trong config) và `instance` (từ target address) labels vào tất cả scraped metrics.

</details>

---

**Q2.4**: Default retention period của Prometheus là bao lâu?

A) 7 ngày  
B) 15 ngày  
C) 30 ngày  
D) 90 ngày

<details>
<summary>Đáp án: B</summary>

Default retention của Prometheus là 15 ngày (`--storage.tsdb.retention.time=15d`). Có thể thay đổi bằng command line flag.

</details>

---

**Q2.5**: Sự khác biệt giữa Histogram và Summary là gì?

A) Histogram chính xác hơn Summary  
B) Summary có thể aggregate across instances, Histogram không  
C) Histogram có thể aggregate across instances, Summary không  
D) Không có sự khác biệt

<details>
<summary>Đáp án: C</summary>

Histogram tính quantiles server-side (trong PromQL), nên có thể aggregate across instances. Summary tính quantiles client-side, không thể aggregate. Histogram thường được recommend hơn vì tính linh hoạt.

</details>

---

**Q2.6**: File-based service discovery refresh interval mặc định là bao lâu?

A) 5 giây  
B) 15 giây  
C) 30 giây  
D) 5 phút

<details>
<summary>Đáp án: D</summary>

Default refresh interval cho file-based service discovery là 5 phút. Có thể cấu hình với `refresh_interval` trong `file_sd_configs`.

</details>

---

## Domain 3: PromQL (28%)

**Q3.1**: Query nào tính per-second request rate trong 5 phút?

A) `http_requests_total[5m]`  
B) `rate(http_requests_total[5m])`  
C) `increase(http_requests_total[5m])`  
D) `delta(http_requests_total[5m])`

<details>
<summary>Đáp án: B</summary>

`rate()` tính per-second average rate trong time range. `increase()` tính total increase (không per-second). `delta()` dùng cho gauges. `[5m]` là range vector, không thể graph trực tiếp.

</details>

---

**Q3.2**: Query nào tính 95th percentile latency từ histogram?

A) `http_request_duration_seconds{quantile="0.95"}`  
B) `histogram_quantile(0.95, http_request_duration_seconds_bucket)`  
C) `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))`  
D) `quantile(0.95, http_request_duration_seconds)`

<details>
<summary>Đáp án: C</summary>

Đúng cách: `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))`. Cần dùng `rate()` với bucket để tính rate trước, sau đó `histogram_quantile()` tính percentile. Option A là Summary syntax, không phải Histogram.

</details>

---

**Q3.3**: Sự khác biệt giữa `by` và `without` trong aggregation là gì?

A) `by` giữ labels được chỉ định, `without` loại bỏ labels được chỉ định  
B) `by` loại bỏ labels được chỉ định, `without` giữ labels được chỉ định  
C) Không có sự khác biệt  
D) `by` dùng cho sum, `without` dùng cho avg

<details>
<summary>Đáp án: A</summary>

`sum by (job) (metric)` giữ lại chỉ label `job`. `sum without (instance) (metric)` loại bỏ label `instance`, giữ tất cả labels còn lại.

</details>

---

**Q3.4**: Query nào trả về top 5 endpoints theo request count?

A) `top5(sum by (endpoint) (http_requests_total))`  
B) `topk(5, sum by (endpoint) (http_requests_total))`  
C) `sort_desc(sum by (endpoint) (http_requests_total))[0:5]`  
D) `limit(5, sum by (endpoint) (http_requests_total))`

<details>
<summary>Đáp án: B</summary>

`topk(k, expr)` trả về k time series có giá trị cao nhất. `topk(5, sum by (endpoint) (http_requests_total))` trả về 5 endpoints có nhiều requests nhất.

</details>

---

**Q3.5**: Offset modifier dùng để làm gì?

A) Shift query forward in time  
B) Shift query backward in time  
C) Add time offset to timestamps  
D) Change evaluation interval

<details>
<summary>Đáp án: B</summary>

`offset` shift query backward in time. `metric offset 1h` trả về giá trị của metric 1 giờ trước. Dùng để so sánh current vs historical values.

</details>

---

**Q3.6**: Query nào tính error rate percentage?

A) `http_requests_total{status="500"} / http_requests_total * 100`  
B) `rate(http_requests_total{status="500"}[5m]) / rate(http_requests_total[5m]) * 100`  
C) `sum(http_requests_total{status="500"}) / sum(http_requests_total) * 100`  
D) `count(http_requests_total{status="500"}) / count(http_requests_total) * 100`

<details>
<summary>Đáp án: B</summary>

Đúng cách: dùng `rate()` với counters để tính per-second rate, sau đó tính tỷ lệ. Option A query counter trực tiếp (sai). Option C dùng sum của counter values (sai). Option D dùng count (đếm time series, không phải requests).

</details>

---

**Q3.7**: `absent()` function trả về gì?

A) 0 nếu metric tồn tại, 1 nếu không  
B) 1 nếu metric absent, empty nếu metric tồn tại  
C) Empty nếu metric absent, 1 nếu tồn tại  
D) Luôn trả về 1

<details>
<summary>Đáp án: B</summary>

`absent(expr)` trả về 1 nếu expr không có time series (metric absent), và trả về empty result nếu expr có time series. Dùng để alert khi metric bị missing.

</details>

---

## Domain 4: Instrumentation and Exporters (20%)

**Q4.1**: Metric name nào tuân theo naming conventions?

A) `HTTPRequestsTotal`  
B) `http-requests-total`  
C) `http_requests_total`  
D) `httpRequestsTotal`

<details>
<summary>Đáp án: C</summary>

Prometheus naming conventions: lowercase, underscores (không hyphens hoặc camelCase), counters kết thúc bằng `_total`. `http_requests_total` là đúng.

</details>

---

**Q4.2**: Đâu là ví dụ về high cardinality label (nên tránh)?

A) `status_code` với values "200", "404", "500"  
B) `method` với values "GET", "POST", "PUT"  
C) `user_id` với millions of unique values  
D) `environment` với values "prod", "staging", "dev"

<details>
<summary>Đáp án: C</summary>

`user_id` có thể có millions of unique values, tạo ra millions of time series. Điều này gây memory và performance issues. Labels nên có low cardinality (< 100 unique values).

</details>

---

**Q4.3**: node_exporter expose metrics trên port nào?

A) 9090  
B) 9091  
C) 9093  
D) 9100

<details>
<summary>Đáp án: D</summary>

node_exporter mặc định expose metrics trên port 9100. Prometheus server: 9090, Pushgateway: 9091, Alertmanager: 9093.

</details>

---

**Q4.4**: Khi scrape Pushgateway, tại sao cần `honor_labels: true`?

A) Để tăng performance  
B) Để giữ labels từ pushed metrics  
C) Để enable authentication  
D) Để set scrape timeout

<details>
<summary>Đáp án: B</summary>

`honor_labels: true` giữ nguyên labels từ pushed metrics (job, instance từ batch jobs). Nếu không có, Prometheus sẽ override bằng labels từ scrape config, làm mất thông tin về job gốc.

</details>

---

**Q4.5**: blackbox_exporter dùng để làm gì?

A) Monitor Linux system metrics  
B) Probe endpoints (HTTP, TCP, ICMP, DNS)  
C) Monitor PostgreSQL  
D) Collect JVM metrics

<details>
<summary>Đáp án: B</summary>

blackbox_exporter probe external endpoints: HTTP (check status code, response time), TCP (check connectivity), ICMP (ping), DNS (check resolution). node_exporter cho system metrics, postgres_exporter cho PostgreSQL.

</details>

---

## Domain 5: Alerting and Dashboarding (14%)

**Q5.1**: Alert ở trạng thái "pending" có nghĩa là gì?

A) Alert đã được gửi đến Alertmanager  
B) Condition đang true nhưng chưa đủ `for` duration  
C) Alert đang bị silenced  
D) Alert đã được resolved

<details>
<summary>Đáp án: B</summary>

"Pending" nghĩa là PromQL expression đang true nhưng chưa đủ thời gian `for` duration. Sau khi đủ `for` duration, alert chuyển sang "firing" và được gửi đến Alertmanager.

</details>

---

**Q5.2**: `group_wait` trong Alertmanager config có tác dụng gì?

A) Thời gian chờ trước khi gửi notification đầu tiên  
B) Thời gian chờ giữa các notifications  
C) Thời gian chờ trước khi resolve alert  
D) Thời gian chờ trước khi retry

<details>
<summary>Đáp án: A</summary>

`group_wait` là thời gian Alertmanager chờ sau khi nhận alert đầu tiên trong group trước khi gửi notification. Cho phép batch nhiều alerts vào một notification.

</details>

---

**Q5.3**: Inhibition rules trong Alertmanager dùng để làm gì?

A) Mute alerts trong maintenance window  
B) Suppress lower-severity alerts khi higher-severity alert firing  
C) Route alerts đến different receivers  
D) Group related alerts

<details>
<summary>Đáp án: B</summary>

Inhibition rules suppress (mute) target alerts khi source alert đang firing. Ví dụ: suppress warning alerts khi critical alert firing cho cùng instance, tránh notification spam.

</details>

---

**Q5.4**: Đâu là best practice khi viết alerting rules?

A) Alert on causes, not symptoms  
B) Set `for: 0s` để alert ngay lập tức  
C) Alert on symptoms, not causes  
D) Không cần annotations

<details>
<summary>Đáp án: C</summary>

Best practice: alert on symptoms (user-facing issues) không phải causes (internal issues). Ví dụ: alert on high error rate (symptom) thay vì high CPU (cause). Cũng cần `for` duration để tránh flapping, và annotations để cung cấp context.

</details>

---

**Q5.5**: Grafana variable `$instance` được define như thế nào?

A) Hardcoded trong dashboard  
B) Query: `label_values(up, instance)`  
C) Environment variable  
D) Command line argument

<details>
<summary>Đáp án: B</summary>

Grafana variables thường được define bằng PromQL query. `label_values(up, instance)` trả về tất cả unique values của label `instance` từ metric `up`. Variable này sau đó có thể dùng trong panel queries như `{instance="$instance"}`.

</details>

---

## Mixed Domain Questions

**Q6.1**: Query nào phù hợp nhất để alert khi disk space < 20%?

A) `node_filesystem_avail_bytes < 0.2`  
B) `node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.2`  
C) `(node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes > 0.8`  
D) B và C đều đúng

<details>
<summary>Đáp án: D</summary>

Cả B và C đều tính disk usage percentage và alert khi < 20% available (hoặc > 80% used). B tính available percentage, C tính used percentage. Cả hai đều valid.

</details>

---

**Q6.2**: Đâu là cách đúng để monitor SSL certificate expiry?

A) node_exporter  
B) blackbox_exporter với `probe_ssl_earliest_cert_expiry`  
C) Custom exporter  
D) Prometheus không thể monitor SSL certs

<details>
<summary>Đáp án: B</summary>

blackbox_exporter expose `probe_ssl_earliest_cert_expiry` metric khi probe HTTPS endpoints. Query: `(probe_ssl_earliest_cert_expiry - time()) / 86400 < 30` để alert khi cert expires trong 30 ngày.

</details>

---

**Q6.3**: Recording rule nào phù hợp cho query `sum by (job) (rate(http_requests_total[5m]))`?

A) `job:http_requests:rate5m`  
B) `http_requests_rate_5m`  
C) `rate_http_requests_by_job`  
D) `job_http_requests_total_rate`

<details>
<summary>Đáp án: A</summary>

Recording rule naming convention: `level:metric:operations`. `job:http_requests:rate5m` = level "job" (aggregated by job), metric "http_requests", operation "rate5m" (5-minute rate). Đây là convention chuẩn của Prometheus.

</details>

---

## Score Tracking

| Domain | Số Câu | Đúng | Tỷ Lệ |
|--------|--------|------|-------|
| Domain 1: Observability | 5 | | |
| Domain 2: Fundamentals | 6 | | |
| Domain 3: PromQL | 7 | | |
| Domain 4: Instrumentation | 5 | | |
| Domain 5: Alerting | 5 | | |
| Mixed | 3 | | |
| **Tổng** | **31** | | |

**Mục tiêu**: 80%+ (25/31 câu đúng)

## Tiếp Theo

- [Exam Tips](./09-exam-tips.md)
- Nếu score < 80%: Review domain tương ứng
- Nếu score >= 80%: Sẵn sàng đăng ký thi!
