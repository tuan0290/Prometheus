# Module 04: Instrumentation

**Cấp độ**: Intermediate  
**Thời gian ước tính**: 8 giờ  
**Yêu cầu trước**: Hoàn thành PromQL

## Tổng Quan

Module này hướng dẫn cách tạo custom metrics và instrument applications. Bạn sẽ học về client libraries, naming conventions, và best practices cho instrumentation.

## Mục Tiêu Học Tập

Sau khi hoàn thành module này, bạn sẽ có thể:
- Sử dụng Prometheus client libraries (Go, Python, Java)
- Tạo custom metrics trong application code
- Áp dụng metric naming conventions
- Quản lý label cardinality
- Sử dụng common exporters cho infrastructure monitoring

## Nội Dung

1. [Client Libraries](./01-client-libraries.md) - Prometheus client libraries
2. [Custom Metrics](./02-custom-metrics.md) - Tạo custom metrics
3. [Common Exporters](./03-common-exporters.md) - Exporters phổ biến
4. [Best Practices](./04-best-practices.md) - Naming và cardinality

## Checkpoint

Đánh dấu khi bạn hoàn thành:
- [ ] Instrument được application với client library
- [ ] Tạo được custom counter, gauge, histogram metrics
- [ ] Áp dụng được naming conventions
- [ ] Hiểu và tránh được high cardinality issues
- [ ] Biết cách sử dụng common exporters

## Tiếp Theo

Sau khi hoàn thành module này, tiếp tục với: [Alerting](../05-alerting/README.md)
