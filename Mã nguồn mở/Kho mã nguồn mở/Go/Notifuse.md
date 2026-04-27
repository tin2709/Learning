Dựa trên mã nguồn của **Notifuse** (nền tảng Marketing Automation và Email Transactional mã nguồn mở), dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống:

### 1. Công nghệ cốt lõi (Core Technology)

Notifuse sử dụng một "stack" công nghệ hiện đại, ưu tiên hiệu suất cao và khả năng tự vận hành (self-hosted):

*   **Backend:**
    *   **Ngôn ngữ:** **Go (Golang) 1.25.x**, tận dụng tính năng thực thi đồng thời (concurrency) để xử lý hàng triệu email.
    *   **Framework:** Sử dụng thư viện tiêu chuẩn `http.ServeMux` (không dùng framework bên thứ ba như Gin/Echo), giúp mã nguồn tối giản và hiệu quả.
    *   **Database:** **PostgreSQL 17** kết hợp với **Squirrel** (Query Builder) để tạo các câu lệnh SQL an toàn và linh hoạt.
    *   **Email Rendering:** **MJML** (gomjml) để tạo email responsive và **Liquid** (liquidgo) cho các nội dung động (dynamic content).
*   **Frontend:**
    *   **Admin Console:** React 18, TypeScript, Vite, Ant Design (UI Library).
    *   **Notification Center:** React 19, Tailwind CSS 4, Radix UI, Lucide React.
*   **Observability (Khả năng quan sát):** **OpenCensus** tích hợp sẵn để tracing và xuất dữ liệu ra Jaeger, Zipkin, Prometheus, AWS X-Ray.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Hệ thống áp dụng triệt để **Clean Architecture** (Kiến trúc sạch) để đảm bảo tính dễ bảo trì và kiểm thử:

*   **Phân lớp logic (Layered Pattern):**
    *   `Domain`: Chứa thực thể (Entities) và quy tắc nghiệp vụ cốt lõi.
    *   `Service`: Xử lý logic nghiệp vụ và điều phối dữ liệu.
    *   `Repository`: Lớp truy cập dữ liệu (Postgres).
    *   `HTTP`: Xử lý API handlers và middleware.
*   **Kiến trúc Đa thuê (Multi-tenancy):** Quản lý theo cấu trúc **Workspaces**. Mỗi Workspace có thể có cấu hình và dữ liệu khách hàng riêng biệt.
*   **Hệ thống Migration tùy chỉnh:** Không dùng các công cụ migration thông thường, Notifuse xây dựng một hệ thống migration theo phiên bản (`vMAJOR.minor`) cho cả database hệ thống và database của từng Workspace riêng lẻ.
*   **Pluggable Providers:** Kiến trúc cho phép dễ dàng thay đổi hoặc thêm mới các nhà cung cấp Email (Amazon SES, Mailgun, SMTP...) thông qua các Interface đồng nhất.

### 3. Kỹ thuật lập trình chính (Primary Programming Techniques)

*   **Dependency Injection (DI):** Sử dụng các hàm `NewService`, `NewRepository` để "tiêm" phụ thuộc, giúp việc thay thế các Mock object khi viết Unit Test trở nên dễ dàng.
*   **Graceful Shutdown:** Hệ thống xử lý tín hiệu (SIGTERM) rất kỹ lưỡng, cho phép các tiến trình gửi email đang chạy có tới 65-70 giây để hoàn tất hoặc lưu trạng thái trước khi tắt hẳn server.
*   **Xử lý lỗi (Error Handling):** Sử dụng `fmt.Errorf("context: %w", err)` để bọc lỗi và duy trì chuỗi lỗi (error chain), giúp việc debug trong log trở nên rõ ràng.
*   **Concurrency & Circuit Breaker:** Sử dụng goroutines để xử lý hàng đợi email và kỹ thuật "Circuit Breaker" để tạm ngưng gửi thư nếu nhà cung cấp (như AWS SES) gặp sự cố liên tục.
*   **Internationalization (i18n):** Frontend sử dụng **LinguiJS** với cơ chế biên dịch (compile) sang file `.po`, giúp bản địa hóa giao diện một cách chuyên nghiệp.

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng đi của một chiến dịch Email (Broadcast) hoặc Email tự động (Automation):

1.  **Giai đoạn Khởi tạo (Initialization):** Khi server chạy, hệ thống kiểm tra phiên bản DB. Nếu có thay đổi, nó sẽ chạy `System Migrations` trước, sau đó lặp qua từng Workspace để chạy `Workspace Migrations`.
2.  **Luồng Tạo nội dung:** Người dùng sử dụng **Visual Email Builder** (kết hợp MJML và Tiptap) để thiết kế template. Dữ liệu được lưu dưới dạng JSON/MJML.
3.  **Luồng Xử lý Phân đoạn (Segmentation):** Hệ thống quét các Contact trong database dựa trên các điều kiện (filter) mà người dùng thiết lập qua UI để tạo danh sách người nhận.
4.  **Luồng Gửi thư (Orchestration):**
    *   `Broadcast Orchestrator` sẽ chia nhỏ danh sách người nhận thành các batch.
    *   Sử dụng `Rate Limiter` để kiểm soát tốc độ gửi (tránh bị đánh dấu spam).
    *   Gọi qua `Email Provider Service` để đẩy thư đến Amazon SES/Mailgun...
5.  **Luồng Theo dõi (Tracking):** Khi khách hàng mở/click email, một yêu cầu sẽ gửi về `Public Handler`. Hệ thống đẩy sự kiện vào `Task Scheduler` để cập nhật thống kê vào bảng `analytics` và `message_history` không đồng bộ.

**Tổng kết:** Notifuse là một dự án Go chuẩn mực, áp dụng các kỹ thuật backend chuyên sâu để giải quyết bài toán hiệu suất và độ tin cậy trong việc gửi email quy mô lớn, đồng thời cung cấp giao diện quản trị (React) cực kỳ tinh tế và dễ dùng.