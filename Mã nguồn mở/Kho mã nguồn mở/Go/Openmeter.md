Dựa trên cấu trúc thư mục và mã nguồn của dự án **OpenMeter**, dưới đây là bản phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống dưới dạng một file README tiếng Việt chuyên sâu.

---

# Phân Tích Kỹ Thuật Dự Án OpenMeter

OpenMeter là một nền tảng mã nguồn mở về **Metering & Billing** (Đo lường & Tính phí) hiệu năng cao, được thiết kế dành riêng cho các mô hình AI, API và SaaS. Hệ thống giải quyết bài toán khó nhất trong Billing: Thu thập hàng triệu sự kiện (events) theo thời gian thực và chuyển đổi chúng thành hóa đơn chính xác.

## 1. Công Nghệ Cốt Lõi (Core Tech Stack)

Dự án sử dụng mô hình "Modern Data Stack" cho bài toán tài chính:

*   **Ngôn ngữ lập trình:** 
    *   **Go (73%):** Dùng cho backend service, worker xử lý dữ liệu nhờ ưu thế về concurrency (Goroutines).
    *   **TypeScript (21%):** Xây dựng các Client SDK và phần logic portal cho người dùng.
*   **Hệ thống lưu trữ & Truy vấn:**
    *   **ClickHouse:** "Trái tim" của hệ thống, dùng để lưu trữ và phân tích hàng tỷ sự kiện usage với tốc độ cực nhanh.
    *   **PostgreSQL:** Lưu trữ dữ liệu nghiệp vụ (Metadata), cấu hình gói cước (Plans), đăng ký (Subscriptions) và thông tin khách hàng.
    *   **Redis:** Dùng làm Cache và Deduplication (chống trùng lặp dữ liệu) trong quá trình Ingestion.
*   **Hạ tầng dữ liệu (Data Infrastructure):**
    *   **Apache Kafka:** Hệ thống hàng đợi tin nhắn (Message Queue) đảm bảo tính chịu lỗi và khả năng mở rộng khi lượng event tăng đột biến.
    *   **Benthos (Redpanda Connect):** Được sử dụng trong `collector` để thu thập dữ liệu từ nhiều nguồn (K8s, logs, HTTP) một cách linh hoạt.
*   **Công cụ phát triển (Dev Tools):**
    *   **Ent (Facebook/Ariel):** ORM mạnh mẽ cho Go, dùng để quản lý schema PostgreSQL thông qua mã nguồn (Code-first).
    *   **TypeSpec (Microsoft):** Định nghĩa API chuẩn hóa, từ đó tự động sinh ra OpenAPI spec và SDK cho Go, Python, JS.
    *   **Atlas:** Quản lý migration database PostgreSQL chuyên nghiệp.

## 2. Tư Duy Kiến Trúc (Architectural Design)

OpenMeter áp dụng kiến trúc **Event-Driven Microservices** phối hợp với **Clean Architecture**:

*   **Tách biệt Ingestion và Processing:** Hệ thống không ghi trực tiếp vào database chính khi nhận event. Event được đẩy vào Kafka để đảm bảo API luôn phản hồi dưới 10ms.
*   **Kiến trúc Lambda rút gọn:** Dữ liệu thô (raw events) nằm trong ClickHouse, dữ liệu trạng thái (current balances) được tính toán và snapshot thường xuyên bởi các `balance-worker`.
*   **Domain-Driven Design (DDD):** Mã nguồn được chia theo các domain nghiệp vụ rõ rệt trong thư mục `openmeter/`:
    *   `billing/`: Quản lý hóa đơn và quy trình thu phí.
    *   `entitlement/`: Quản lý quyền hạn sử dụng (ví dụ: dùng tối đa 1000 tokens).
    *   `productcatalog/`: Quản lý các gói cước và giá cả.

## 3. Các Kỹ Thuật Chính Nổi Bật

*   **Idempotency & Deduplication:** Sử dụng Redis/Memory để kiểm tra `event_id`. Đảm bảo một sự kiện usage chỉ được tính phí đúng một lần duy nhất dù hệ thống có retry.
*   **High-Throughput Ingestion:** Tối ưu hóa việc batching dữ liệu từ Kafka vào ClickHouse thông qua `sink-worker`, giúp giảm tải IO cho database.
*   **Flexible Aggregation:** Hỗ trợ nhiều kiểu tính toán (SUM, COUNT, MAX, UNIQUE_COUNT) trực tiếp trên các thuộc tính của JSON payload (sử dụng JSONPath).
*   **Automated SDK Generation:** Sử dụng TypeSpec để đảm bảo sự đồng bộ tuyệt đối giữa server và các SDK (Go, Python, JS). Developer chỉ cần sửa file `.tsp`, toàn bộ code client sẽ được tự động cập nhật.
*   **Webhook Management:** Tích hợp **Svix** để gửi thông báo thời gian thực khi khách hàng dùng hết hạn mức (threshold alerts).

## 4. Luồng Hoạt Động (Data Flow)

Hệ thống hoạt động theo 4 bước chính:

### Bước 1: Thu thập (Ingestion)
Người dùng gửi usage event qua API hoặc thông qua `collector` (thu thập logs/metrics). 
- **Validation:** Kiểm tra định dạng CloudEvents.
- **Deduplication:** Kiểm tra ID trùng lặp.
- **Produce:** Đẩy vào Kafka topic.

### Bước 2: Lưu trữ (Sinking)
`sink-worker` tiêu thụ dữ liệu từ Kafka.
- **Batching:** Gom các sự kiện lại thành từng nhóm.
- **Write:** Ghi vào ClickHouse theo dạng cột (columnar) để tối ưu truy vấn phân tích.

### Bước 3: Xử lý nghiệp vụ (Processing)
- **Balance Worker:** Theo dõi lượng sử dụng trong ClickHouse và so sánh với hạn mức (Entitlements) trong PostgreSQL.
- **Billing Worker:** Định kỳ (hoặc theo sự kiện) tính toán số tiền dựa trên usage và cấu hình Plan (ví dụ: $0.01 cho mỗi 1k tokens).

### Bước 4: Thực thi & Thông báo (Enforcement & Notification)
- **Access Check:** Khi ứng dụng hỏi "User X còn quyền gọi API không?", hệ thống kiểm tra số dư hiện tại.
- **Notification:** Nếu usage đạt ngưỡng (ví dụ 80%), Svix sẽ trigger webhook gửi đến hệ thống của khách hàng.
- **Payment:** Kết nối với Stripe để thực hiện trừ tiền thực tế.

---

## 5. Hướng dẫn phát triển nhanh (Quick Start)

Dự án sử dụng **Nix** và **direnv** để thiết lập môi trường chuẩn:

1. **Khởi động hạ tầng:** `make up` (Chạy Kafka, ClickHouse, Postgres, Redis qua Docker).
2. **Chạy Service:** `make run` (Khởi động OpenMeter server).
3. **Kiểm tra:** `make test` (Chạy unit test và integration test).
4. **Sinh code:** `make generate` (Khi thay đổi Schema hoặc API).

---
*Dự án này là minh chứng tốt cho việc kết hợp giữa Go (hiệu suất xử lý) và ClickHouse (hiệu suất dữ liệu lớn) để giải quyết bài toán tài chính phức tạp.*