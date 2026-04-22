# Pushgateway

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Kiến Trúc](#kiến-trúc)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Tổng Quan

Pushgateway là một intermediate service cho phép short-lived jobs push metrics đến Prometheus. Nó giải quyết vấn đề của Prometheus pull model khi scrape targets không tồn tại đủ lâu để được scraped.

**Lưu ý quan trọng**: Pushgateway chỉ nên được sử dụng trong các trường hợp đặc biệt. Nó không phải là giải pháp chung cho tất cả monitoring needs.

## Kiến Trúc

```
┌─────────────────────────────────────────────────────────┐
│              Prometheus Server                          │
│                                                         │
│                  Scrape /metrics                        │
└────────────────────────┬────────────────────────────────┘
                         │
                         │ Pull
                         ▼
                  ┌──────────────┐
                  │ Pushgateway  │
                  │   :9091      │
                  └──────┬───────┘
                         ▲
                         │ Push
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  Batch   │    │  Cron    │    │  Script  │
  │   Job    │    │   Job    │    │   Job    │
  └──────────┘    └──────────┘    └──────────┘
```

### Data Flow

```
1. Job runs
   │
   ▼
2. Job pushes metrics to Pushgateway
   │
   ▼
3. Pushgateway stores metrics in memory
   │
   ▼
4. Prometheus scrapes Pushgateway
   │
   ▼
5. Metrics available in Prometheus
```

## Cách Hoạt Động

### Push Mechanism

Jobs push metrics đến Pushgateway qua HTTP POST/PUT:

```bash
# Push single metric
echo "some_metric 3.14" | curl --data-binary @- \
  http://pushgateway:9091/metrics/job/some_job

# Push multiple metrics
cat <<EOF | curl --data-binary @- \
  http://pushgateway:9091/metrics/job/batch_job/instance/server1
# TYPE batch_duration_seconds gauge
batch_duration_seconds 123.45
# TYPE batch_records_processed counter
batch_records_processed 1000
EOF
```

### Grouping Keys

Metrics được group theo job và optional instance labels:

```
URL format: /metrics/job/<JOB_NAME>{/label_name/<LABEL_VALUE>}

Examples:
/metrics/job/backup_job
/metrics/job/backup_job/instance/server1
/metrics/job/backup_job/instance/server1/database/mysql
```

### Metric Persistence

- Metrics được lưu trong memory của Pushgateway
- Metrics tồn tại cho đến khi:
  - Bị overwrite bởi push mới
  - Bị delete qua API
  - Pushgateway restart (metrics bị mất)

### Delete Metrics

```bash
# Delete all metrics for a job
curl -X DELETE http://pushgateway:9091/metrics/job/some_job

# Delete metrics for specific instance
curl -X DELETE http://pushgateway:9091/metrics/job/some_job/instance/server1
```

## Use Cases

### 1. Batch Jobs

**Scenario**: Nightly backup job chạy 30 phút

```bash
#!/bin/bash
# backup.sh

# Start time
start_time=$(date +%s)

# Run backup
backup_database

# Calculate duration
end_time=$(date +%s)
duration=$((end_time - start_time))

# Push metrics to Pushgateway
cat <<EOF | curl --data-binary @- \
  http://pushgateway:9091/metrics/job/backup_job/instance/$(hostname)
# TYPE backup_duration_seconds gauge
backup_duration_seconds $duration
# TYPE backup_last_success_timestamp gauge
backup_last_success_timestamp $end_time
# TYPE backup_size_bytes gauge
backup_size_bytes $(du -sb /backup | cut -f1)
EOF
```

**Alert on backup failure**:
```yaml
- alert: BackupJobFailed
  expr: time() - backup_last_success_timestamp > 86400
  for: 1h
  labels:
    severity: critical
  annotations:
    summary: "Backup job hasn't succeeded in 24 hours"
```

### 2. Cron Jobs

**Scenario**: Hourly data processing job

```python
#!/usr/bin/env python3
import time
import requests
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway

# Create registry
registry = CollectorRegistry()

# Define metrics
job_duration = Gauge('job_duration_seconds', 'Job duration', registry=registry)
records_processed = Gauge('records_processed_total', 'Records processed', registry=registry)
job_last_success = Gauge('job_last_success_timestamp', 'Last success timestamp', registry=registry)

# Run job
start = time.time()
try:
    # Process data
    count = process_data()
    
    # Record metrics
    job_duration.set(time.time() - start)
    records_processed.set(count)
    job_last_success.set(time.time())
    
    # Push to gateway
    push_to_gateway('pushgateway:9091', job='data_processing', registry=registry)
except Exception as e:
    print(f"Job failed: {e}")
    # Push failure metric
    job_last_success.set(0)
    push_to_gateway('pushgateway:9091', job='data_processing', registry=registry)
```

### 3. CI/CD Pipelines

**Scenario**: Track build và deployment metrics

```bash
#!/bin/bash
# deploy.sh

JOB_NAME="deployment"
INSTANCE="$(git rev-parse --short HEAD)"

# Start deployment
start_time=$(date +%s)

# Deploy application
if deploy_app; then
    status=1  # Success
else
    status=0  # Failure
fi

# Calculate duration
duration=$(($(date +%s) - start_time))

# Push metrics
cat <<EOF | curl --data-binary @- \
  http://pushgateway:9091/metrics/job/$JOB_NAME/instance/$INSTANCE
# TYPE deployment_duration_seconds gauge
deployment_duration_seconds $duration
# TYPE deployment_status gauge
deployment_status $status
# TYPE deployment_timestamp gauge
deployment_timestamp $(date +%s)
EOF
```

### 4. Scheduled Reports

**Scenario**: Daily report generation

```python
#!/usr/bin/env python3
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway
import time

registry = CollectorRegistry()

# Metrics
report_generation_time = Gauge('report_generation_seconds', 'Report generation time', registry=registry)
report_size = Gauge('report_size_bytes', 'Report size in bytes', registry=registry)
report_rows = Gauge('report_rows_total', 'Total rows in report', registry=registry)

# Generate report
start = time.time()
report = generate_daily_report()
duration = time.time() - start

# Record metrics
report_generation_time.set(duration)
report_size.set(len(report))
report_rows.set(report.count('\n'))

# Push to gateway
push_to_gateway('pushgateway:9091', job='daily_report', registry=registry)
```

## Cấu Hình

### Cài Đặt Pushgateway

```bash
# Download
wget https://github.com/prometheus/pushgateway/releases/download/v1.7.0/pushgateway-1.7.0.linux-amd64.tar.gz
tar xvfz pushgateway-*.tar.gz
cd pushgateway-*

# Run
./pushgateway
```

### Systemd Service

```ini
# /etc/systemd/system/pushgateway.service
[Unit]
Description=Prometheus Pushgateway
After=network.target

[Service]
Type=simple
User=prometheus
ExecStart=/usr/local/bin/pushgateway \
  --web.listen-address=:9091 \
  --web.telemetry-path=/metrics \
  --persistence.file=/var/lib/pushgateway/metrics.db \
  --persistence.interval=5m
Restart=always

[Install]
WantedBy=multi-user.target
```

### Prometheus Configuration

```yaml
scrape_configs:
  - job_name: 'pushgateway'
    honor_labels: true  # Preserve job labels from pushed metrics
    static_configs:
      - targets: ['pushgateway:9091']
```

**Lưu ý**: `honor_labels: true` quan trọng để preserve job labels từ pushed metrics.

### Client Libraries

**Python**:
```python
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway

registry = CollectorRegistry()
g = Gauge('job_last_success_unixtime', 'Last time job succeeded', registry=registry)
g.set_to_current_time()
push_to_gateway('pushgateway:9091', job='batch_job', registry=registry)
```

**Go**:
```go
package main

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/push"
)

func main() {
    completionTime := prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "job_completion_timestamp",
        Help: "Job completion timestamp",
    })
    completionTime.SetToCurrentTime()
    
    if err := push.New("http://pushgateway:9091", "batch_job").
        Collector(completionTime).
        Push(); err != nil {
        panic(err)
    }
}
```

**Shell Script**:
```bash
# Simple push
echo "some_metric 3.14" | curl --data-binary @- \
  http://pushgateway:9091/metrics/job/some_job

# With instance label
cat <<EOF | curl --data-binary @- \
  http://pushgateway:9091/metrics/job/some_job/instance/some_instance
some_metric{label="value"} 3.14
EOF
```

## Best Practices

### 1. Khi NÊN Sử Dụng Pushgateway

✅ **Short-lived jobs**:
- Batch jobs
- Cron jobs
- CI/CD pipelines
- Scheduled scripts

✅ **Jobs không có stable network identity**:
- Lambda functions
- Serverless functions
- One-off scripts

### 2. Khi KHÔNG NÊN Sử Dụng Pushgateway

❌ **Long-running services**:
- Web servers
- Databases
- Microservices

**Lý do**: 
- Mất khả năng detect service down (up metric luôn = 1)
- Không có automatic cleanup khi service down
- Tăng complexity không cần thiết

❌ **High-frequency metrics**:
- Metrics cần push mỗi giây
- Real-time monitoring

**Lý do**:
- Pushgateway không được thiết kế cho high throughput
- Có thể gây bottleneck

### 3. Metric Naming

Include timestamp metrics:

```bash
cat <<EOF | curl --data-binary @- \
  http://pushgateway:9091/metrics/job/backup_job
# TYPE backup_last_success_timestamp gauge
backup_last_success_timestamp $(date +%s)
# TYPE backup_duration_seconds gauge
backup_duration_seconds 123.45
EOF
```

Alert on stale metrics:

```yaml
- alert: BackupJobStale
  expr: time() - backup_last_success_timestamp > 86400
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Backup job hasn't run in 24 hours"
```

### 4. Error Handling

Always push metrics, even on failure:

```bash
#!/bin/bash

if run_job; then
    success=1
else
    success=0
fi

cat <<EOF | curl --data-binary @- \
  http://pushgateway:9091/metrics/job/my_job
# TYPE job_success gauge
job_success $success
# TYPE job_last_run_timestamp gauge
job_last_run_timestamp $(date +%s)
EOF
```

### 5. Cleanup

Delete metrics after job completion:

```bash
#!/bin/bash

# Push metrics
push_metrics

# Cleanup after 1 hour
sleep 3600
curl -X DELETE http://pushgateway:9091/metrics/job/my_job/instance/$(hostname)
```

### 6. High Availability

**Problem**: Pushgateway là single point of failure

**Solution 1**: Push to multiple Pushgateways

```bash
for gateway in pushgateway1:9091 pushgateway2:9091; do
    echo "some_metric 3.14" | curl --data-binary @- \
      http://$gateway/metrics/job/some_job
done
```

**Solution 2**: Use load balancer

```
┌──────────┐
│   Job    │
└────┬─────┘
     │
     ▼
┌────────────┐
│   LB       │
└─────┬──────┘
      │
      ├──────────┬──────────┐
      ▼          ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│Pushgateway│ │Pushgateway│ │Pushgateway│
│    1     │ │    2     │ │    3     │
└──────────┘ └──────────┘ └──────────┘
```

### 7. Monitoring Pushgateway

Monitor Pushgateway itself:

```yaml
- alert: PushgatewayDown
  expr: up{job="pushgateway"} == 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Pushgateway is down"

- alert: PushgatewayHighMemory
  expr: process_resident_memory_bytes{job="pushgateway"} > 1e9
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Pushgateway using > 1GB memory"
```

### 8. Persistence

Enable persistence để survive restarts:

```bash
./pushgateway \
  --persistence.file=/var/lib/pushgateway/metrics.db \
  --persistence.interval=5m
```

**Lưu ý**: Persistence không thay thế proper backup strategy.

## Limitations

### 1. Single Point of Failure

Pushgateway là stateful service. Nếu down, metrics bị mất.

**Mitigation**: 
- Enable persistence
- Run multiple instances
- Push to multiple gateways

### 2. Stale Metrics

Metrics không tự động expire khi job không chạy.

**Mitigation**:
- Include timestamp metrics
- Alert on stale metrics
- Implement cleanup logic

### 3. No Automatic Service Discovery

Không thể tự động discover jobs như với pull model.

**Mitigation**:
- Use consistent job naming
- Document all jobs pushing to gateway

### 4. Loss of Up Metric

Không thể detect khi job down (up metric luôn = 1 cho Pushgateway).

**Mitigation**:
- Use timestamp metrics
- Alert on stale timestamps

### 5. Cardinality Issues

Mỗi unique job/instance combination tạo new metric group.

**Mitigation**:
- Limit instance labels
- Implement cleanup
- Monitor Pushgateway memory

## Tài Liệu Liên Quan

- [Prometheus Server](./01-prometheus-server.md) - Cấu hình scraping
- [Exporters](./02-exporters.md) - Alternative cho long-running services
- [Instrumentation](../04-instrumentation/README.md) - Client libraries
- [Best Practices](../04-instrumentation/04-best-practices.md) - Metric design

## Tài Liệu Tham Khảo

- [Pushgateway Documentation](https://github.com/prometheus/pushgateway)
- [When to Use Pushgateway](https://prometheus.io/docs/practices/pushing/)
- [Pushgateway Best Practices](https://prometheus.io/docs/practices/instrumentation/#batch-jobs)
