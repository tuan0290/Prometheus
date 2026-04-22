# Storage and Retention

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Kiến Trúc](#kiến-trúc)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Tổng Quan

Prometheus sử dụng local time-series database (TSDB) để lưu trữ metrics data. Storage system được thiết kế để hiệu quả về cả performance và disk space, với khả năng nén dữ liệu xuống ~1.3 bytes per sample.

Các thành phần chính:
- **Local Storage**: On-disk TSDB với blocks và WAL
- **Remote Storage**: Tích hợp với external storage systems
- **Retention Policies**: Quản lý data lifecycle

## Kiến Trúc

```
┌─────────────────────────────────────────────────────────┐
│              Prometheus Server                          │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │              Memory (Head Block)                 │  │
│  │         Recent data (last 2 hours)               │  │
│  └────────────────────┬─────────────────────────────┘  │
│                       │                                │
│                       ▼                                │
│  ┌──────────────────────────────────────────────────┐  │
│  │          Write-Ahead Log (WAL)                   │  │
│  │         Durability for head block                │  │
│  └────────────────────┬─────────────────────────────┘  │
│                       │                                │
│                       ▼                                │
│  ┌──────────────────────────────────────────────────┐  │
│  │              Persistent Blocks                   │  │
│  │  ┌────────┐  ┌────────┐  ┌────────┐             │  │
│  │  │Block 1 │  │Block 2 │  │Block 3 │  ...        │  │
│  │  │(2h)    │  │(2h)    │  │(2h)    │             │  │
│  │  └────────┘  └────────┘  └────────┘             │  │
│  └──────────────────────────────────────────────────┘  │
│                       │                                │
└───────────────────────┼────────────────────────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │  Remote Storage  │
              │  (Optional)      │
              └──────────────────┘
```

### Storage Hierarchy

```
data/
├── wal/                          # Write-Ahead Log
│   ├── 00000000
│   ├── 00000001
│   └── checkpoint.00000002/
├── 01ABCDEFGHIJKLMNOP/          # Block 1 (2h)
│   ├── chunks/
│   │   └── 000001
│   ├── index
│   ├── meta.json
│   └── tombstones
├── 01QRSTUVWXYZ123456/          # Block 2 (2h)
│   ├── chunks/
│   ├── index
│   ├── meta.json
│   └── tombstones
└── 01ZYXWVUTSRQPONMLK/          # Block 3 (compacted)
    ├── chunks/
    ├── index
    ├── meta.json
    └── tombstones
```

## Cách Hoạt Động

### 1. Write Path

```
1. Scrape metrics
   │
   ▼
2. Write to Head Block (in-memory)
   │
   ▼
3. Write to WAL (on-disk, for durability)
   │
   ▼
4. After 2 hours, persist Head Block to disk
   │
   ▼
5. Create new Head Block
   │
   ▼
6. Compact old blocks
```

### 2. Head Block

Head block lưu trữ recent data (mặc định 2 giờ) trong memory:

- **In-memory**: Fast writes và reads
- **WAL backup**: Durability nếu crash
- **Periodic persistence**: Flush to disk mỗi 2 giờ

### 3. Persistent Blocks

Blocks được lưu trữ trên disk:

**Block structure**:
```
01ABCDEFGHIJKLMNOP/
├── chunks/           # Compressed time-series data
│   └── 000001       # Chunk file
├── index            # Inverted index for fast lookups
├── meta.json        # Block metadata
└── tombstones       # Deleted series markers
```

**meta.json**:
```json
{
  "ulid": "01ABCDEFGHIJKLMNOP",
  "minTime": 1609459200000,
  "maxTime": 1609466400000,
  "stats": {
    "numSamples": 1000000,
    "numSeries": 10000,
    "numChunks": 50000
  },
  "compaction": {
    "level": 1,
    "sources": ["01ABCDEFGHIJKLMNOP"]
  }
}
```

### 4. Compaction

Compaction merge multiple blocks thành blocks lớn hơn:

```
Level 1: 2h blocks
┌────┐ ┌────┐ ┌────┐
│ 2h │ │ 2h │ │ 2h │
└────┘ └────┘ └────┘
       │
       ▼ Compact
Level 2: 6h block
┌──────────────┐
│      6h      │
└──────────────┘
       │
       ▼ Compact
Level 3: 18h block
┌────────────────────────┐
│         18h            │
└────────────────────────┘
```

**Benefits**:
- Reduce number of blocks
- Improve query performance
- Better compression ratio

### 5. Retention

Data được delete khi vượt quá retention period:

```
Timeline:
├─────────┼─────────┼─────────┼─────────┼─────────┤
0h       2h        4h        6h        8h       10h
│                                              │
└──────────────────────────────────────────────┘
              Retention: 10h
                                               │
                                               ▼
                                         Delete blocks
                                         older than 10h
```

### 6. Write-Ahead Log (WAL)

WAL đảm bảo durability cho head block:

**Write process**:
1. Write sample to WAL
2. Write sample to head block (memory)
3. Acknowledge write

**Recovery process**:
1. Read WAL on startup
2. Replay samples to rebuild head block
3. Continue normal operation

**WAL segments**:
```
wal/
├── 00000000          # Segment 1 (128MB)
├── 00000001          # Segment 2 (128MB)
├── 00000002          # Segment 3 (active)
└── checkpoint.00000001/  # Checkpoint (can delete 00000000)
```

## Use Cases

### 1. Short-term Retention (Local Storage)

**Scenario**: Monitor production với 15 days retention

> **Prometheus 3.x**: `--storage.tsdb.retention.time` và `--storage.tsdb.retention.size` đã deprecated dưới dạng CLI flags. Nên cấu hình trong `prometheus.yml`:
> ```yaml
> storage:
>   tsdb:
>     retention:
>       time: 15d
>       size: 50GB
> ```

```bash
# Vẫn hoạt động nhưng deprecated - dùng config file thay thế
prometheus \
  --storage.tsdb.path=/var/lib/prometheus/data \
  --storage.tsdb.retention.time=15d \
  --storage.tsdb.retention.size=50GB
```

**Disk usage estimate**:
```
Samples per second: 100,000
Retention: 15 days
Bytes per sample: 1.3

Disk = 100,000 * 86,400 * 15 * 1.3 bytes
     = ~169 GB
```

### 2. Long-term Storage (Remote Write)

**Scenario**: Keep metrics for 1 year

```yaml
# prometheus.yml
remote_write:
  - url: "https://remote-storage.example.com/api/v1/write"
    queue_config:
      capacity: 10000
      max_shards: 5
      max_samples_per_send: 1000
      batch_send_deadline: 5s
    write_relabel_configs:
      # Only send important metrics
      - source_labels: [__name__]
        regex: 'important_.*'
        action: keep
```

### 3. Multi-tenancy Storage

**Scenario**: Separate storage per tenant

```yaml
# Tenant 1
remote_write:
  - url: "https://storage.example.com/api/v1/write"
    headers:
      X-Tenant-ID: tenant1
    write_relabel_configs:
      - source_labels: [tenant]
        regex: tenant1
        action: keep

# Tenant 2
remote_write:
  - url: "https://storage.example.com/api/v1/write"
    headers:
      X-Tenant-ID: tenant2
    write_relabel_configs:
      - source_labels: [tenant]
        regex: tenant2
        action: keep
```

### 4. Disaster Recovery

**Scenario**: Backup Prometheus data

```bash
# Create snapshot
curl -XPOST http://localhost:9090/api/v1/admin/tsdb/snapshot

# Snapshot created at: /var/lib/prometheus/data/snapshots/20240101T120000Z-abc123

# Backup snapshot
tar -czf prometheus-backup-$(date +%Y%m%d).tar.gz \
  /var/lib/prometheus/data/snapshots/20240101T120000Z-abc123

# Upload to S3
aws s3 cp prometheus-backup-20240101.tar.gz \
  s3://backups/prometheus/
```

## Cấu Hình

### Local Storage Configuration

```bash
prometheus \
  --storage.tsdb.path=/var/lib/prometheus/data \
  --storage.tsdb.retention.time=15d \
  --storage.tsdb.retention.size=50GB \
  --storage.tsdb.wal-compression \
  --storage.tsdb.min-block-duration=2h \
  --storage.tsdb.max-block-duration=36h
```

**Parameters**:
- `storage.tsdb.path`: Data directory
- `storage.tsdb.retention.time`: Time-based retention
- `storage.tsdb.retention.size`: Size-based retention
- `storage.tsdb.wal-compression`: Enable WAL compression
- `storage.tsdb.min-block-duration`: Minimum block duration
- `storage.tsdb.max-block-duration`: Maximum block duration

### Remote Write Configuration

```yaml
remote_write:
  - url: "https://remote-storage.example.com/api/v1/write"
    
    # Authentication
    basic_auth:
      username: prometheus
      password: secret
    
    # TLS
    tls_config:
      ca_file: /etc/prometheus/ca.crt
      cert_file: /etc/prometheus/client.crt
      key_file: /etc/prometheus/client.key
    
    # Queue configuration
    queue_config:
      capacity: 10000              # Queue capacity
      max_shards: 5                # Max parallel shards
      min_shards: 1                # Min parallel shards
      max_samples_per_send: 1000   # Samples per request
      batch_send_deadline: 5s      # Max wait time
      min_backoff: 30ms            # Min retry backoff
      max_backoff: 100ms           # Max retry backoff
    
    # Metadata
    metadata_config:
      send: true
      send_interval: 1m
    
    # Relabeling
    write_relabel_configs:
      # Drop expensive metrics
      - source_labels: [__name__]
        regex: 'go_.*'
        action: drop
      
      # Add remote label
      - target_label: remote
        replacement: 'true'
```

### Remote Read Configuration

```yaml
remote_read:
  - url: "https://remote-storage.example.com/api/v1/read"
    
    # Authentication
    basic_auth:
      username: prometheus
      password: secret
    
    # Read recent data from local, old data from remote
    read_recent: true
    
    # Required matchers (only read specific metrics)
    required_matchers:
      job: "important-job"
```

### Storage Backends

**Cortex**:
```yaml
remote_write:
  - url: "http://cortex:9009/api/prom/push"
```

**Thanos**:
```yaml
# Use Thanos Sidecar instead of remote_write
# Sidecar uploads blocks to object storage
```

**VictoriaMetrics**:
```yaml
remote_write:
  - url: "http://victoriametrics:8428/api/v1/write"
```

**InfluxDB**:
```yaml
remote_write:
  - url: "http://influxdb:8086/api/v1/prom/write?db=prometheus"
```

**M3DB**:
```yaml
remote_write:
  - url: "http://m3coordinator:7201/api/v1/prom/remote/write"
```

## Best Practices

### 1. Capacity Planning

**Calculate storage requirements**:

```
Variables:
- Samples per second (S)
- Retention time in seconds (R)
- Bytes per sample (B) = ~1.3

Disk space = S × R × B

Example:
- 100,000 samples/sec
- 15 days retention (1,296,000 seconds)
- 1.3 bytes/sample

Disk = 100,000 × 1,296,000 × 1.3
     = ~169 GB
```

**Add buffer**:
```
Recommended disk = Calculated × 1.5
                 = 169 GB × 1.5
                 = ~254 GB
```

### 2. Retention Strategy

**Time-based retention**:
```bash
# Keep 15 days
--storage.tsdb.retention.time=15d
```

**Size-based retention**:
```bash
# Keep max 50GB
--storage.tsdb.retention.size=50GB
```

**Combined**:
```bash
# Keep 15 days OR 50GB (whichever comes first)
--storage.tsdb.retention.time=15d \
--storage.tsdb.retention.size=50GB
```

### 3. WAL Compression

Enable WAL compression để save disk space:

```bash
--storage.tsdb.wal-compression
```

**Benefits**:
- Reduce WAL disk usage by ~50%
- Minimal CPU overhead
- Faster recovery (less data to read)

### 4. Block Duration

Adjust block duration for your use case:

```bash
# Default: 2h min, 36h max
--storage.tsdb.min-block-duration=2h \
--storage.tsdb.max-block-duration=36h

# Long retention: Increase max duration
--storage.tsdb.max-block-duration=72h
```

### 5. Remote Storage Strategy

**Tiered storage**:
```
Recent data (7 days):  Local storage (fast queries)
Old data (1 year):     Remote storage (long-term retention)
```

**Configuration**:
```yaml
# Local retention: 7 days
--storage.tsdb.retention.time=7d

# Remote write: All data
remote_write:
  - url: "https://remote-storage.example.com/api/v1/write"

# Remote read: For queries beyond 7 days
remote_read:
  - url: "https://remote-storage.example.com/api/v1/read"
    read_recent: true
```

### 6. Monitoring Storage

Monitor storage health:

```yaml
# Disk space
- alert: PrometheusDiskSpaceLow
  expr: |
    (
      node_filesystem_avail_bytes{mountpoint="/var/lib/prometheus"}
      /
      node_filesystem_size_bytes{mountpoint="/var/lib/prometheus"}
    ) < 0.1
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Prometheus disk space < 10%"

# WAL corruptions
- alert: PrometheusWALCorruptions
  expr: rate(prometheus_tsdb_wal_corruptions_total[5m]) > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Prometheus WAL corruptions detected"

# Compaction failures
- alert: PrometheusCompactionsFailing
  expr: rate(prometheus_tsdb_compactions_failed_total[5m]) > 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Prometheus compactions failing"

# Remote write lag
- alert: PrometheusRemoteWriteLag
  expr: |
    (
      prometheus_remote_storage_highest_timestamp_in_seconds
      - ignoring(remote_name, url) group_right
      prometheus_remote_storage_queue_highest_sent_timestamp_seconds
    ) > 300
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Remote write lag > 5 minutes"
```

### 7. Backup Strategy

**Snapshot-based backup**:
```bash
#!/bin/bash
# backup.sh

# Create snapshot
SNAPSHOT=$(curl -XPOST http://localhost:9090/api/v1/admin/tsdb/snapshot | jq -r .data.name)

# Backup snapshot
tar -czf prometheus-backup-$(date +%Y%m%d).tar.gz \
  /var/lib/prometheus/data/snapshots/$SNAPSHOT

# Upload to S3
aws s3 cp prometheus-backup-$(date +%Y%m%d).tar.gz \
  s3://backups/prometheus/

# Cleanup old snapshots
find /var/lib/prometheus/data/snapshots/ -mtime +7 -delete
```

**Continuous backup with Thanos**:
```yaml
# Thanos Sidecar continuously uploads blocks to object storage
# No manual backup needed
```

### 8. Performance Tuning

**Increase memory for head block**:
```bash
# Default: 2h head block
# Increase for high cardinality
--storage.tsdb.min-block-duration=4h
```

**Tune compaction**:
```bash
# Disable compaction during peak hours
# Use external compactor (Thanos Compactor)
```

**Optimize queries**:
```yaml
# Limit query time range
--query.max-samples=50000000
--query.timeout=2m
```

## Tài Liệu Liên Quan

- [Prometheus Server](./01-prometheus-server.md) - Server configuration
- [Prometheus Architecture](../01-fundamentals/04-prometheus-architecture.md) - Architecture overview
- [Best Practices](../04-instrumentation/04-best-practices.md) - Metric design
- [Thanos Integration](../06-integrations/README.md) - Long-term storage

## Tài Liệu Tham Khảo

- [Prometheus Storage](https://prometheus.io/docs/prometheus/latest/storage/)
- [Remote Write Specification](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write)
- [TSDB Format](https://github.com/prometheus/prometheus/blob/main/tsdb/docs/format/README.md)
