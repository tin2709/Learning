Dựa trên mã nguồn và cấu trúc của dự án **Meteroid**, đây là phân tích chi tiết về hạ tầng định giá và tính cước (pricing & billing) này:

### 1. Công nghệ cốt lõi (Core Technology)

Meteroid được xây dựng với định hướng hiệu suất cực cao và khả năng mở rộng lớn, sử dụng các công nghệ hiện đại nhất:

*   **Ngôn ngữ lập trình:** **Rust (chiếm 70%)** cho toàn bộ Backend. Lựa chọn này đảm bảo an toàn bộ nhớ, tốc độ xử lý đồng thời (concurrency) tuyệt vời và tiêu thụ tài nguyên thấp. Frontend sử dụng **TypeScript & React**.
*   **Giao tiếp (Communication):**
    *   **gRPC (Tonic):** Sử dụng cho giao tiếp nội bộ giữa các service (Meteroid API, Metering API) để đạt độ trễ thấp nhất.
    *   **Protobuf (Buf):** Quản lý định nghĩa API thống nhất.
    *   **REST API (Axum):** Cung cấp giao thức HTTP cho người dùng cuối và tích hợp bên ngoài.
*   **Cơ sở dữ liệu (Storage Strategy):**
    *   **PostgreSQL (Diesel ORM):** Lưu trữ dữ liệu giao dịch (Transactional data) như thông tin khách hàng, gói cước, đăng ký và cấu hình.
    *   **ClickHouse:** Chuyên dụng để lưu trữ và truy vấn hàng tỷ sự kiện sử dụng (usage events) theo thời gian thực nhờ khả năng xử lý OLAP mạnh mẽ.
*   **Hệ thống tin nhắn (Messaging):** **Kafka (hoặc Redpanda)**. Đóng vai trò là bộ đệm cho luồng Ingestion, đảm bảo hệ thống không bị mất dữ liệu ngay cả khi lưu lượng sự kiện tăng đột biến.
*   **Rendering:** **Typst**. Một công cụ mới thay thế cho LaTeX/HTML-to-PDF, giúp render hóa đơn (Invoice) PDF cực nhanh và đẹp mắt.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Meteroid tuân thủ mô hình **Microservices-in-a-monorepo**:

*   **Tách biệt Ingestion và Billing:** Hệ thống tách riêng `metering` (tiếp nhận sự kiện) và `meteroid` (logic nghiệp vụ/tính tiền). Điều này cho phép scale riêng phần tiếp nhận sự kiện (thường có tải trọng rất lớn) mà không ảnh hưởng đến logic quản lý hóa đơn.
*   **Hệ thống định danh (Stripe-like IDs):** Sử dụng các tiền tố như `org_`, `cus_`, `sub_` kết hợp với **Base62 encoding** và **UUID v7**. Cách tiếp cận này giúp ID thân thiện với con người nhưng vẫn đảm bảo tính duy nhất và sắp xếp theo thời gian.
*   **Telemetry-first:** Hệ thống tích hợp sâu OpenTelemetry (OTEL), Prometheus và Grafana ngay từ lớp `common-logging`, cho phép quan sát chi tiết từng request gRPC và hiệu năng xử lý.
*   **Multi-tenant & Modular:** Hỗ trợ đa tổ chức (Multi-organization) và chia nhỏ mã nguồn thành các crate dùng chung (`common-config`, `common-domain`, `common-utils`).

### 3. Các kỹ thuật chính (Key Techniques)

*   **Distributed Locking (Advisory Locks):** Sử dụng tính năng khóa cố vấn của Postgres (`pg_try_advisory_lock`) để đảm bảo các quy trình quan trọng như chốt hóa đơn (Invoicing Finalize) hoặc tính tiền không bị chạy trùng lặp khi có nhiều node scheduler.
*   **Idempotency (Tính bất biến):** Triển khai lớp Middleware cho gRPC/REST để lưu cache kết quả xử lý dựa trên `idempotency-key`, ngăn chặn việc tính tiền hai lần khi client gửi lại request do lỗi mạng.
*   **Outbox Pattern & PGMQ:** Sử dụng hàng đợi tin nhắn dựa trên Postgres (PGMQ) để đảm bảo tính nhất quán giữa việc cập nhật database và phát đi sự kiện (ví dụ: tạo xong Customer thì mới phát Webhook).
*   **Proration & Lifecycle Management:** Thuật toán tính toán phần bù (proration) phức tạp khi khách hàng nâng cấp/hạ cấp gói cước giữa chu kỳ thanh toán.
*   **Grandfathering:** Kỹ thuật phiên bản hóa (Versioning) các Plan, cho phép khách hàng cũ giữ mức giá cũ trong khi áp dụng giá mới cho khách hàng mới.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Hệ thống vận hành theo chu kỳ khép kín:

1.  **Luồng Ingestion (Sự kiện):**
    *   Ứng dụng khách gửi sự kiện qua `Metering API`.
    *   Sự kiện được đẩy vào `Kafka`.
    *   `ClickHouse` tiêu thụ sự kiện từ Kafka và lưu trữ ở dạng thô/tổng hợp.
2.  **Luồng Scheduler (Lập lịch):**
    *   Service `Scheduler` chạy định kỳ, quét các `Subscriptions` trong Postgres.
    *   Khi đến kỳ hạn, nó yêu cầu `Meteroid API` tính toán tiền.
3.  **Luồng Billing (Tính tiền):**
    *   `Meteroid API` gửi truy vấn gRPC sang `Metering API`.
    *   `Metering API` thực hiện aggregation trên `ClickHouse` để lấy lượng usage thực tế.
    *   `Meteroid API` áp dụng công thức giá (Flat, Tiered, Hybrid) và tạo hóa đơn nháp (Draft Invoice) trong Postgres.
4.  **Luồng Finalization (Hoàn tất):**
    *   Hóa đơn được finalize, `Typst` render thành tệp PDF.
    *   Hệ thống gọi integration (ví dụ: Stripe) để trừ tiền thực tế và gửi Webhook/Email cho khách hàng qua `Svix`.

**Kết luận:** Meteroid là một bản triển khai xuất sắc bằng **Rust**, giải quyết bài toán billing phức tạp bằng cách kết hợp thế mạnh của cơ sở dữ liệu quan hệ (Postgres) cho tính nhất quán và cơ sở dữ liệu cột (ClickHouse) cho tốc độ xử lý dữ liệu lớn. Đây là giải pháp thay thế mã nguồn mở mạnh mẽ cho các dịch vụ như Orb hoặc Lago.