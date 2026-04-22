# Troubleshooting Guide

## Prometheus Server

### Prometheus Không Start

**Triệu chứng**: Service fails to start

```bash
# Check logs
sudo journalctl -u prometheus -n 50

# Validate config
promtool check config /etc/prometheus/prometheus.yml

# Check permissions
ls -la /etc/prometheus/ /var/lib/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

**Common causes**:
- Config syntax error → `promtool check config`
- Permission denied → fix ownership
- Port already in use → `sudo lsof -i :9090`

---

### Target DOWN

**Triệu chứng**: Target shows state "DOWN" in UI

```bash
# Check service running
sudo systemctl status node_exporter

# Test connectivity
curl http://localhost:9100/metrics

# Check port
sudo netstat -tlnp | grep 9100

# Check firewall
sudo ufw status
sudo firewall-cmd --list-all
```

**Common causes**:
- Service not running → start service
- Wrong port → check config
- Firewall blocking → open port
- Network issue → check connectivity

---

### Scrape Errors

**Triệu chứng**: Errors in target details

```bash
# Check Prometheus logs
sudo journalctl -u prometheus | grep -i "scrape\|error"

# Test scrape manually
curl -v http://target:port/metrics

# Check scrape timeout
# Increase scrape_timeout in prometheus.yml
```

---

### High Memory Usage

**Triệu chứng**: Prometheus using too much RAM

```bash
# Check memory
ps aux | grep prometheus

# Check TSDB stats
curl http://localhost:9090/api/v1/status/tsdb | jq .

# Check cardinality
curl http://localhost:9090/api/v1/status/tsdb | jq '.data.seriesCountByMetricName[:10]'
```

**Solutions**:
- Reduce retention: `--storage.tsdb.retention.time=7d`
- Drop high-cardinality metrics with `metric_relabel_configs`
- Increase scrape interval
- Use recording rules for complex queries

---

### Slow Queries

**Triệu chứng**: Queries take too long

```bash
# Check query stats in UI
# Navigate to Graph → Execute → check execution time

# Use recording rules for expensive queries
# Reduce time range
# Add more specific label filters
```

---

## Alertmanager

### Alerts Not Firing

**Triệu chứng**: Alert condition met but no notification

```bash
# Check rules loaded
curl http://localhost:9090/api/v1/rules | jq '.data.groups[].rules[] | select(.type=="alerting")'

# Check alert state
curl http://localhost:9090/api/v1/alerts | jq .

# Validate rules
promtool check rules /etc/prometheus/rules/*.yml

# Check Prometheus connected to Alertmanager
curl http://localhost:9090/api/v1/alertmanagers
```

---

### Notifications Not Sent

**Triệu chứng**: Alert firing but no email/Slack

```bash
# Check Alertmanager logs
sudo journalctl -u alertmanager -f

# Validate config
amtool check-config /etc/alertmanager/alertmanager.yml

# Check Alertmanager status
curl http://localhost:9093/api/v2/status | jq .

# Test SMTP
telnet smtp.gmail.com 587

# Check active alerts in Alertmanager
curl http://localhost:9093/api/v2/alerts | jq .
```

---

### Too Many Notifications

**Solutions**:
- Increase `repeat_interval`
- Improve `group_by` to batch related alerts
- Add inhibition rules
- Use silences for known issues

---

## PromQL Issues

### Query Returns Empty

```promql
# Check metric exists
{__name__=~"metric_name.*"}

# Check time range has data
metric_name[1h]

# Check labels
metric_name{job="my_job"}

# Check target is up
up{job="my_job"}
```

---

### histogram_quantile Returns NaN

**Cause**: Not enough data or wrong query

```promql
# Wrong - missing rate()
histogram_quantile(0.95, metric_bucket)

# Correct
histogram_quantile(0.95, rate(metric_bucket[5m]))

# Check bucket data exists
metric_bucket
```

---

### Rate Returns 0 or Unexpected Values

```promql
# Check counter has data
metric_total

# Check time range is sufficient (>= 4x scrape_interval)
rate(metric_total[1m])  # Too short if scrape_interval=15s
rate(metric_total[5m])  # Better

# Check for counter resets
resets(metric_total[1h])
```

---

## Exporters

### node_exporter Not Collecting Metrics

```bash
# Check service
sudo systemctl status node_exporter

# Check metrics
curl http://localhost:9100/metrics | grep node_cpu

# Check collectors enabled
node_exporter --help | grep collector

# Run with debug
node_exporter --log.level=debug
```

---

### blackbox_exporter Probe Fails

```bash
# Test probe manually
curl 'http://localhost:9115/probe?target=https://example.com&module=http_2xx'

# Check target reachable
curl -I https://example.com

# Check blackbox config
cat /etc/blackbox_exporter/blackbox.yml

# Check logs
sudo journalctl -u blackbox_exporter -f
```

---

### postgres_exporter Connection Failed

```bash
# Test connection
psql -U prometheus_monitor -h localhost -d postgres

# Check pg_hba.conf
sudo grep md5 /var/lib/pgsql/data/pg_hba.conf

# Check DATA_SOURCE_NAME
sudo systemctl cat postgres_exporter | grep DATA_SOURCE

# Check PostgreSQL logs
sudo journalctl -u postgresql -f
```

---

## Service Discovery

### Targets Not Discovered

```bash
# Check file exists and valid JSON
jq . /etc/prometheus/file_sd/targets.json

# Check permissions
sudo -u prometheus cat /etc/prometheus/file_sd/targets.json

# Check Prometheus logs
sudo journalctl -u prometheus | grep file_sd

# Wait for refresh_interval
# Default: 5 minutes
```

---

### Labels Not Applied

```bash
# Check labels in target file
jq '.[] | .labels' /etc/prometheus/file_sd/targets.json

# Check relabel_configs syntax
promtool check config /etc/prometheus/prometheus.yml

# Check labels in Prometheus UI
# Status → Targets → click target
```

---

## Storage Issues

### Disk Space Full

```bash
# Check disk usage
df -h /var/lib/prometheus

# Check TSDB size
du -sh /var/lib/prometheus/

# Reduce retention
# Edit prometheus.service:
# --storage.tsdb.retention.time=7d

# Delete old data (careful!)
curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=old_metric'
curl -X POST http://localhost:9090/api/v1/admin/tsdb/clean_tombstones
```

---

### Data Corruption

```bash
# Check TSDB integrity
promtool tsdb analyze /var/lib/prometheus

# Repair (may lose data)
promtool tsdb repair /var/lib/prometheus

# Check WAL
ls -la /var/lib/prometheus/wal/
```

---

## Quick Diagnostic Commands

```bash
# Check all Prometheus services
sudo systemctl status prometheus alertmanager node_exporter

# Check all ports
sudo netstat -tlnp | grep -E '9090|9091|9093|9100|9115|9187'

# Check Prometheus health
curl http://localhost:9090/-/healthy
curl http://localhost:9090/-/ready

# Check Alertmanager health
curl http://localhost:9093/-/healthy

# Reload configs without restart
curl -X POST http://localhost:9090/-/reload
curl -X POST http://localhost:9093/-/reload

# Check active alerts
curl http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | {name: .labels.alertname, state: .state}'

# Check targets
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```
