# Alertmanager Configuration

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Kiến Trúc Alertmanager](#kiến-trúc-alertmanager)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Routing Tree](#routing-tree)
- [Grouping Alerts](#grouping-alerts)
- [Inhibition Rules](#inhibition-rules)
- [Silencing Alerts](#silencing-alerts)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)
- [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

## Tổng Quan

Alertmanager xử lý alerts được gửi từ Prometheus (và các client khác). Nó đảm nhận việc deduplication, grouping, routing, và gửi notifications đến các kênh phù hợp như email, Slack, PagerDuty.

**Chức năng chính của Alertmanager:**
- **Grouping**: Nhóm các alerts liên quan thành một notification
- **Inhibition**: Tắt alerts khi có alert khác đang firing
- **Silencing**: Tạm thời tắt notifications cho alerts cụ thể
- **Routing**: Điều hướng alerts đến đúng receiver
- **Deduplication**: Loại bỏ alerts trùng lặp

## Kiến Trúc Alertmanager

```
┌─────────────────────────────────────────────────────────────┐
│                      Alertmanager                           │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   Dispatcher │───▶│   Inhibitor  │───▶│   Silencer   │  │
│  │              │    │              │    │              │  │
│  │ Nhận alerts  │    │ Lọc alerts   │    │ Kiểm tra     │  │
│  │ từ Prometheus│    │ bị inhibit   │    │ silences     │  │
│  └──────────────┘    └──────────────┘    └──────┬───────┘  │
│                                                  │          │
│  ┌──────────────┐    ┌──────────────┐            │          │
│  │   Grouper    │◀───│    Router    │◀───────────┘          │
│  │              │    │              │                        │
│  │ Nhóm alerts  │    │ Routing tree │                        │
│  └──────┬───────┘    └──────────────┘                        │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │  Notifier    │                                           │
│  │              │                                           │
│  │ Gửi thông    │                                           │
│  │ báo          │                                           │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│  Receivers: Email | Slack | PagerDuty  │
└────────────────────────────────────────┘
```

### Các Thành Phần

| Thành Phần | Chức Năng |
|-----------|-----------|
| Dispatcher | Nhận alerts từ Prometheus, bắt đầu pipeline |
| Inhibitor | Kiểm tra inhibition rules, lọc alerts bị suppress |
| Silencer | Kiểm tra active silences, lọc alerts bị silence |
| Router | Áp dụng routing tree để tìm receiver phù hợp |
| Grouper | Nhóm alerts theo group_by labels |
| Notifier | Gửi notifications đến receivers |

## Cách Hoạt Động

### Pipeline Xử Lý Alert

1. **Nhận Alert**: Prometheus gửi alert đến `/api/v1/alerts`
2. **Deduplication**: Loại bỏ alerts trùng lặp (cùng fingerprint)
3. **Inhibition Check**: Kiểm tra xem alert có bị inhibit không
4. **Silence Check**: Kiểm tra xem alert có bị silence không
5. **Routing**: Tìm route phù hợp trong routing tree
6. **Grouping**: Nhóm alerts theo `group_by` labels
7. **Wait**: Chờ `group_wait` để thu thập thêm alerts
8. **Send**: Gửi notification đến receiver
9. **Repeat**: Gửi lại sau `repeat_interval` nếu alert vẫn firing

### Timing Parameters

```
Alert arrives
     │
     ▼
┌─────────────────────────────────────────────────────┐
│  group_wait (30s)                                   │
│  Chờ để thu thập alerts cùng group                  │
└─────────────────────────────────────────────────────┘
     │
     ▼ First notification sent
     │
     ▼
┌─────────────────────────────────────────────────────┐
│  group_interval (5m)                                │
│  Chờ trước khi gửi notification cho group mới       │
└─────────────────────────────────────────────────────┘
     │
     ▼ New alerts in group
     │
     ▼
┌─────────────────────────────────────────────────────┐
│  repeat_interval (4h)                               │
│  Gửi lại nếu alert vẫn firing                       │
└─────────────────────────────────────────────────────┘
```

## Routing Tree

### Cấu Trúc Routing Tree

Routing tree là cây quyết định để xác định receiver nào nhận alert:

```yaml
route:
  # Route gốc - catch-all
  receiver: default-receiver
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    # Route con 1: Critical alerts
    - match:
        severity: critical
      receiver: pagerduty-critical
      continue: false  # Dừng matching sau route này

    # Route con 2: Warning alerts cho team platform
    - match:
        severity: warning
        team: platform
      receiver: slack-platform
      group_by: ['alertname', 'instance']

    # Route con 3: Database alerts
    - match_re:
        alertname: "^Database.*"
      receiver: email-dba
      routes:
        # Route con lồng nhau
        - match:
            severity: critical
          receiver: pagerduty-dba
```

### Match Operators

```yaml
routes:
  # Exact match
  - match:
      severity: critical
      environment: production

  # Regex match
  - match_re:
      service: "^(api|web|worker)$"
      alertname: ".*High.*"

  # Kết hợp
  - match:
      severity: warning
    match_re:
      instance: "prod-.*"
```

### Continue Flag

```yaml
routes:
  # continue: true - tiếp tục matching các routes sau
  - match:
      severity: critical
    receiver: pagerduty
    continue: true  # Vẫn tiếp tục check routes khác

  # continue: false (mặc định) - dừng sau khi match
  - match:
      team: platform
    receiver: slack-platform
    continue: false
```

## Grouping Alerts

### Mục Đích Grouping

Grouping giúp tránh notification storm khi nhiều alerts cùng firing:

```
Không có grouping:
  Alert 1: server1 down → Email 1
  Alert 2: server2 down → Email 2
  Alert 3: server3 down → Email 3
  ... 50 emails!

Có grouping (group_by: [alertname]):
  Alerts 1-50: servers down → 1 Email tổng hợp
```

### Cấu Hình Grouping

```yaml
route:
  # Nhóm theo alertname và cluster
  group_by: ['alertname', 'cluster', 'service']

  # Chờ 30s để thu thập alerts cùng group
  group_wait: 30s

  # Chờ 5m trước khi gửi notification cho group mới
  group_interval: 5m

  # Gửi lại sau 4h nếu alert vẫn firing
  repeat_interval: 4h
```

### Group By Đặc Biệt

```yaml
# Nhóm tất cả alerts thành một notification
group_by: ['...']

# Không nhóm - mỗi alert là một notification riêng
group_by: []
```

## Inhibition Rules

### Mục Đích Inhibition

Inhibition tắt notifications cho alerts khi có alert khác đang firing. Ví dụ: khi server down, không cần gửi alert về các service trên server đó.

```
Không có inhibition:
  Alert: NodeDown (server1)
  Alert: ServiceDown (api trên server1)
  Alert: DatabaseDown (db trên server1)
  → 3 notifications!

Có inhibition:
  Alert: NodeDown (server1) → 1 notification
  Alert: ServiceDown (api) → bị inhibit
  Alert: DatabaseDown (db) → bị inhibit
  → 1 notification!
```

### Cấu Hình Inhibition

```yaml
inhibit_rules:
  # Khi NodeDown firing, inhibit các alerts khác trên cùng instance
  - source_match:
      alertname: NodeDown
      severity: critical
    target_match:
      severity: warning
    # Labels phải match giữa source và target
    equal: ['instance', 'cluster']

  # Khi có critical alert, inhibit warning alerts cùng service
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['alertname', 'service']

  # Khi cluster down, inhibit tất cả alerts trong cluster
  - source_match:
      alertname: ClusterDown
    target_match_re:
      alertname: ".*"
    equal: ['cluster']
```

### Ví Dụ Thực Tế

```yaml
inhibit_rules:
  # Database master down → inhibit replica alerts
  - source_match:
      alertname: DatabaseMasterDown
    target_match:
      alertname: DatabaseReplicaLag
    equal: ['datacenter']

  # Deployment in progress → inhibit latency alerts
  - source_match:
      alertname: DeploymentInProgress
    target_match:
      alertname: HighLatency
    equal: ['service', 'environment']
```

## Silencing Alerts

### Tạo Silence Qua UI

Truy cập Alertmanager UI tại `http://localhost:9093`:
1. Click "New Silence"
2. Thêm matchers (label=value)
3. Đặt thời gian bắt đầu và kết thúc
4. Thêm comment giải thích lý do
5. Click "Create"

### Tạo Silence Qua API

```bash
# Tạo silence
curl -X POST http://localhost:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {
        "name": "alertname",
        "value": "NodeDown",
        "isRegex": false
      },
      {
        "name": "instance",
        "value": "server1:9100",
        "isRegex": false
      }
    ],
    "startsAt": "2024-01-01T00:00:00Z",
    "endsAt": "2024-01-01T04:00:00Z",
    "createdBy": "admin",
    "comment": "Maintenance window for server1"
  }'

# Liệt kê silences
curl http://localhost:9093/api/v2/silences

# Xóa silence
curl -X DELETE http://localhost:9093/api/v2/silences/{silence_id}
```

### Tạo Silence Qua amtool

```bash
# Cài đặt amtool
go install github.com/prometheus/alertmanager/cmd/amtool@latest

# Tạo silence
amtool silence add \
  --alertmanager.url=http://localhost:9093 \
  --author="admin" \
  --comment="Maintenance" \
  --duration=4h \
  alertname=NodeDown instance=server1:9100

# Liệt kê silences
amtool silence query --alertmanager.url=http://localhost:9093

# Expire silence
amtool silence expire --alertmanager.url=http://localhost:9093 {silence_id}
```

## Use Cases

### Khi Nào Dùng Grouping

- Nhiều instances cùng loại có thể fail cùng lúc
- Alerts từ cùng một service nên được nhóm lại
- Giảm notification noise trong incident lớn

### Khi Nào Dùng Inhibition

- Server/node down → inhibit service alerts trên node đó
- Deployment đang chạy → inhibit performance alerts
- Maintenance mode → inhibit tất cả alerts cho component đó

### Khi Nào Dùng Silence

- Planned maintenance window
- Known issue đang được xử lý
- Testing/staging environment
- Tạm thời tắt noisy alert trong khi điều tra

## Cấu Hình

### Cấu Hình Alertmanager Đầy Đủ

```yaml
# alertmanager.yml
global:
  # Thời gian chờ khi resolve alert
  resolve_timeout: 5m

  # SMTP settings cho email
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'password'
  smtp_require_tls: true

  # Slack webhook URL mặc định
  slack_api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'

# Templates cho notifications
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routing tree
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    # Critical alerts → PagerDuty
    - match:
        severity: critical
      receiver: pagerduty-critical
      group_wait: 10s
      repeat_interval: 1h

    # Warning alerts → Slack
    - match:
        severity: warning
      receiver: slack-warnings
      group_wait: 1m

    # Database alerts → DBA team
    - match:
        team: dba
      receiver: email-dba
      routes:
        - match:
            severity: critical
          receiver: pagerduty-dba

# Inhibition rules
inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['alertname', 'cluster', 'service']

  - source_match:
      alertname: NodeDown
    target_match_re:
      alertname: ".*"
    equal: ['instance']

# Receivers
receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: '<pagerduty-integration-key>'
        description: '{{ template "pagerduty.default.description" . }}'

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warning'
        title: '{{ template "slack.default.title" . }}'
        text: '{{ template "slack.default.text" . }}'

  - name: 'email-dba'
    email_configs:
      - to: 'dba-team@example.com'
        send_resolved: true

  - name: 'pagerduty-dba'
    pagerduty_configs:
      - routing_key: '<dba-pagerduty-key>'
```

### Khởi Động Alertmanager

```bash
# Chạy Alertmanager
./alertmanager --config.file=alertmanager.yml

# Với Docker
docker run -d \
  -p 9093:9093 \
  -v /path/to/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
  prom/alertmanager

# Validate config
amtool check-config alertmanager.yml
```

### Reload Config

```bash
# Gửi SIGHUP
kill -HUP $(pidof alertmanager)

# Hoặc HTTP API
curl -X POST http://localhost:9093/-/reload
```

## Best Practices

### 1. Thiết Kế Routing Tree Rõ Ràng

```yaml
# Tổ chức theo: severity → team → service
route:
  routes:
    - match:
        severity: critical
      routes:
        - match:
            team: platform
          receiver: pagerduty-platform
        - match:
            team: backend
          receiver: pagerduty-backend
      receiver: pagerduty-default  # Fallback

    - match:
        severity: warning
      receiver: slack-warnings
```

### 2. Đặt Thời Gian Phù Hợp

```yaml
route:
  # Production: nhanh hơn
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    # Critical: rất nhanh
    - match:
        severity: critical
      group_wait: 10s
      repeat_interval: 1h

    # Info: chậm hơn, ít noise
    - match:
        severity: info
      group_wait: 5m
      repeat_interval: 24h
```

### 3. Luôn Có Default Receiver

```yaml
route:
  # Luôn có receiver mặc định để không miss alerts
  receiver: default-catch-all
  routes:
    # Specific routes...
```

### 4. Sử Dụng Templates

```yaml
templates:
  - '/etc/alertmanager/templates/*.tmpl'

receivers:
  - name: slack
    slack_configs:
      - channel: '#alerts'
        # Dùng template thay vì hardcode
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
```

### 5. High Availability

```yaml
# Chạy nhiều Alertmanager instances
# Chúng tự đồng bộ qua gossip protocol

# Instance 1
./alertmanager \
  --config.file=alertmanager.yml \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager2:9094

# Instance 2
./alertmanager \
  --config.file=alertmanager.yml \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager1:9094
```

## Tài Liệu Liên Quan

- [Alerting Rules](./01-alerting-rules.md) - Viết alerting rules trong Prometheus
- [Notification Channels](./03-notification-channels.md) - Cấu hình Email, Slack, PagerDuty
- [Alertmanager Component](../02-components/03-alertmanager.md) - Tổng quan kiến trúc
- [Lab 04 - Alertmanager](../07-labs/lab-04-alertmanager/README.md) - Bài tập thực hành

## Tài Liệu Tham Khảo

- [Alertmanager Configuration](https://prometheus.io/docs/alerting/latest/configuration/) - Official documentation
- [Alertmanager Routing](https://prometheus.io/docs/alerting/latest/alertmanager/) - Routing tree guide
- [amtool](https://github.com/prometheus/alertmanager#amtool) - CLI tool cho Alertmanager
- [Alertmanager API](https://petstore.swagger.io/?url=https://raw.githubusercontent.com/prometheus/alertmanager/main/api/v2/openapi.yaml) - API documentation
