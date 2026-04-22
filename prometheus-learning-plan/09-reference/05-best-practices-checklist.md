# Best Practices Checklist

## Installation & Configuration

- [ ] Chạy Prometheus với dedicated non-root user (`prometheus`)
- [ ] Set retention policy phù hợp (`--storage.tsdb.retention.time`)
- [ ] Enable `--web.enable-lifecycle` để hot reload config
- [ ] Validate config trước khi deploy: `promtool check config`
- [ ] Validate rules: `promtool check rules`
- [ ] Version control tất cả config files
- [ ] Backup config và data định kỳ

## Metric Naming

- [ ] Dùng lowercase và underscores
- [ ] Include units trong tên metric (seconds, bytes, total)
- [ ] Dùng base units (seconds không milliseconds)
- [ ] Counters kết thúc bằng `_total`
- [ ] Format: `<namespace>_<subsystem>_<name>_<unit>`
- [ ] Tên rõ ràng, mô tả đúng metric

## Labels

- [ ] Giữ cardinality thấp (< 100 unique values per label)
- [ ] Không dùng user_id, session_id, request_id làm labels
- [ ] Labels có ý nghĩa và consistent
- [ ] Không thay đổi label names sau khi deploy
- [ ] Dùng `job` và `instance` labels đúng cách

## PromQL

- [ ] Luôn dùng `rate()` hoặc `increase()` với counters
- [ ] Dùng `rate()` với histogram buckets trước `histogram_quantile()`
- [ ] Chọn time range phù hợp (>= 4x scrape_interval)
- [ ] Dùng recording rules cho complex/expensive queries
- [ ] Test queries trước khi dùng trong alerts

## Alerting

- [ ] Alert on symptoms, không phải causes
- [ ] Set `for` duration để tránh flapping (minimum 1-2 phút)
- [ ] Dùng severity labels (critical, warning, info)
- [ ] Viết annotations rõ ràng với summary và description
- [ ] Include runbook links trong annotations
- [ ] Test alerts định kỳ
- [ ] Dùng silences cho planned maintenance

## Alertmanager

- [ ] Group related alerts để giảm noise
- [ ] Set `repeat_interval` phù hợp (tránh spam)
- [ ] Cấu hình inhibition rules
- [ ] Test notification channels
- [ ] Document routing rules
- [ ] Backup Alertmanager config

## Security

- [ ] Restrict network access với firewall
- [ ] Dùng TLS cho Prometheus endpoints
- [ ] Không expose Prometheus trực tiếp ra internet
- [ ] Dùng reverse proxy (nginx) với authentication
- [ ] Rotate credentials định kỳ
- [ ] Audit access logs

## Performance

- [ ] Monitor Prometheus itself (self-monitoring)
- [ ] Set scrape_timeout < scrape_interval
- [ ] Dùng recording rules cho expensive queries
- [ ] Monitor cardinality: `prometheus_tsdb_head_series`
- [ ] Set appropriate scrape intervals (không quá thấp)
- [ ] Monitor scrape duration: `scrape_duration_seconds`

## Exporters

- [ ] Dùng official exporters khi có thể
- [ ] Verify exporter metrics accuracy
- [ ] Monitor exporter health
- [ ] Document exporter configuration
- [ ] Set appropriate scrape intervals per exporter

## Storage

- [ ] Monitor disk usage: `node_filesystem_avail_bytes`
- [ ] Set retention phù hợp với use case
- [ ] Cân nhắc remote storage cho long-term retention
- [ ] Regular backup của TSDB data
- [ ] Monitor TSDB stats: `prometheus_tsdb_*`

## Dashboards (Grafana)

- [ ] Dùng variables cho flexible filtering
- [ ] Include time range selector
- [ ] Document panel descriptions
- [ ] Use consistent color schemes
- [ ] Include links to runbooks
- [ ] Export và version control dashboards
