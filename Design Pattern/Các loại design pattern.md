## 1. Factory Pattern - tạo object đúng chuẩn, đúng lúc
Dùng khi cần tạo object phức tạp, nhiều loại, thay đổi linh hoạt
Ví dụ:
- Tạo connection DB theo môi trường
- Init các loại payment gateway (Momo, ZaloPay, Stripe…)
- Dùng cực nhiều ở các lớp service / adapter

## 2. Singleton - dùng nhưng phải cẩn thận
Rất phổ biến cho:
- Config loader
- Logger
- Connection pool
Tuy nhiên:
- Không thread-safe dùng sai là toang
- Trong Java, Spring sẽ inject singleton kiểu controlled

## 3. Strategy Pattern - thay đổi hành vi runtime
Gặp thường khi có nhiều cách xử lý cùng 1 vấn đề
Ví dụ:
- Tính phí ship theo nhiều khu vực
- Cách xử lý retry khác nhau tuỳ service
- Apply discount theo nhiều rule
- Kết hợp với DI Framework sẽ rất mạnh

## 4. Observer / Event-driven / Must trong microservices
- Không thể thiếu trong hệ thống async hoặc event-based
- Khi user checkout gửi nhiều event
- Service A đổi trạng thái notify B, C, D
- Framework thường dùng: Kafka, RabbitMQ, EventBridge...

## 5. Decorator - mở rộng hành vi mà không sửa code gốc
Dùng khi cần add thêm logic quanh core logic:
- Logging
- Auth check
- Caching
- Rate limit
Trong Spring Boot hay NestJS thì chính interceptor/middleware chính là decorator pattern

## 6. Adapter - nối 2 thứ không tương thích
- Dùng khi tích hợp hệ thống cũ (legacy) với module mới
- Convert API data model cũ sang chuẩn mới
- Tích hợp bên thứ ba vào hệ thống mình
- Đặc biệt quan trọng trong migration system

## 7. Builder - tạo object siêu linh hoạt
Dùng khi object có nhiều trường tùy chọn, cấu trúc phức tạp
Ví dụ:
- Tạo request gửi đi
- Tạo DTO trong tầng service
- Mapping entity -> model
- Đặc biệt hữu ích trong các API SDK

