Dựa trên thông tin từ cấu trúc thư mục và các tệp cấu hình của dự án **Budibase**, dưới đây là phân tích chi tiết về các khía cạnh công nghệ, kiến trúc và kỹ thuật của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ & Framework:**
    *   **TypeScript (62.6%):** Sử dụng xuyên suốt cho cả Frontend và Backend để đảm bảo an toàn về kiểu.
    *   **Svelte (32.3%):** Framework chính cho phần Frontend (Builder và Client runtime). Dự án đang trong lộ trình nâng cấp lên **Svelte 5**.
    *   **Node.js & Koa:** Runtime và Web Framework phía Server, nổi tiếng với tính nhẹ và hiệu suất cao thông qua middleware.
*   **Hệ quản trị cơ sở dữ liệu & Lưu trữ:**
    *   **CouchDB:** Cơ sở dữ liệu NoSQL chính, lưu trữ metadata của các ứng dụng và dữ liệu nội bộ.
    *   **Redis:** Sử dụng làm bộ nhớ đệm (cache), quản lý session và hàng đợi công việc (queuing).
    *   **MinIO:** Hệ thống lưu trữ đối tượng tương thích với S3, dùng cho các tệp đính kèm và tài sản tĩnh.
    *   **Structured Query Server (SQS):** Một thành phần đặc thù của Budibase giúp thực thi truy vấn SQL trên nền tảng CouchDB.
*   **AI & Hạ tầng:**
    *   **LiteLLM:** Proxy/Gateway để kết nối với nhiều mô hình ngôn ngữ lớn (LLM) khác nhau theo cách thống nhất.
    *   **isolated-vm:** Thư viện cho phép thực thi mã JavaScript của người dùng trong một môi trường sandbox an toàn trên Server.

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Kiến trúc Monorepo (Lerna & Nx):** Dự án chia nhỏ thành các package chuyên biệt để tái sử dụng code:
    *   `backend-core`, `frontend-core`: Logic dùng chung cho từng phía.
    *   `shared-core`: Logic và kiểu dữ liệu dùng chung cho cả Client/Server.
    *   `builder`: Môi trường IDE (No-code editor).
    *   `client`: Runtime engine để chạy ứng dụng đã thiết kế.
    *   `server` & `worker`: Phân tách giữa xử lý API và xử lý tác vụ nền (background jobs).
*   **Metadata-Driven (Hướng siêu dữ liệu):** Budibase không biên dịch ứng dụng ra mã nguồn thô. Thay vào đó, nó tạo ra một lược đồ (schema) JSON mô tả UI, dữ liệu và automation. Package `client` sẽ đóng vai trò là "interpreter" để dựng ứng dụng thời gian thực từ schema đó.
*   **AI-Native Operations:** Kiến trúc mới nhấn mạnh vào "AI Agents". Hệ thống tích hợp khả năng RAG (Retrieval Augmented Generation), cho phép Agent truy cập vào Knowledge Base (từ SharePoint, tệp tin nội bộ) để thực hiện các nghiệp vụ tự động.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Sandboxed Execution:** Kỹ thuật thực thi mã tùy chỉnh của người dùng thông qua `isolated-vm` để ngăn chặn các cuộc tấn công injection hoặc tiêu tốn tài nguyên hệ thống quá mức.
*   **Unified Data Integration:** Cung cấp một lớp trừu tượng (abstraction layer) cho phép Budibase giao tiếp với hàng chục loại database khác nhau (PostgreSQL, MongoDB, MySQL, v.v.) bằng một giao diện duy nhất.
*   **Reactive Stores:** Sử dụng Svelte Stores mạnh mẽ để quản lý trạng thái phức tạp trong trình kéo thả (Builder), đảm bảo giao diện cập nhật ngay lập tức khi người dùng thay đổi thuộc tính.
*   **Event-Driven Automations:** Sử dụng hàng đợi Redis và worker để xử lý các chuỗi logic (Automation flows) một cách bất đồng bộ, giúp Server chính không bị treo khi xử lý các tác vụ nặng như gửi email hàng loạt hoặc xử lý dữ liệu lớn.

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Giai đoạn Thiết kế (Design Time):**
    *   Người dùng sử dụng `builder` (Svelte) để kéo thả UI và thiết lập logic.
    *   Metadata được gửi về `server` (Koa) và lưu vào `CouchDB`.
2.  **Giai đoạn Thực thi (Runtime):**
    *   Khi người dùng truy cập ứng dụng, `client` tải schema từ `server`.
    *   `client` engine dựng giao diện và kết nối các binding dữ liệu.
3.  **Luồng AI & Automation:**
    *   Khi một sự kiện kích hoạt (webhook, cron, hoặc chat với Agent), `server` đẩy yêu cầu vào `Redis`.
    *   `worker` nhận việc, nếu là Agent, nó sẽ gọi qua `LiteLLM` để lấy phản hồi từ mô hình AI.
    *   Nếu Agent cần dữ liệu, nó thực hiện các "Tools" (truy vấn DB, đọc file) thông qua API nội bộ của Budibase.
4.  **Xử lý dữ liệu ngoại vi:**
    *   Các yêu cầu truy vấn đến DB bên ngoài (ví dụ Postgres) được `server` xử lý thông qua các trình điều khiển (integrations) đã được tối ưu hóa, sau đó format kết quả trả về cho Frontend hiển thị dưới dạng bảng hoặc biểu đồ.

### Tổng kết
Budibase là một hệ thống phức tạp kết hợp giữa **Low-code truyền thống** và **AI Agent hiện đại**. Điểm mạnh nhất của nó nằm ở khả năng **tự lưu trữ (self-hosting)** với độ bảo mật cao và khả năng **mở rộng thông qua code** (JavaScript/TypeScript) dù vẫn giữ được sự đơn giản của một công cụ No-code.