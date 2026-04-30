Dưới đây là phân tích chi tiết về dự án **Probo** (getprobo/probo) - một nền tảng quản trị tuân thủ (GRC - Governance, Risk, and Compliance) mã nguồn mở dành cho startup.

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án sử dụng mô hình Monorepo với sự kết hợp mạnh mẽ giữa hiệu suất của Go và tính hiện đại của hệ sinh thái TypeScript:

*   **Backend (Go):** Sử dụng Go (v1.21+) làm ngôn ngữ chủ đạo cho server API (`probod`). Hệ thống API được xây dựng dựa trên **GraphQL** (sử dụng `gqlgen`) để cung cấp giao tiếp kiểu dữ liệu chặt chẽ (strongly-typed).
*   **Frontend (React & TypeScript):** Gồm hai ứng dụng chính:
    *   `apps/console`: Bảng điều khiển quản trị nội bộ.
    *   `apps/trust`: Cổng thông tin tin cậy (Trust Center) dành cho khách hàng bên ngoài.
    *   **Relay:** Framework của Meta được dùng để fetch dữ liệu GraphQL, giúp tối ưu hóa hiệu suất và quản lý dữ liệu phức tạp.
*   **Cơ sở dữ liệu:** **PostgreSQL** là kho lưu trữ chính. Dự án sử dụng `pgx/v5` để tối ưu kết nối và truy vấn.
*   **Hệ sinh thái quan sát (Full-stack Observability):** Tích hợp sâu **OpenTelemetry** với bộ công cụ của Grafana Labs: Prometheus (metrics), Loki (logs), Tempo (traces).
*   **Lưu trữ & Hạ tầng:** Sử dụng **SeaweedFS** (tương thích S3) để lưu trữ tài liệu minh chứng, **ChromeDP** (Headless Chrome) để xuất báo cáo PDF, và **Keycloak** cho xác thực OIDC/SAML.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Probo tập trung vào tính minh bạch, khả năng mở rộng và cô lập dữ liệu:

*   **Kiến trúc API-first & Codegen:** Hầu hết mã nguồn từ server (Go types) đến client (Relay fragments) đều được sinh tự động (generate) từ file định nghĩa GraphQL. Điều này đảm bảo tính đồng bộ tuyệt đối giữa frontend và backend.
*   **Định danh toàn cầu (Global Identifiers - GID):** Probo sử dụng hệ thống GID để quản lý thực thể. Mỗi ID mang thông tin về loại thực thể và ID tổ chức (TenantID), giúp việc tra cứu và phân quyền đa thuê (multi-tenancy) trở nên an toàn và hiệu quả.
*   **Phân quyền dựa trên Policy (IAM-style):** Thay vì RBAC đơn giản, hệ thống sử dụng một engine chính sách với các điều kiện (Conditions) phức tạp, cho phép kiểm soát truy cập đến từng tài liệu hoặc control cụ thể.
*   **Lớp trừu tượng hóa dữ liệu (Coredata):** Cung cấp các công cụ như `Scoper` để tự động lọc dữ liệu theo tổ chức, đảm bảo một tổ chức không bao giờ thấy dữ liệu của tổ chức khác (Tenant isolation).

### 3. Các kỹ thuật chính (Key Techniques)

*   **Worker Pattern:** Backend sử dụng các Go workers để xử lý tác vụ nền (như thu thập minh chứng, quét lỗ hổng). Kỹ thuật `FOR UPDATE SKIP LOCKED` trong SQL được dùng để đảm bảo nhiều worker có thể chạy song song mà không tranh chấp dữ liệu.
*   **Snapshot-to-Document Migration:** Hệ thống có cơ chế chuyển đổi các trạng thái kiểm tra (snapshots) thành các tài liệu chính thức, phục vụ cho việc kiểm toán (Audit-ready).
*   **Headless Browser Rendering:** Sử dụng headless Chrome để render các trang web phức tạp thành file PDF minh chứng, đảm bảo nội dung báo cáo giống hệt những gì người dùng thấy trên màn hình.
*   **Automation với n8n:** Cung cấp một node cộng đồng cho n8n (`packages/n8n-node`), cho phép người dùng tự động hóa quy trình tuân thủ bằng cách kết nối Probo với hàng trăm ứng dụng khác (Slack, Jira, AWS...).
*   **Headless Cookie Banner:** Một package riêng biệt (`packages/cookie-banner`) được thiết kế để quản lý sự chấp thuận quyền riêng tư một cách linh hoạt, không phụ thuộc giao diện.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Thiết lập Khung (Framework Setup):** Người dùng chọn các tiêu chuẩn tuân thủ (SOC2, GDPR...) từ thư viện JSON có sẵn trong `apps/console/public/data/frameworks/`.
2.  **Đánh giá rủi ro & Control:** Hệ thống liệt kê các "Control" cần thực hiện. Người dùng liên kết các Control này với các "Risk" (rủi ro) và "Measure" (biện pháp đo lường).
3.  **Thu thập minh chứng (Evidence Collection):**
    *   **Tự động:** Qua các Connectors (GitHub, Google Workspace) hoặc các workflow n8n.
    *   **Thủ công:** Người dùng tải file lên, hệ thống mô tả file bằng LLM (OpenAI/Anthropic) và lưu trữ vào SeaweedFS.
4.  **Kiểm soát tài liệu (Document Management):** Các chính sách (policies) được soạn thảo qua rich-text editor (ProseMirror), ký điện tử và lưu trữ phiên bản.
5.  **Công khai Tin cậy (Trust Center):** Khi đã tuân thủ, tổ chức công khai trạng thái qua `apps/trust`. Tại đây, đối tác có thể yêu cầu truy cập tài liệu bảo mật, ký NDA tự động.
6.  **Giám sát liên tục:** Các worker định kỳ kiểm tra trạng thái các Control. Nếu có sai lệch, hệ thống sẽ tạo "Finding" (phát hiện lỗi) và gửi thông báo qua email/Slack.

**Kết luận:** Probo không chỉ là một công cụ quản lý file, mà là một hệ thống tự động hóa quy trình tuân thủ khép kín, biến các tiêu chuẩn khô khan thành dữ liệu có thể theo dõi và chứng minh được trong thời gian thực.