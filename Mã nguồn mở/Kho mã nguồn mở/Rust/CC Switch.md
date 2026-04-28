Dựa trên thông tin từ kho lưu trữ và các tệp tài liệu đi kèm, dưới đây là phân tích chi tiết về dự án **CC Switch** - một công cụ quản lý máy để bàn (Desktop) đa nền tảng dành cho các AI CLI (Claude Code, Codex, Gemini CLI, OpenCode, OpenClaw và Hermes Agent).

### 1. Công nghệ cốt lõi (Core Technology)

CC Switch được xây dựng trên một ngăn xếp công nghệ hiện đại, tập trung vào hiệu suất cao và trải nghiệm người dùng bản địa:

*   **Tauri 2.0 (Framework chính):** Sử dụng Rust cho logic backend và Webview (React/TS) cho giao diện. Tauri giúp ứng dụng cực kỳ nhẹ, bảo mật và tiêu tốn ít tài nguyên hơn so với Electron.
*   **Rust (Backend):** Xử lý các tác vụ nặng như:
    *   **Hyper:** Làm nền tảng cho hệ thống Proxy nội bộ (Local Proxy) để chuyển tiếp và biến đổi định dạng API.
    *   **SQLite:** Cơ sở dữ liệu chính để lưu trữ cấu hình nhà cung cấp, MCP servers, và lịch sử phiên.
    *   **Serde:** Để xử lý việc đọc/ghi các tệp cấu hình phức tạp của các công cụ khác (JSON, TOML, YAML, .env).
*   **React 18 & TypeScript (Frontend):** Sử dụng bộ UI **shadcn/ui** và **TailwindCSS** để tạo giao diện hiện đại.
*   **TanStack Query v5:** Quản lý trạng thái bất đồng bộ và đồng bộ hóa dữ liệu giữa UI và Database (SSOT).
*   **Hệ thống Proxy nội bộ:** Chuyển đổi định dạng dữ liệu giữa các giao thức khác nhau (ví dụ: biến đổi Anthropic Messages sang OpenAI Chat Completions).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của CC Switch được thiết kế theo mô hình **Middleware Manager** (Trình quản lý trung gian):

*   **SSOT (Single Source of Truth):** Tất cả dữ liệu được tập trung vào SQLite (`cc-switch.db`). Khi người dùng thay đổi cấu hình trên app, hệ thống sẽ tự động đồng bộ xuống các tệp cấu hình "sống" (live files) của các công cụ CLI tương ứng.
*   **Kiến trúc phân lớp rõ ràng:** 
    *   *Lớp Lệnh (Commands):* Cổng giao tiếp IPC giữa JS và Rust.
    *   *Lớp Dịch vụ (Services):* Chứa logic nghiệp vụ (Xử lý MCP, quản lý Provider).
    *   *Lớp DAO (Data Access Object):* Tương tác trực tiếp với Database.
*   **Kiến trúc Proxy đa tầng:** Proxy không chỉ đơn thuần là chuyển tiếp (forwarding) mà còn bao gồm các bộ lọc (filters), bộ hiệu chỉnh (rectifiers - sửa lỗi chữ ký thinking), và cơ chế ngắt mạch (circuit breaker) để đảm bảo tính sẵn sàng cao (High Availability).
*   **Khả năng mở rộng (Scalability):** Hỗ trợ "Universal Provider", cho phép một cấu hình duy nhất được áp dụng đồng thời cho nhiều ứng dụng (OpenCode, OpenClaw).

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Atomic Writes (Ghi dữ liệu nguyên tử):** Sử dụng kỹ thuật ghi vào tệp tạm rồi mới đổi tên (rename) để đảm bảo tệp cấu hình không bao giờ bị hỏng (corrupted) nếu ứng dụng bị crash giữa chừng.
*   **JSON5 & TOML Parsing:** Xử lý việc chỉnh sửa tệp cấu hình mà vẫn giữ nguyên các chú thích (comments) và định dạng gốc của người dùng (Round-trip editing).
*   **OAuth Device Flow Emulation:** Tự triển khai các luồng xác thực OAuth cho GitHub Copilot và ChatGPT để lấy Token mà không cần thông qua trình duyệt chính thức của các CLI đó.
*   **Virtualized Lists:** Sử dụng `@tanstack/react-virtual` để hiển thị hàng ngàn tin nhắn trong Session Manager mà vẫn đảm bảo độ mượt 60fps.
*   **Deep Linking:** Xử lý giao thức `ccswitch://` để cho phép người dùng nhập cấu hình nhanh chóng từ các trang web hoặc tài liệu chỉ bằng một cú nhấp chuột.
*   **Biến đổi luồng dữ liệu (SSE Stream Transformation):** Kỹ thuật xử lý các gói tin UTF-8 bị chia cắt tại biên (chunk boundary) để tránh lỗi hiển thị ký tự rác (U+FFFD) khi streaming qua Proxy.

### 4. Luồng hoạt động hệ thống (System Operation Flow)

1.  **Khởi động:** 
    *   Hệ thống kiểm tra các công cụ CLI đã cài đặt (Claude, Codex...).
    *   Tự động phát hiện và nhập cấu hình hiện có vào DB (nếu là lần đầu).
    *   Khởi tạo biểu tượng ở khay hệ thống (System Tray).
2.  **Quản lý cấu hình:** Người dùng thêm Provider -> DB lưu trữ -> Khi nhấn "Enable", app thực hiện ghi đè hoặc hợp nhất (merge) vào tệp cấu hình của CLI đó (ví dụ: `~/.claude/settings.json`).
3.  **Local Routing (Proxy Mode):**
    *   CLI tool gửi request đến `localhost:15721`.
    *   Proxy của CC Switch bắt request, kiểm tra Provider nào đang được chọn.
    *   Nếu cần, Proxy sẽ thay đổi Header, biến đổi Payload từ định dạng app này sang app kia (Format Mapping).
    *   Gửi yêu cầu đến máy chủ AI thật.
4.  **Failover (Chuyển vùng lỗi):** Nếu máy chủ A lỗi, Proxy tự động chuyển sang máy chủ B trong danh sách chờ mà người dùng không cần can thiệp hay khởi động lại CLI.
5.  **Đồng bộ hóa đám mây:** Sử dụng WebDAV hoặc thư mục chung (Dropbox/iCloud) để đồng bộ SQLite DB giữa các thiết bị khác nhau.

### Tổng kết
**CC Switch** không chỉ là một giao diện GUI cho các file cấu hình, mà nó là một **Lớp điều phối (Orchestration Layer)** mạnh mẽ, giúp thống nhất trải nghiệm lập trình AI trên Terminal, giải quyết bài toán phân mảnh cấu hình và tối ưu hóa chi phí API thông qua hệ thống Proxy thông minh.