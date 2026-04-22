# Service Discovery

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Kiến Trúc](#kiến-trúc)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Tổng Quan

Service Discovery là cơ chế tự động phát hiện và cập nhật danh sách targets để Prometheus scrape. Thay vì cấu hình static targets, service discovery cho phép Prometheus tự động adapt với dynamic infrastructure.

Prometheus hỗ trợ nhiều service discovery mechanisms:
- **Static Config**: Cấu hình thủ công (không phải SD thực sự)
- **File-based SD**: Đọc targets từ files
- **Kubernetes SD**: Tự động discover pods, services, endpoints
- **Consul SD**: Discover services từ Consul
- **EC2 SD**: Discover EC2 instances
- **DNS SD**: Discover targets qua DNS records

## Kiến Trúc

```
┌─────────────────────────────────────────────────────────┐
│              Prometheus Server                          │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │         Service Discovery Manager                │  │
│  │                                                  │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐      │  │
│  │  │   File   │  │Kubernetes│  │  Consul  │      │  │
│  │  │    SD    │  │    SD    │  │    SD    │ ...  │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘      │  │
│  └───────┼─────────────┼─────────────┼─────────────┘  │
│          │             │             │                │
│          ▼             ▼             ▼                │
│  ┌──────────────────────────────────────────────────┐  │
│  │           Target Manager                         │  │
│  │  (Apply relabeling, filtering)                   │  │
│  └──────────────────────────────────────────────────┘  │
│          │                                            │
└──────────┼────────────────────────────────────────────┘
           │
           ▼
    ┌──────────────┐
    │   Targets    │
    │  (Scraping)  │
    └──────────────┘
```

### Discovery Flow

```
1. Service Discovery
   │
   ▼
2. Generate Target List
   │
   ▼
3. Apply Relabeling
   │
   ▼
4. Filter Targets
   │
   ▼
5. Scrape Active Targets
```

## Cách Hoạt Động

### Meta Labels

Service discovery tạo meta labels (prefix `__meta_`) chứa thông tin về targets. Meta labels được sử dụng trong relabeling và bị drop trước khi scrape.

**Example Kubernetes meta labels**:
```
__meta_kubernetes_pod_name="my-app-abc123"
__meta_kubernetes_namespace="production"
__meta_kubernetes_pod_label_app="my-app"
__meta_kubernetes_pod_annotation_prometheus_io_scrape="true"
```

### Relabeling

Relabeling transform meta labels thành final labels:

```yaml
relabel_configs:
  # Keep only pods with annotation prometheus.io/scrape=true
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  
  # Use custom port from annotation
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
    action: replace
    target_label: __address__
    regex: (.+)
    replacement: $1:${1}
  
  # Add namespace label
  - source_labels: [__meta_kubernetes_namespace]
    target_label: namespace
```

## Use Cases

### 1. Static Configuration

**Scenario**: Fixed list of targets

```yaml
scrape_configs:
  - job_name: 'static-nodes'
    static_configs:
      - targets:
          - 'server1:9100'
          - 'server2:9100'
          - 'server3:9100'
        labels:
          env: 'production'
          region: 'us-east-1'
      
      - targets:
          - 'server4:9100'
          - 'server5:9100'
        labels:
          env: 'staging'
          region: 'us-west-1'
```

**Pros**:
- Simple và straightforward
- Không cần external dependencies

**Cons**:
- Phải update config khi thêm/xóa targets
- Không phù hợp cho dynamic environments

### 2. File-based Service Discovery

**Scenario**: Dynamic targets được generate bởi external system

**File format** (`targets.json`):
```json
[
  {
    "targets": ["server1:9100", "server2:9100"],
    "labels": {
      "env": "production",
      "job": "node"
    }
  },
  {
    "targets": ["server3:9100"],
    "labels": {
      "env": "staging",
      "job": "node"
    }
  }
]
```

**Prometheus config**:
```yaml
scrape_configs:
  - job_name: 'file-sd'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/*.json'
          - '/etc/prometheus/targets/*.yml'
        refresh_interval: 30s
```

**Generate targets script**:
```bash
#!/bin/bash
# generate-targets.sh

# Query infrastructure API
curl -s https://api.example.com/servers | jq '[
  {
    "targets": [.[] | "\(.ip):9100"],
    "labels": {
      "env": "production",
      "region": .region
    }
  }
]' > /etc/prometheus/targets/servers.json
```

### 3. Kubernetes Service Discovery

**Scenario**: Monitor Kubernetes cluster

#### Pod Discovery

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    
    relabel_configs:
      # Only scrape pods with annotation prometheus.io/scrape=true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      
      # Use custom path from annotation (default: /metrics)
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      
      # Use custom port from annotation (default: pod port)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      
      # Add namespace label
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      
      # Add pod name label
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      
      # Add app label from pod label
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
```

**Pod annotations**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
  labels:
    app: my-app
spec:
  containers:
    - name: app
      image: my-app:latest
      ports:
        - containerPort: 8080
```

#### Service Discovery

```yaml
scrape_configs:
  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
      - role: service
    
    relabel_configs:
      # Only scrape services with annotation prometheus.io/scrape=true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      
      # Use custom port
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: $1:${1}
      
      # Add service name
      - source_labels: [__meta_kubernetes_service_name]
        target_label: service
```

#### Endpoints Discovery

```yaml
scrape_configs:
  - job_name: 'kubernetes-endpoints'
    kubernetes_sd_configs:
      - role: endpoints
    
    relabel_configs:
      # Only scrape endpoints with service annotation
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      
      # Add namespace
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      
      # Add service name
      - source_labels: [__meta_kubernetes_service_name]
        target_label: service
      
      # Add pod name
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
```

### 4. Consul Service Discovery

**Scenario**: Services registered in Consul

```yaml
scrape_configs:
  - job_name: 'consul-services'
    consul_sd_configs:
      - server: 'consul.example.com:8500'
        datacenter: 'dc1'
        services: []  # Empty = all services
        tags:
          - 'prometheus'
        refresh_interval: 30s
    
    relabel_configs:
      # Use service name as job label
      - source_labels: [__meta_consul_service]
        target_label: job
      
      # Add datacenter label
      - source_labels: [__meta_consul_dc]
        target_label: datacenter
      
      # Add node label
      - source_labels: [__meta_consul_node]
        target_label: node
      
      # Add all service tags as labels
      - source_labels: [__meta_consul_tags]
        regex: ',(.*),'
        replacement: '$1'
        target_label: tags
```

**Register service in Consul**:
```json
{
  "service": {
    "name": "my-app",
    "tags": ["prometheus"],
    "port": 8080,
    "check": {
      "http": "http://localhost:8080/health",
      "interval": "10s"
    }
  }
}
```

### 5. EC2 Service Discovery

**Scenario**: Monitor EC2 instances

```yaml
scrape_configs:
  - job_name: 'ec2-nodes'
    ec2_sd_configs:
      - region: us-east-1
        access_key: YOUR_ACCESS_KEY
        secret_key: YOUR_SECRET_KEY
        port: 9100
        filters:
          - name: tag:Environment
            values: [production]
          - name: instance-state-name
            values: [running]
    
    relabel_configs:
      # Use private IP
      - source_labels: [__meta_ec2_private_ip]
        target_label: __address__
        replacement: '${1}:9100'
      
      # Add instance ID
      - source_labels: [__meta_ec2_instance_id]
        target_label: instance_id
      
      # Add availability zone
      - source_labels: [__meta_ec2_availability_zone]
        target_label: availability_zone
      
      # Add instance type
      - source_labels: [__meta_ec2_instance_type]
        target_label: instance_type
      
      # Add tags as labels
      - source_labels: [__meta_ec2_tag_Name]
        target_label: name
      
      - source_labels: [__meta_ec2_tag_Environment]
        target_label: environment
```

### 6. DNS Service Discovery

**Scenario**: Discover targets via DNS SRV records

```yaml
scrape_configs:
  - job_name: 'dns-sd'
    dns_sd_configs:
      - names:
          - '_prometheus._tcp.example.com'
        type: SRV
        refresh_interval: 30s
    
    relabel_configs:
      # Add hostname label
      - source_labels: [__meta_dns_name]
        target_label: hostname
```

**DNS SRV record**:
```
_prometheus._tcp.example.com. 300 IN SRV 0 0 9100 server1.example.com.
_prometheus._tcp.example.com. 300 IN SRV 0 0 9100 server2.example.com.
_prometheus._tcp.example.com. 300 IN SRV 0 0 9100 server3.example.com.
```

## Cấu Hình

### Relabeling Actions

**keep**: Chỉ giữ targets match regex
```yaml
- source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
  action: keep
  regex: true
```

**drop**: Drop targets match regex
```yaml
- source_labels: [__meta_kubernetes_namespace]
  action: drop
  regex: kube-system
```

**replace**: Replace target label
```yaml
- source_labels: [__meta_kubernetes_pod_name]
  target_label: pod
  action: replace
```

**labelmap**: Map meta labels to regular labels
```yaml
- action: labelmap
  regex: __meta_kubernetes_pod_label_(.+)
```

**labeldrop**: Drop labels matching regex
```yaml
- action: labeldrop
  regex: __meta_kubernetes_pod_label_pod_template_.*
```

**labelkeep**: Keep only labels matching regex
```yaml
- action: labelkeep
  regex: __name__|job|instance|namespace|pod
```

### Multiple Service Discovery

Combine multiple SD mechanisms:

```yaml
scrape_configs:
  - job_name: 'multi-sd'
    # Static targets
    static_configs:
      - targets: ['static-server:9100']
    
    # File-based SD
    file_sd_configs:
      - files: ['/etc/prometheus/targets/*.json']
    
    # Kubernetes SD
    kubernetes_sd_configs:
      - role: pod
    
    # Common relabeling for all targets
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
```

## Best Practices

### 1. Use Appropriate SD Mechanism

**Static**: Small, stable infrastructure
**File-based**: Custom discovery logic
**Kubernetes**: Kubernetes clusters
**Consul**: Consul-based infrastructure
**Cloud**: Cloud provider instances

### 2. Efficient Relabeling

Order relabeling rules efficiently:

```yaml
relabel_configs:
  # 1. Filter first (keep/drop)
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  
  # 2. Then transform
  - source_labels: [__meta_kubernetes_pod_name]
    target_label: pod
```

### 3. Label Cardinality

Avoid high cardinality labels:

```yaml
# ❌ Bad: Instance ID changes frequently
- source_labels: [__meta_ec2_instance_id]
  target_label: instance

# ✅ Good: Use stable identifier
- source_labels: [__meta_ec2_tag_Name]
  target_label: instance
```

### 4. Namespace Isolation

Separate configs by namespace:

```yaml
scrape_configs:
  # Production pods
  - job_name: 'k8s-pods-production'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: [production]
  
  # Staging pods
  - job_name: 'k8s-pods-staging'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: [staging]
```

### 5. Testing Relabeling

Test relabeling rules:

```bash
# Check discovered targets
curl http://localhost:9090/api/v1/targets

# Check target labels
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {labels}'

# Use promtool
promtool check config prometheus.yml
```

### 6. Monitoring Service Discovery

Monitor SD performance:

```yaml
- alert: ServiceDiscoveryFailure
  expr: prometheus_sd_discovered_targets == 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "No targets discovered for {{ $labels.config }}"

- alert: HighTargetChurn
  expr: rate(prometheus_target_sync_length_seconds_count[5m]) > 1
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "High target churn rate"
```

### 7. Security

Use RBAC for Kubernetes SD:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
```

## Tài Liệu Liên Quan

- [Prometheus Server](./01-prometheus-server.md) - Cấu hình scraping
- [Exporters](./02-exporters.md) - Targets để discover
- [Kubernetes Integration](../06-integrations/01-kubernetes-integration.md) - Chi tiết Kubernetes SD
- [Lab 06 - Service Discovery](../07-labs/lab-06-service-discovery/README.md) - Bài tập thực hành

## Tài Liệu Tham Khảo

- [Prometheus Service Discovery](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)
- [Kubernetes SD Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)
- [Relabeling](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config)
