# Lab 02 - Custom Metrics

## Tổng Quan

Lab này hướng dẫn bạn tạo và expose custom metrics từ application. Bạn sẽ sử dụng Prometheus client library để instrument code và expose metrics endpoint.

**Cấp độ**: Intermediate  
**Thời gian ước tính**: 90 phút  
**Yêu cầu trước**: Hoàn thành Lab 01 và Instrumentation module

## Mục Tiêu

Trong bài lab này, bạn sẽ:
- Tạo simple application với Prometheus client library
- Implement counter, gauge, và histogram metrics
- Expose /metrics endpoint
- Cấu hình Prometheus scrape custom application
- Query custom metrics với PromQL

## Nội Dung

- [Instructions](./instructions.md) - Hướng dẫn từng bước
- [Verification](./verification.md) - Cách kiểm tra kết quả

## Kết Quả Mong Đợi

Sau khi hoàn thành lab này, bạn sẽ có:
- Application expose custom metrics
- Prometheus scraping custom metrics
- Có thể query và visualize custom metrics
- Hiểu cách instrument application code

## Tiếp Theo

Sau khi hoàn thành lab này, tiếp tục với: [Lab 03 - PromQL Queries](../lab-03-promql-queries/README.md)
