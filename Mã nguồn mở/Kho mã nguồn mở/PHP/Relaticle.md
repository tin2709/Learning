Dựa trên các tài liệu và cấu trúc mã nguồn của **Relaticle**, đây là phân tích chi tiết về dự án CRM mã nguồn mở dành cho kỷ nguyên AI này:

### 1. Công nghệ cốt lõi (Core Technologies)
Relaticle sử dụng những công nghệ mới nhất trong hệ sinh thái PHP/Laravel (tính đến đầu năm 2025):

*   **Ngôn ngữ & Runtime**: **PHP 8.4+** (tận dụng các tính năng mới nhất như *property promotion*, *strict types*).
*   **Framework chính**: **Laravel 12** (phiên bản mới nhất).
*   **Giao diện quản trị (TUI/Admin Panel)**: **Filament v5**. Đây là một framework SDUI (Server-Driven UI) cực kỳ mạnh mẽ, giúp xây dựng giao diện quản lý nhanh chóng.
*   **Reactivity**: **Livewire v4** và **Alpine.js v3** cho phép tạo các thành phần giao diện động mà không cần viết nhiều JavaScript.
*   **AI Native**: Tích hợp **MCP (Model Context Protocol)** server với hơn 30 công cụ sẵn có, cho phép các AI Agent như Claude hay GPT tương tác trực tiếp với dữ liệu CRM.
*   **Cơ sở dữ liệu**: **PostgreSQL 17+** (Dự án tuyên bố sử dụng độc quyền Postgres, không hỗ trợ MySQL/SQLite để tận dụng tối đa tính năng của Postgres).
*   **Kiểm thử & Chất lượng**: **Pest v4** (testing), **PHPStan** (static analysis), **Rector** (automated refactoring).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của Relaticle được thiết kế để phục vụ cả người dùng cuối và các AI Agent:

*   **Action-Oriented Architecture**: Mọi logic nghiệp vụ ghi dữ liệu (Create, Update, Delete) đều được đóng gói trong các **Action Classes** (`app/Actions`). Các lớp này là "nguồn chân lý" duy nhất, giúp tái sử dụng logic giữa giao diện Web, REST API và MCP Tools.
*   **Agent-Native Infrastructure**: Thay vì coi AI là một tính năng bổ sung, Relaticle xây dựng hạ tầng để AI có thể "đọc hiểu" schema và thực thi tác vụ. MCP server giúp AI khám phá các công cụ và tài nguyên của hệ thống một cách tự động.
*   **Multi-Team Isolation**: Hệ thống phân quyền 5 lớp (Layered Authorization) với cơ chế cô lập dữ liệu theo Team, đảm bảo an toàn cho mô hình SaaS đa khách hàng.
*   **Package-based Modularity**: Các tính năng phức tạp như `ImportWizard`, `SystemAdmin`, `Documentation` được tách thành các gói (packages) riêng trong thư mục `packages/`, giúp mã nguồn sạch sẽ và dễ bảo trì.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Dynamic Custom Fields**: Hỗ trợ 22 loại trường tùy chỉnh mà không cần chạy migration database. Hệ thống tự động xử lý việc lưu trữ, mã hóa (encryption) và lọc dữ liệu trên các trường này.
*   **Tenant Scoping**: Sử dụng Middleware (`SetApiTeamContext`) và Global Scopes để tự động lọc dữ liệu theo Team mà không cần viết điều kiện `where team_id` thủ công ở mọi nơi.
*   **AI Record Summary**: Dịch vụ `RecordSummaryService` sử dụng LLM (như Anthropic) để tự động tạo tóm tắt cho các bản ghi (Công ty, Cơ hội, Liên hệ), giúp người dùng nắm bắt thông tin nhanh chóng.
*   **FlowForge (Kanban Boards)**: Một kỹ thuật xây dựng bảng Kanban tùy chỉnh cho việc quản lý Task và Opportunity với khả năng kéo thả và tính toán vị trí theo số thập phân (Decimal Position) để tránh xung đột thứ tự.
*   **Polymorphic Relationships**: Sử dụng rộng rãi cho `Notes` và `Tasks` để một ghi chú hoặc nhiệm vụ có thể gắn liền với nhiều loại thực thể khác nhau (Company, People, Opportunity).

---

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Luồng Yêu cầu (Request Flow)**:
    *   **Người dùng Web**: Truy cập qua Filament Dashboard -> Livewire Component gọi Action -> Action thực thi logic và lưu vào Model.
    *   **AI Agent**: Gửi yêu cầu qua MCP Server -> MCP Tool gọi Action tương ứng -> Trả về kết quả JSON đã được định dạng cho AI hiểu.
    *   **Hệ thống bên thứ ba**: Gọi REST API v1 (được document bằng Scribe) -> Controller gọi Action.
2.  **Luồng Dữ liệu (Data Flow)**:
    *   Dữ liệu được lưu trữ vào PostgreSQL. Các trường tùy chỉnh được lưu vào bảng `custom_field_values`.
    *   Các tác vụ nặng (như Fetch Favicon, Gửi email, Import dữ liệu) được đẩy vào hàng đợi (Queue) xử lý bởi **Laravel Horizon**.
3.  **Luồng Cài đặt & Khởi tạo**:
    *   Sử dụng lệnh `composer app-install` để tự động chạy `relaticle:install`.
    *   Hệ thống có cơ chế `OnboardSeed` để tạo dữ liệu mẫu giúp người dùng mới làm quen ngay lập tức.

**Đánh giá**: Relaticle là một dự án CRM "tương lai", nơi ranh giới giữa người dùng người thật và AI Agent được xóa nhòa thông qua một kiến trúc logic tập trung và giao thức MCP tiên phong.