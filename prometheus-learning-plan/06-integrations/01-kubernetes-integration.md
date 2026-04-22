# Kubernetes Integration

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Kiến Trúc](#kiến-trúc)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
  - [Kubernetes Service Discovery](#kubernetes-service-discovery)
  - [kube-state-metrics](#kube-state-metrics)
  - [node-exporter DaemonSet](#node-exporter-daemonset)
  - [Prometheus Operator](#prometheus-operator)
  - [Common Kubernetes Alerts](#common-kubernetes-alerts)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)
- [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

---

## Tổng Quan

Kubernetes là nền tảng container orchestration phổ biến nhất hiện nay. Việc monitor Kubernetes cluster đòi hỏi thu thập metrics từ nhiều tầng khác nhau: infrastructure (nodes), platform (Kubernetes objects), và application (containers/pods).

Prometheus là công cụ monitoring được khuyến nghị chính thức cho Kubernetes, với hỗ trợ native service discovery và nhiều exporters chuyên dụng.

**Các thành phần chính để monitor Kubernetes:**
- **kube-state-metrics**: Metrics về trạng thái Kubernetes objects (Deployments, Pods, Services...)
- **node-exporter**: Metrics về hardware và OS của từng node
- **cAdvisor**: Metrics về resource usage của containers (tích hợp sẵn trong kubelet)
- **Prometheus Operator**: Quản lý Prometheus trên Kubernetes bằng CRDs

---

## Kiến Trúc

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │   Node 1     │  │   Node 2     │  │    Node 3        │  │
│  │              │  │              │  │                  │  │
│  │ node-exporter│  │ node-exporter│  │  node-exporter   │  │
│  │ (DaemonSet)  │  │ (DaemonSet)  │  │  (DaemonSet)     │  │
│  │              │  │              │  │                  │  │
│  │  kubelet     │  │  kubelet     │  │  kubelet         │  │
│  │  (cAdvisor)  │  │  (cAdvisor)  │  │  (cAdvisor)      │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Control Plane                          │   │
│  │  kube-apiserver  kube-scheduler  kube-controller    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────┐   ┌──────────────────────────────┐   │
│  │ kube-state-metrics│   │  Prometheus (Operator)       │   │
│  │  (Deployment)    │   │  - ServiceMonitor CRDs        │   │
│  └──────────────────┘   │  - PrometheusRule CRDs        │   │
│                          └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ scrape
                    ┌──────────────────┐
                    │   Prometheus     │
                    │   Server         │
                    └──────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │    Grafana       │
                    └──────────────────┘
```

---

## Cách Hoạt Động

### 1. Kubernetes Service Discovery

Prometheus sử dụng Kubernetes API để tự động phát hiện targets cần scrape. Có 5 loại service discovery roles:

| Role | Mô tả |
|------|-------|
| `node` | Phát hiện tất cả nodes trong cluster |
| `pod` | Phát hiện tất cả pods đang chạy |
| `service` | Phát hiện tất cả services |
| `endpoints` | Phát hiện endpoints của services |
| `ingress` | Phát hiện ingress resources |

### 2. Metrics Flow

```
Kubernetes Objects
      │
      ▼
kube-state-metrics ──────────────────────────────┐
      │                                           │
kubelet/cAdvisor ─────────────────────────────── ├──► Prometheus ──► Grafana
      │                                           │
node-exporter ────────────────────────────────── ┘
      │
Application Pods (custom metrics)
```

### 3. RBAC Permissions

Prometheus cần RBAC permissions để đọc Kubernetes API:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
```

---

## Use Cases

1. **Cluster Health Monitoring**: Theo dõi trạng thái nodes, pods, deployments
2. **Resource Capacity Planning**: Phân tích CPU/memory usage để lên kế hoạch scaling
3. **Application Performance**: Monitor latency, error rates của microservices
4. **Cost Optimization**: Phát hiện pods sử dụng quá nhiều hoặc quá ít resources
5. **SLO Tracking**: Đo lường availability và performance của services
6. **Incident Detection**: Alert khi pods crash, nodes không healthy

---

## Cấu Hình

### Kubernetes Service Discovery

Cấu hình Prometheus để scrape metrics từ Kubernetes cluster:

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Scrape Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Scrape Kubernetes API server
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
      - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

  # Scrape Kubernetes nodes (kubelet)
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

  # Scrape cAdvisor (container metrics)
  - job_name: 'kubernetes-cadvisor'
    kubernetes_sd_configs:
      - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

  # Scrape pods với annotation prometheus.io/scrape: "true"
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

  # Scrape services với annotation prometheus.io/scrape: "true"
  - job_name: 'kubernetes-service-endpoints'
    kubernetes_sd_configs:
      - role: endpoints
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name
```

### kube-state-metrics

kube-state-metrics expose metrics về trạng thái Kubernetes objects từ Kubernetes API.

**Cài đặt bằng kubectl:**

```bash
# Cài đặt kube-state-metrics
kubectl apply -f https://github.com/kubernetes/kube-state-metrics/releases/download/v2.10.0/standard/

# Hoặc dùng Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-state-metrics prometheus-community/kube-state-metrics \
  --namespace monitoring \
  --create-namespace
```

**Deployment manifest:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: monitoring
  labels:
    app: kube-state-metrics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
        - name: kube-state-metrics
          image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.10.0
          ports:
            - name: http-metrics
              containerPort: 8080
            - name: telemetry
              containerPort: 8081
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /
              port: 8081
            initialDelaySeconds: 5
            timeoutSeconds: 5
          resources:
            requests:
              cpu: 10m
              memory: 32Mi
            limits:
              cpu: 250m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics
  namespace: monitoring
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
spec:
  selector:
    app: kube-state-metrics
  ports:
    - name: http-metrics
      port: 8080
      targetPort: http-metrics
    - name: telemetry
      port: 8081
      targetPort: telemetry
```

**Metrics quan trọng từ kube-state-metrics:**

```promql
# Số pods đang chạy theo namespace
count(kube_pod_status_phase{phase="Running"}) by (namespace)

# Deployments không đủ replicas
kube_deployment_status_replicas_available < kube_deployment_spec_replicas

# Pods đang ở trạng thái Pending
kube_pod_status_phase{phase="Pending"} == 1

# Node không ready
kube_node_status_condition{condition="Ready",status="true"} == 0

# PersistentVolumeClaims không bound
kube_persistentvolumeclaim_status_phase{phase!="Bound"} == 1
```

### node-exporter DaemonSet

node-exporter thu thập metrics về hardware và OS của từng node. Dùng DaemonSet để đảm bảo chạy trên mọi node.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9100"
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      tolerations:
        - operator: Exists
          effect: NoSchedule
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          args:
            - --path.procfs=/host/proc
            - --path.sysfs=/host/sys
            - --path.rootfs=/host/root
            - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
          ports:
            - containerPort: 9100
              protocol: TCP
          resources:
            requests:
              cpu: 10m
              memory: 24Mi
            limits:
              cpu: 200m
              memory: 100Mi
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
            - name: root
              mountPath: /host/root
              mountPropagation: HostToContainer
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
        - name: root
          hostPath:
            path: /
---
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9100"
spec:
  selector:
    app: node-exporter
  ports:
    - name: metrics
      port: 9100
      targetPort: 9100
  clusterIP: None  # Headless service
```

### Prometheus Operator

Prometheus Operator đơn giản hóa việc deploy và quản lý Prometheus trên Kubernetes bằng Custom Resource Definitions (CRDs).

**Cài đặt Prometheus Operator với kube-prometheus-stack:**

```bash
# Thêm Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Cài đặt kube-prometheus-stack (bao gồm Prometheus, Grafana, Alertmanager)
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=standard \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

**ServiceMonitor CRD - Định nghĩa target scraping:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring
  labels:
    release: kube-prometheus-stack  # Phải match với Prometheus selector
spec:
  selector:
    matchLabels:
      app: my-app  # Chọn services có label này
  namespaceSelector:
    matchNames:
      - production
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
      scheme: http
```

**PrometheusRule CRD - Định nghĩa alerting rules:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: kubernetes.rules
      interval: 30s
      rules:
        - alert: KubePodCrashLooping
          expr: |
            rate(kube_pod_container_status_restarts_total[15m]) * 60 * 5 > 0
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} đang crash loop"
            description: "Container {{ $labels.container }} restart {{ $value }} lần trong 15 phút"

        - alert: KubeNodeNotReady
          expr: kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 15m
          labels:
            severity: critical
          annotations:
            summary: "Node {{ $labels.node }} không ready"
```

**Prometheus CRD - Cấu hình Prometheus instance:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 2
  retention: 30d
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      release: kube-prometheus-stack
  ruleSelector:
    matchLabels:
      release: kube-prometheus-stack
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: standard
        resources:
          requests:
            storage: 50Gi
  resources:
    requests:
      memory: 400Mi
    limits:
      memory: 2Gi
  alerting:
    alertmanagers:
      - namespace: monitoring
        name: alertmanager-main
        port: web
```

### Common Kubernetes Alerts

```yaml
# kubernetes-alerts.yml
groups:
  - name: kubernetes-node-alerts
    rules:
      - alert: NodeHighCPU
        expr: |
          100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.instance }} CPU cao"
          description: "CPU usage: {{ $value | humanize }}%"

      - alert: NodeHighMemory
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.instance }} memory cao"
          description: "Memory usage: {{ $value | humanize }}%"

      - alert: NodeDiskSpaceLow
        expr: |
          (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.instance }} disk sắp đầy"
          description: "Còn {{ $value | humanize }}% disk space"

  - name: kubernetes-pod-alerts
    rules:
      - alert: PodCrashLooping
        expr: |
          rate(kube_pod_container_status_restarts_total[15m]) * 60 * 5 > 0
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} crash looping"

      - alert: PodNotReady
        expr: |
          sum by (namespace, pod) (
            max by(namespace, pod) (
              kube_pod_status_phase{phase=~"Pending|Unknown"}
            ) * on(namespace, pod) group_left(owner_kind)
            max by(namespace, pod, owner_kind) (
              kube_pod_owner{owner_kind!="Job"}
            )
          ) > 0
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} không ready"

      - alert: ContainerOOMKilled
        expr: |
          kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.container }} bị OOMKilled"

  - name: kubernetes-deployment-alerts
    rules:
      - alert: DeploymentReplicasMismatch
        expr: |
          kube_deployment_spec_replicas != kube_deployment_status_replicas_available
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} thiếu replicas"
          description: "Expected {{ $value }} replicas nhưng không đủ available"

      - alert: DeploymentGenerationMismatch
        expr: |
          kube_deployment_status_observed_generation != kube_deployment_metadata_generation
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} rollout bị stuck"
```

---

## Best Practices

1. **Dùng Prometheus Operator**: Quản lý Prometheus bằng CRDs thay vì cấu hình thủ công
2. **Namespace isolation**: Deploy monitoring stack vào namespace riêng (`monitoring`)
3. **Resource limits**: Luôn đặt resource requests/limits cho Prometheus và exporters
4. **Persistent storage**: Dùng PersistentVolumeClaim cho Prometheus data
5. **RBAC minimal**: Chỉ cấp permissions cần thiết cho Prometheus service account
6. **Label consistency**: Dùng labels nhất quán để dễ filter và aggregate
7. **Recording rules**: Tạo recording rules cho queries phức tạp thường dùng
8. **Retention policy**: Cân nhắc retention period phù hợp với storage capacity
9. **High availability**: Chạy 2 replicas Prometheus với Thanos hoặc Cortex cho HA
10. **Annotations cho auto-discovery**: Dùng annotations `prometheus.io/scrape`, `prometheus.io/port` cho pods

---

## Tài Liệu Liên Quan

- [Docker Monitoring](./02-docker-monitoring.md) - Container monitoring với cAdvisor
- [Monitoring Stacks](./04-monitoring-stacks.md) - Complete stack với Prometheus Operator
- [Service Discovery](../02-components/05-service-discovery.md) - Kubernetes service discovery chi tiết
- [Alerting Rules](../05-alerting/01-alerting-rules.md) - Viết alerting rules

---

## Tài Liệu Tham Khảo

- [Prometheus Kubernetes SD Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)
- [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [kube-prometheus-stack Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [node-exporter](https://github.com/prometheus/node_exporter)
