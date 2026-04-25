Dựa trên mã nguồn và tài liệu của dự án **getlago/lago**, đây là phân tích chi tiết về kiến trúc và công nghệ của một nền tảng billing (tính cước) mã nguồn mở hiện đại:

### 1. Công nghệ cốt lõi (Core Technology)
Lago sử dụng chiến lược **Polyglot Persistence** và **Multi-language**, kết hợp sức mạnh của nhiều ngôn ngữ và cơ sở dữ liệu khác nhau:

*   **Backend chính (API & Admin):** Sử dụng **Ruby on Rails**. Đây là nơi xử lý logic nghiệp vụ phức tạp, quản lý quan hệ khách hàng, kế hoạch giá (plans) và giao diện quản trị.
*   **Xử lý hiệu năng cao (Events Processor):** Được viết bằng **Go (Golang)**. Đây là thành phần chuyên biệt để xử lý luồng dữ liệu sự kiện (usage events) cực lớn với độ trễ thấp, giao tiếp qua gRPC và Kafka.
*   **Frontend:** **TypeScript & React**, xây dựng giao diện dashboard hiện đại.
*   **Cơ sở dữ liệu (Storage Strategy):**
    *   **PostgreSQL:** Lưu trữ dữ liệu giao dịch (transactional) và cấu hình hệ thống. Sử dụng `pg_partman` để phân vùng (partitioning) bảng dữ liệu lớn (như `enriched_events`).
    *   **ClickHouse:** Dùng cho lưu trữ và phân tích dữ liệu sử dụng (usage) ở quy mô lớn, cho phép truy vấn aggregation cực nhanh để tính tiền theo thời gian thực.
    *   **Redis:** Đóng 3 vai trò: Hàng đợi công việc (Sidekiq), Cache hệ thống và Lưu trữ trạng thái xử lý sự kiện (Event Store).
*   **Hệ thống tin nhắn (Messaging):** **Redpanda** (một nền tảng tương thích Kafka nhưng viết bằng C++ cho hiệu năng cao hơn). Đây là "xương sống" để vận chuyển các sự kiện từ lúc ingestion đến khi enrichment.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của Lago được thiết kế theo hướng **Event-Driven (Dựa trên sự kiện)** và **Decoupled (Tách biệt hóa)**:

*   **Hybrid Monolith & Microservices:** Lago giữ một "Core Rails" cho các tác vụ quản lý (CRUD) nhưng tách riêng phần "Ingestion & Processing" thành các service Go để tối ưu tài nguyên.
*   **Dedicated Worker Architecture:** Thay vì dùng một worker chung, Lago chia nhỏ các Sidekiq workers thành các nhóm chuyên biệt (Analytics, Billing, Webhook, PDF, AI Agent). Điều này cho phép scale riêng lẻ từng bộ phận (ví dụ: scale Webhook worker khi có quá nhiều thông báo).
*   **Infrastructure-as-Code & Portability:** Hệ thống được container hóa hoàn toàn qua Docker, hỗ trợ từ deployment "Light" (cho startup) đến "Production" (có Traefik, Portainer và khả năng scale ngang).
*   **Payment-Agnostic:** Kiến trúc cho phép tích hợp với bất kỳ cổng thanh toán nào (Stripe, Adyen,...) mà không làm thay đổi logic tính toán cốt lõi.

### 3. Các kỹ thuật chính (Key Techniques)
*   **Event Enrichment (Làm giàu dữ liệu):** Khi một sự kiện thô (raw event) đổ vào, `events-processor` (Go) sẽ ngay lập tức đối chiếu với dữ liệu Subscription trong Postgres để gắn thêm các thông tin về Plan, Charge ID trước khi đẩy vào ClickHouse.
*   **Database Partitioning:** Kỹ thuật chia bảng `enriched_events` theo tháng (Range Partitioning) giúp Postgres duy trì hiệu năng khi dữ liệu lên tới hàng tỷ dòng.
*   **Advanced Encryption:** Sử dụng nhiều tầng bảo mật:
    *   ActiveRecord Encryption cho dữ liệu nhạy cảm trong DB.
    *   HMAC-SHA256 để ký Webhook (Symmetric).
    *   RSA/JWT để ký Webhook (Asymmetric), cho phép khách hàng xác thực mà không cần chia sẻ key bí mật.
*   **Idempotency (Tính bất biến):** Đảm bảo một sự kiện hoặc một yêu cầu thanh toán nếu gửi trùng lặp nhiều lần cũng chỉ được xử lý một lần duy nhất qua `transaction_id`.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Hệ thống vận hành qua 4 giai đoạn chính:

1.  **Ingestion (Tiếp nhận):**
    *   App người dùng gửi sự kiện sử dụng qua HTTP hoặc SQS.
    *   `Connectors` (Redpanda Connect) nhận dữ liệu và đẩy vào topic `events-raw`.
2.  **Enrichment (Xử lý Go):**
    *   Service `events-processor` (Go) đọc từ topic `events-raw`.
    *   Nó kiểm tra cache (Redis) và DB (Postgres) để xác định sự kiện này thuộc Subscription nào, Plan nào.
    *   Dữ liệu đã "giàu" thông tin được đẩy vào topic `events-enriched` và lưu vào ClickHouse.
3.  **Billing (Tính cước - Sidekiq/Clock):**
    *   Service `api-clock` theo dõi thời gian. Đến kỳ hạn billing, nó enqueued một job vào `Billing Worker`.
    *   Worker này truy vấn ClickHouse để lấy tổng lượng usage, áp công thức tính giá và tạo Invoice trong Postgres.
4.  **Action (Hành động):**
    *   `PDF Worker` gọi Gotenberg để sinh file Invoice PDF.
    *   `Webhook Worker` gửi thông báo cho hệ thống bên ngoài.
    *   `Payment Orchestration` thực hiện lệnh trừ tiền qua cổng thanh toán đã cấu hình.

**Kết luận:** Lago là một hệ thống billing được thiết kế cực kỳ bài bản cho quy mô lớn. Việc sử dụng Go cho xử lý sự kiện và ClickHouse cho phân tích giúp nó vượt qua giới hạn hiệu năng của các hệ thống billing truyền thống chỉ dựa trên database quan hệ.