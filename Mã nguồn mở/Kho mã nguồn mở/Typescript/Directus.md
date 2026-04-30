Dựa trên các tệp tin và cấu trúc mã nguồn được cung cấp, dưới đây là phân tích chi tiết về dự án **Directus** — một nền tảng dữ liệu thời gian thực (Data Platform) và Headless CMS hàng đầu.

### 1. Công nghệ cốt lõi (Core Technology)

Directus được xây dựng dưới dạng một **pnpm monorepo**, sử dụng các công nghệ hiện đại nhất trong hệ sinh thái JavaScript/TypeScript:

*   **Ngôn ngữ chính:** **TypeScript (77.5%)** đảm bảo an toàn kiểu dữ liệu trên toàn bộ hệ thống.
*   **Backend (API):** Chạy trên **Node.js** sử dụng framework **Express**. Điểm mạnh nhất là việc sử dụng **Knex.js** làm Query Builder, cho phép hỗ trợ đa cơ sở dữ liệu (PostgreSQL, MySQL, SQLite, Oracle, MS-SQL, CockroachDB).
*   **Frontend (App):** Sử dụng **Vue.js 3** kết hợp với **Vite** để đóng gói và **Pinia** để quản lý trạng thái. Giao diện được thiết kế theo hướng "no-code" cho người dùng cuối.
*   **AI Integration:** Tích hợp sâu các mô hình ngôn ngữ lớn (LLM) từ OpenAI, Anthropic, Google thông qua **Vercel AI SDK**. Hỗ trợ giao thức **MCP (Model Context Protocol)**.
*   **Real-time:** Sử dụng **WebSockets** để cập nhật dữ liệu và cộng tác thời gian thực (Collaborative Editing).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Directus tuân theo triết lý **"Database-First"** và **"Unopinionated"**:

*   **Introspection (Nội soi schema):** Thay vì sở hữu dữ liệu của bạn, Directus quét (inspect) schema cơ sở dữ liệu SQL hiện có và tự động tạo ra các API REST và GraphQL tương ứng. Nó không thay đổi cấu trúc bảng của bạn trừ khi bạn yêu cầu.
*   **Layered Architecture:** 
    *   **Controller Layer:** Xử lý các endpoint và điều hướng.
    *   **Service Layer:** Chứa logic nghiệp vụ cốt lõi (ví dụ: `ItemsService`, `UsersService`, `FilesService`).
    *   **Database Layer:** Chuyển đổi các truy vấn trừu tượng thành SQL thông qua Knex.
*   **Extensibility (Khả năng mở rộng):** Mọi thứ đều có thể được mở rộng thông qua các **Extensions** (Hooks, Endpoints, Layouts, Interfaces, Displays). Đây là kiến trúc module hóa cực cao.
*   **Storage Agnostic:** Trừu tượng hóa việc lưu trữ tệp qua các driver (S3, GCS, Azure, Local) giúp hệ thống linh hoạt trên mọi hạ tầng đám mây.

### 3. Các kỹ thuật chính (Key Techniques)

*   **AST (Abstract Syntax Tree) Query:** Directus chuyển đổi các tham số truy vấn từ URL (filter, sort, fields) thành một cấu trúc cây AST, sau đó mới biên dịch cây này thành câu lệnh SQL an toàn. Kỹ thuật này giúp ngăn chặn SQL Injection và hỗ trợ các truy vấn cực kỳ phức tạp.
*   **Granular RBAC:** Hệ thống phân quyền dựa trên vai trò (Role-Based Access Control) đến từng cấp độ trường (field) và mục dữ liệu (item). Kỹ thuật này sử dụng các "Policies" để lọc dữ liệu ngay từ tầng truy vấn cơ sở dữ liệu.
*   **Event-Driven (Flows):** Một hệ thống tự động hóa dựa trên sự kiện. Kỹ thuật này sử dụng `Emitter` để kích hoạt các hành động (Actions) hoặc lọc dữ liệu (Filters) trước/sau khi thay đổi DB.
*   **Schema Snapshot & Migration:** Kỹ thuật cho phép "chụp ảnh" toàn bộ cấu trúc DB vào tệp JSON và áp dụng nó sang một instance khác, hỗ trợ quy trình CI/CD cho hạ tầng dữ liệu.
*   **AI Context Formatting:** Kỹ thuật tự động định dạng schema và dữ liệu hiện tại thành context (XML-based) để gửi cho AI, giúp AI hiểu được cấu trúc dữ liệu của dự án mà không cần lập trình thủ công.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

**Luồng xử lý một Request API:**
1.  **Khởi tạo (Bootstrap):** Kiểm tra kết nối DB, nạp schema vào bộ nhớ đệm (Cache), và nạp các Extensions.
2.  **Middleware:** 
    *   Xác thực (Authentication) qua Token hoặc Session.
    *   Kiểm tra Rate Limit.
    *   Phân tích cú pháp truy vấn (Sanitize Query).
3.  **Service Processing:** 
    *   Yêu cầu được chuyển đến Service tương ứng (ví dụ: `ItemsService`).
    *   Service kiểm tra quyền hạn của người dùng đối với bảng/trường đó.
    *   Tạo AST từ yêu cầu.
4.  **Database Interaction:** Knex.js thực thi SQL dựa trên AST.
5.  **Output:** 
    *   Emitter kích hoạt các "Action Hook" nếu có.
    *   Dữ liệu được trả về theo định dạng JSON hoặc GraphQL.

**Luồng AI Chat:**
1.  Người dùng gửi câu hỏi từ App Dashboard.
2.  Hệ thống nạp cấu hình AI từ Settings.
3.  Directus thu thập context (đang ở trang nào, đang chọn item nào).
4.  Tất cả được đóng gói thành một System Prompt lớn và gửi tới LLM.
5.  AI có thể gọi ngược lại các **Tools** (ví dụ: đọc schema, cập nhật item) để thực thi hành động trực tiếp trên hệ thống của người dùng.

**Kết luận:** Directus là một hệ thống "máy ảo" mạnh mẽ nằm trên cơ sở dữ liệu SQL, kết hợp giữa tính linh hoạt của mã nguồn mở và sự tiện dụng của các công cụ no-code hiện đại.