# Time-Series Databases

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Time-Series Data là gì?](#time-series-data-là-gì)
- [Đặc Điểm của Time-Series Data](#đặc-điểm-của-time-series-data)
- [Time-Series Database (TSDB)](#time-series-database-tsdb)
- [Tại Sao Cần TSDB?](#tại-sao-cần-tsdb)
- [Cách TSDB Hoạt Động](#cách-tsdb-hoạt-động)
- [Prometheus TSDB](#prometheus-tsdb)
- [So Sánh TSDB với Database Truyền Thống](#so-sánh-tsdb-với-database-truyền-thống)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Giới Thiệu

**Time-Series Database** (TSDB) là loại cơ sở dữ liệu được tối ưu hóa đặc biệt để lưu trữ và truy vấn dữ liệu theo chuỗi thời gian. Prometheus sử dụng TSDB riêng để lưu trữ metrics một cách hiệu quả.

## Time-Series Data là gì?

**Time-series data** (dữ liệu chuỗi thời gian) là dữ liệu được thu thập theo thời gian, mỗi điểm dữ liệu có một timestamp (dấu thời gian).

### Ví Dụ Thực Tế

Giống như việc đo nhiệt độ phòng mỗi phút:

```
Thời gian          Nhiệt độ
─────────────────────────────
10:00:00           22.5°C
10:01:00           22.6°C
10:02:00           22.7°C
10:03:00           22.8°C
10:04:00           22.9°C
```

Khi vẽ thành biểu đồ:

```
Nhiệt độ (°C)
24.0 ┤
23.5 ┤
23.0 ┤                 ╭─────
22.5 ┤         ╭───────╯
22.0 ┤─────────╯
     └─────────────────────────
     10:00  10:02  10:04  10:06
```

### Ví Dụ trong Monitoring

Metrics trong monitoring đều là time-series data:

```
Metric: http_requests_total
─────────────────────────────────────────
Timestamp            Value    Labels
2025-01-15 10:00:00  1000     {method="GET", path="/api"}
2025-01-15 10:01:00  1050     {method="GET", path="/api"}
2025-01-15 10:02:00  1120     {method="GET", path="/api"}
2025-01-15 10:03:00  1180     {method="GET", path="/api"}
```

## Đặc Điểm của Time-Series Data

### 1. Write-Heavy (Ghi Nhiều)

Dữ liệu được ghi liên tục với tần suất cao.

```
┌─────────────────────────────────────┐
│  Typical Workload Pattern           │
├─────────────────────────────────────┤
│  Writes: ████████████████ 95%       │
│  Reads:  ███░░░░░░░░░░░░  5%        │
└─────────────────────────────────────┘
```

**Ví dụ**: Prometheus scrape metrics mỗi 15 giây từ 100 targets = 400 writes/phút

### 2. Immutable (Không Thay Đổi)

Dữ liệu lịch sử không bao giờ được cập nhật, chỉ thêm mới.

```
✓ Thêm điểm dữ liệu mới:
  10:05:00 → 23.0°C

✗ KHÔNG cập nhật dữ liệu cũ:
  10:03:00: 22.8°C → 22.9°C (không làm điều này)
```

### 3. Time-Ordered (Sắp Xếp Theo Thời Gian)

Dữ liệu luôn được sắp xếp theo timestamp.

```
Timestamp        Value
─────────────────────────
10:00:00         100
10:01:00         105
10:02:00         110  ← Luôn tăng dần
10:03:00         115
```

### 4. Append-Only (Chỉ Thêm Vào Cuối)

Dữ liệu mới luôn được thêm vào cuối chuỗi.

```
Existing data:  [10:00] [10:01] [10:02]
                                    ↓
New data:       [10:00] [10:01] [10:02] [10:03] ← Append
```

### 5. High Cardinality (Số Lượng Lớn)

Có thể có hàng triệu time-series khác nhau.

```
http_requests_total{method="GET", path="/api", status="200"}
http_requests_total{method="GET", path="/api", status="404"}
http_requests_total{method="POST", path="/api", status="200"}
http_requests_total{method="POST", path="/users", status="201"}
...
(hàng nghìn combinations)
```

## Time-Series Database (TSDB)

TSDB là cơ sở dữ liệu được thiết kế đặc biệt cho time-series data.

### Các TSDB Phổ Biến

```
┌──────────────────────────────────────┐
│  Popular Time-Series Databases       │
├──────────────────────────────────────┤
│  • Prometheus (monitoring)           │
│  • InfluxDB (general purpose)        │
│  • TimescaleDB (PostgreSQL extension)│
│  • OpenTSDB (Hadoop-based)           │
│  • Graphite (metrics)                │
└──────────────────────────────────────┘
```

### Kiến Trúc TSDB

```
┌─────────────────────────────────────────────┐
│           TIME-SERIES DATABASE              │
├─────────────────────────────────────────────┤
│                                             │
│  ┌─────────────┐      ┌─────────────┐      │
│  │   Write     │      │    Read     │      │
│  │   Path      │      │    Path     │      │
│  └──────┬──────┘      └──────┬──────┘      │
│         ↓                    ↓              │
│  ┌─────────────────────────────────┐       │
│  │      In-Memory Buffer           │       │
│  │  (Recent data for fast access)  │       │
│  └──────────────┬──────────────────┘       │
│                 ↓                           │
│  ┌─────────────────────────────────┐       │
│  │      Compression Engine         │       │
│  │  (Reduce storage size)          │       │
│  └──────────────┬──────────────────┘       │
│                 ↓                           │
│  ┌─────────────────────────────────┐       │
│  │      Disk Storage               │       │
│  │  (Long-term persistent storage) │       │
│  └─────────────────────────────────┘       │
│                                             │
└─────────────────────────────────────────────┘
```

## Tại Sao Cần TSDB?

### 1. Hiệu Suất Cao

TSDB được tối ưu cho workload time-series.

**So sánh với SQL Database**:

```
Query: "Lấy CPU usage trong 1 giờ qua"

SQL Database (MySQL):
  SELECT timestamp, value 
  FROM metrics 
  WHERE name='cpu_usage' 
    AND timestamp > NOW() - INTERVAL 1 HOUR
  
  Performance: ~500ms (phải scan nhiều rows)

TSDB (Prometheus):
  cpu_usage[1h]
  
  Performance: ~50ms (optimized for time-range queries)
```

### 2. Nén Dữ Liệu Hiệu Quả

TSDB sử dụng compression algorithms đặc biệt.

```
Raw data size:     1 GB
After compression: 100 MB (10x reduction)
```

**Kỹ thuật nén**:
- **Delta encoding**: Lưu sự thay đổi thay vì giá trị tuyệt đối
- **Run-length encoding**: Nén các giá trị lặp lại
- **Gorilla compression**: Thuật toán của Facebook cho time-series

### 3. Retention Policies (Chính Sách Lưu Trữ)

Tự động xóa dữ liệu cũ để tiết kiệm không gian.

```
┌─────────────────────────────────────┐
│  Retention Policy Example           │
├─────────────────────────────────────┤
│  Recent data (1 day):               │
│    Resolution: 15s                  │
│    Storage: 10 GB                   │
│         ↓                           │
│  Medium-term (7 days):              │
│    Resolution: 1m (downsampled)     │
│    Storage: 5 GB                    │
│         ↓                           │
│  Long-term (30 days):               │
│    Resolution: 5m (downsampled)     │
│    Storage: 3 GB                    │
│         ↓                           │
│  Older data: DELETED                │
└─────────────────────────────────────┘
```

### 4. Truy Vấn Theo Thời Gian Nhanh

TSDB có index đặc biệt cho time-range queries.

```
Query patterns được tối ưu:
✓ Lấy data trong khoảng thời gian
✓ Aggregation theo thời gian (sum, avg, max)
✓ Downsampling (giảm resolution)
✓ Rate calculations (tính tốc độ thay đổi)
```

## Cách TSDB Hoạt Động

### 1. Data Model

TSDB tổ chức data theo **series** (chuỗi).

```
Series = Metric Name + Labels

Example:
  http_requests_total{method="GET", path="/api", status="200"}
  
  Metric Name: http_requests_total
  Labels: {method="GET", path="/api", status="200"}
```

Mỗi series có một chuỗi các data points:

```
Series: http_requests_total{method="GET"}
─────────────────────────────────────────
Timestamp            Value
2025-01-15 10:00:00  1000
2025-01-15 10:01:00  1050
2025-01-15 10:02:00  1120
2025-01-15 10:03:00  1180
```

### 2. Storage Layout

TSDB chia data thành các **blocks** theo thời gian.

```
┌─────────────────────────────────────────┐
│  TSDB Storage Structure                 │
├─────────────────────────────────────────┤
│                                         │
│  Block 1: 00:00 - 02:00                 │
│  ├── index (series metadata)            │
│  ├── chunks (compressed data)           │
│  └── tombstones (deleted data)          │
│                                         │
│  Block 2: 02:00 - 04:00                 │
│  ├── index                              │
│  ├── chunks                             │
│  └── tombstones                         │
│                                         │
│  Block 3: 04:00 - 06:00                 │
│  ├── index                              │
│  ├── chunks                             │
│  └── tombstones                         │
│                                         │
└─────────────────────────────────────────┘
```

### 3. Write Path

Quá trình ghi dữ liệu:

```
┌──────────────┐
│  New Sample  │  cpu_usage{host="server1"} = 75.5
└──────┬───────┘
       ↓
┌──────────────┐
│  WAL         │  Write-Ahead Log (crash recovery)
│  (Disk)      │
└──────┬───────┘
       ↓
┌──────────────┐
│  Memory      │  In-memory buffer (fast access)
│  Buffer      │
└──────┬───────┘
       ↓
┌──────────────┐
│  Compress    │  Compress and write to disk
│  & Persist   │  (every 2 hours)
└──────────────┘
```

### 4. Read Path

Quá trình đọc dữ liệu:

```
Query: cpu_usage[1h]
       ↓
┌──────────────┐
│  Parse Query │  Xác định time range và labels
└──────┬───────┘
       ↓
┌──────────────┐
│  Find Series │  Tìm matching series trong index
└──────┬───────┘
       ↓
┌──────────────┐
│  Read Chunks │  Đọc compressed chunks từ disk
└──────┬───────┘
       ↓
┌──────────────┐
│  Decompress  │  Giải nén data
└──────┬───────┘
       ↓
┌──────────────┐
│  Return Data │  Trả về kết quả
└──────────────┘
```

## Prometheus TSDB

Prometheus có TSDB riêng được tối ưu cho monitoring.

### Đặc Điểm

```
┌─────────────────────────────────────────┐
│  Prometheus TSDB Features               │
├─────────────────────────────────────────┤
│  • Local storage (on-disk)              │
│  • Efficient compression (Gorilla)      │
│  • Fast queries (inverted index)        │
│  • Configurable retention               │
│  • No external dependencies             │
│  • Write-Ahead Log (WAL) for durability │
└─────────────────────────────────────────┘
```

### Storage Structure

```
prometheus/
└── data/
    ├── wal/                    # Write-Ahead Log
    │   ├── 00000001
    │   └── 00000002
    ├── 01HQXXX/                # Block (2h window)
    │   ├── chunks/
    │   │   └── 000001
    │   ├── index
    │   ├── meta.json
    │   └── tombstones
    ├── 01HQYYY/                # Another block
    │   ├── chunks/
    │   ├── index
    │   └── meta.json
    └── lock                    # Lock file
```

### Compression Example

```
Original samples (uncompressed):
─────────────────────────────────
Timestamp            Value
1610000000           100
1610000015           102
1610000030           105
1610000045           103
1610000060           107

Size: 80 bytes (16 bytes per sample)

After Prometheus compression:
─────────────────────────────────
Base timestamp: 1610000000
Delta encoding: [0, +15, +15, +15, +15]
Base value: 100
Delta encoding: [0, +2, +3, -2, +4]

Size: ~12 bytes (85% reduction!)
```

## So Sánh TSDB với Database Truyền Thống

### Relational Database (MySQL, PostgreSQL)

```
┌─────────────────────────────────────┐
│  Relational Database                │
├─────────────────────────────────────┤
│  Strengths:                         │
│  • ACID transactions                │
│  • Complex joins                    │
│  • Data updates/deletes             │
│  • Flexible schema                  │
│                                     │
│  Weaknesses for time-series:        │
│  • Slow for time-range queries      │
│  • Poor compression                 │
│  • High storage cost                │
│  • Complex indexing needed          │
└─────────────────────────────────────┘
```

### Time-Series Database

```
┌─────────────────────────────────────┐
│  Time-Series Database               │
├─────────────────────────────────────┤
│  Strengths:                         │
│  • Fast time-range queries          │
│  • Excellent compression            │
│  • Optimized for append-only        │
│  • Built-in retention policies      │
│                                     │
│  Weaknesses:                        │
│  • No updates/deletes               │
│  • Limited join capabilities        │
│  • Specialized use case             │
└─────────────────────────────────────┘
```

### Performance Comparison

```
Scenario: Store 1 million metrics/second for 30 days

Relational Database:
  Storage: ~5 TB (uncompressed)
  Query time: 5-10 seconds
  Cost: High (SSD required)

Time-Series Database:
  Storage: ~500 GB (10x compression)
  Query time: 50-100 milliseconds
  Cost: Medium (HDD acceptable)
```

## Tài Liệu Liên Quan

- [Monitoring and Observability](./01-monitoring-observability.md) - Tại sao cần time-series data
- [Metrics Types](./03-metrics-types.md) - Các loại metrics được lưu trong TSDB
- [Prometheus Architecture](./04-prometheus-architecture.md) - Cách Prometheus sử dụng TSDB

## Tài Liệu Tham Khảo

- [Prometheus Storage Documentation](https://prometheus.io/docs/prometheus/latest/storage/)
- [Facebook Gorilla Paper](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf)
- [Time-Series Database Comparison](https://db-engines.com/en/ranking/time+series+dbms)
