# Lab 05 - Exporters

## Tổng Quan

Lab này hướng dẫn bạn tích hợp multiple exporters để monitor infrastructure. Bạn sẽ setup và configure các exporters phổ biến như blackbox_exporter, mysql_exporter, và postgres_exporter.

**Cấp độ**: Intermediate  
**Thời gian ước tính**: 90 phút  
**Yêu cầu trước**: Hoàn thành Lab 04 và Components module

## Mục Tiêu

Trong bài lab này, bạn sẽ:
- Cài đặt blackbox_exporter cho endpoint monitoring
- Setup mysql_exporter hoặc postgres_exporter
- Cấu hình Prometheus scrape multiple exporters
- Query metrics từ các exporters
- Tạo dashboards cho exporter metrics

## Nội Dung

- [Instructions](./instructions.md) - Hướng dẫn từng bước
- [Verification](./verification.md) - Cách kiểm tra kết quả

## Kết Quả Mong Đợi

Sau khi hoàn thành lab này, bạn sẽ có:
- Multiple exporters đang chạy
- Prometheus scraping từ tất cả exporters
- Có thể monitor HTTP endpoints, databases
- Hiểu cách configure và troubleshoot exporters

## Tiếp Theo

Sau khi hoàn thành lab này, tiếp tục với: [Lab 06 - Service Discovery](../lab-06-service-discovery/README.md)
