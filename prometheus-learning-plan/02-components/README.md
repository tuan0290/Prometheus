# Module 02: Components

**Cấp độ**: Intermediate  
**Thời gian ước tính**: 12 giờ  
**Yêu cầu trước**: Hoàn thành Fundamentals

## Tổng Quan

Module này giải thích chi tiết các thành phần của Prometheus ecosystem. Bạn sẽ học về Prometheus Server, Exporters, Alertmanager, Pushgateway, Service Discovery, và Storage mechanisms.

## Mục Tiêu Học Tập

Sau khi hoàn thành module này, bạn sẽ có thể:
- Hiểu cách Prometheus Server hoạt động (retrieval, storage, HTTP API)
- Cấu hình và sử dụng các exporters phổ biến
- Thiết lập Alertmanager cho alert routing
- Biết khi nào sử dụng Pushgateway
- Cấu hình service discovery mechanisms
- Quản lý storage và data retention

## Nội Dung

1. [Prometheus Server](./01-prometheus-server.md) - Core server component
2. [Exporters](./02-exporters.md) - Metrics exporters
3. [Alertmanager](./03-alertmanager.md) - Alert routing và notification
4. [Pushgateway](./04-pushgateway.md) - Push metrics gateway
5. [Service Discovery](./05-service-discovery.md) - Dynamic target discovery
6. [Storage and Retention](./06-storage-retention.md) - Data storage mechanisms

## Checkpoint

Đánh dấu khi bạn hoàn thành:
- [ ] Hiểu 3 phần chính của Prometheus Server
- [ ] Biết cách sử dụng node_exporter và blackbox_exporter
- [ ] Có thể cấu hình Alertmanager routing rules
- [ ] Hiểu limitations của Pushgateway
- [ ] Cấu hình được service discovery cho Kubernetes
- [ ] Quản lý được retention policies

## Tiếp Theo

Sau khi hoàn thành module này, tiếp tục với: [PromQL](../03-promql/README.md)
