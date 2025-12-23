Dựa trên mã nguồn và cấu trúc thư mục của dự án **OneUptime**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và kỹ thuật của hệ thống này:

### 1. Công nghệ cốt lõi (Core Technologies)
OneUptime là một nền tảng "Full-stack Observability" hiện đại, sử dụng các công nghệ mạnh mẽ nhất trong hệ sinh thái JavaScript/TypeScript:

*   **Ngôn ngữ:** TypeScript (chiếm >90%) được sử dụng xuyên suốt từ Frontend đến Backend, đảm bảo an toàn về kiểu dữ liệu (Type-safety).
*   **Frontend:** React, Tailwind CSS, Esbuild (để build nhanh hơn), EJS (cho các template phía server).
*   **Backend:** Node.js, Express.
*   **Cơ sở dữ liệu (Đa dạng hóa theo mục đích):**
    *   **PostgreSQL:** Lưu trữ dữ liệu quan hệ (User, Project, Config) thông qua **TypeORM**.
    *   **ClickHouse:** Chuyên dụng để lưu trữ dữ liệu hiệu suất cao như Logs, Metrics, Spans (Telemetry) và Analytics.
    *   **Redis:** Dùng để làm Cache, quản lý hàng đợi (Queues) và điều phối các tác vụ phân tán.
*   **Cơ sở hạ tầng & DevOps:**
    *   **Docker & Docker Compose:** Container hóa toàn bộ dịch vụ.
    *   **Kubernetes (Helm Charts):** Hỗ trợ triển khai quy mô lớn (Production).
    *   **Nginx:** Đóng vai trò là Ingress/Reverse Proxy.
*   **Giám sát & Telemetry:** Hỗ trợ giao thức OpenTelemetry (OTel), Fluentd, Fluent Bit và Syslog.
*   **Tích hợp AI:** Sử dụng Hugging Face và các LLM server để hỗ trợ phân tích sự cố tự động.

---

### 2. Tư duy kiến trúc (Architectural Thinking)
Dự án được thiết kế theo mô hình **Monorepo** với kiến trúc hướng dịch vụ (Service-Oriented Architecture):

*   **Cấu trúc Monorepo:** Tất cả các dịch vụ (Accounts, Dashboard, API, Probes, Workers) nằm trong một kho lưu trữ duy nhất nhưng độc lập về logic triển khai.
*   **Dịch vụ dùng chung (Common Module):** Thư mục `Common/` chứa các Model, Type, và Utility được dùng chung cho cả Frontend và Backend. Điều này giúp tránh lặp lại mã (DRY) và đồng bộ hóa logic nghiệp vụ.
*   **Tách biệt Ingest và Process:**
    *   Các dịch vụ `Ingest` (ProbeIngest, Telemetry, IncomingRequestIngest) chỉ làm nhiệm vụ nhận dữ liệu nhanh nhất có thể.
    *   Việc xử lý nặng được đẩy xuống `Worker` và `Workflow` thông qua hàng đợi (Redis Queue).
*   **Kiến trúc Đa thuê chủ (Multi-tenancy):** Hệ thống được thiết kế ngay từ đầu để hỗ trợ nhiều "Project" riêng biệt, mỗi dự án có cấu hình, người dùng và dữ liệu giám sát riêng.
*   **Tư duy Mở rộng (Scalability):** Tách biệt dữ liệu giao dịch (Postgres) và dữ liệu quan sát (ClickHouse) cho phép hệ thống xử lý hàng tỷ bản ghi logs/metrics mà không làm chậm ứng dụng chính.

---

### 3. Kỹ thuật chính (Main Technical Skills)
Dự án thể hiện các kỹ thuật lập trình và hệ thống cao cấp:

*   **Abstract Database Layer:** Sử dụng TypeORM cho Postgres và một lớp wrapper tùy chỉnh cho ClickHouse để quản lý schema và thực thi truy vấn analytics phức tạp.
*   **Real-time Processing:** Sử dụng Socket.io và Redis để cập nhật trạng thái hạ tầng lên Dashboard ngay lập tức khi có sự cố.
*   **Hệ thống Workflow tự động:** Cho phép người dùng tạo các luồng tự động (ví dụ: Nếu Monitor chết -> Chạy Script JavaScript -> Gửi Slack).
*   **Infrastructure as Code (IaC):** Tự động tạo mã cho Terraform Provider, giúp người dùng quản lý hạ tầng OneUptime bằng code.
*   **Security & Compliance:** Tích hợp sẵn SSO (SAML/Okta), xác thực 2 lớp (2FA/WebAuthn), và quản lý chứng chỉ SSL tự động qua Let's Encrypt.
*   **Global Probing:** Kỹ thuật triển khai các `Probe` ở nhiều vị trí địa lý khác nhau để kiểm tra độ trễ và tính khả dụng của website từ khắp nơi trên thế giới.

---

### 4. Tóm tắt luồng hoạt động (Operational Workflow)

1.  **Thu thập (Ingestion):**
    *   **Probes:** Định kỳ gửi request (HTTP, Port, Ping) đến tài nguyên của khách hàng.
    *   **Agent/OTel:** Gửi trực tiếp Logs/Metrics từ server khách hàng về hệ thống.
2.  **Tiếp nhận & Xếp hàng:** Dữ liệu thô gửi về các endpoint `Ingest`, sau đó được đẩy vào Redis Queue.
3.  **Xử lý (Processing):**
    *   `Worker` lấy dữ liệu từ Queue, so sánh với các tiêu chí (Criteria) mà khách hàng đã thiết lập.
    *   Nếu phát hiện bất thường (ví dụ: HTTP 500), `Worker` sẽ tạo `Incident` hoặc `Alert`.
4.  **Thông báo (Notification):** Dịch vụ `Notification` sẽ kiểm tra lịch trực (On-Call) và gửi cảnh báo qua Email, SMS, Slack hoặc gọi điện thoại trực tiếp.
5.  **Hiển thị & Công khai:**
    *   Trạng thái mới được cập nhật lên `Dashboard` cho nội bộ team kỹ thuật.
    *   Đồng thời cập nhật lên `Status Page` công khai để thông báo cho khách hàng của họ.
6.  **Tự động hóa (Automation):** Nếu có `Workflow` được thiết lập, hệ thống sẽ thực thi các hành động tự động như chạy script phục hồi hoặc gọi webhook bên thứ ba.

### Kết luận:
OneUptime không chỉ là một công cụ giám sát đơn thuần mà là một hệ thống **Enterprise-grade** toàn diện. Sự kết hợp giữa **TypeScript**, **ClickHouse** và tư duy **Monorepo** giúp dự án này vừa dễ bảo trì cho các contributor, vừa đủ mạnh mẽ để thay thế các giải pháp trả phí như Datadog hay PagerDuty.