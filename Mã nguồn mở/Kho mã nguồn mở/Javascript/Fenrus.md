Dựa trên mã nguồn của dự án **Fenrus**, một ứng dụng dashboard cá nhân hiện đại, dưới đây là phân tích chi tiết về các khía cạnh kỹ thuật:

### 1. Công nghệ cốt lõi (Core Tech Stack)
Fenrus là một sự kết hợp thú vị giữa hiệu năng của **.NET** và tính linh hoạt của **JavaScript**:

*   **Framework chính:** **.NET 7.0** với **Blazor Server Side**. Đây là lựa chọn chiến lược để xây dựng giao diện tương tác thời gian thực mà không cần viết quá nhiều API REST, nhờ sử dụng **SignalR** để đồng bộ hóa trạng thái giữa server và browser.
*   **Cơ sở dữ liệu:** **LiteDB** – một NoSQL database nhúng (embedded) viết bằng C#, rất nhẹ và không yêu cầu cài đặt server DB riêng (giống SQLite nhưng cho dữ liệu dạng tài liệu).
*   **Công cụ thực thi Smart Apps:** **Jint**. Đây là điểm đặc biệt nhất: Fenrus chạy các đoạn mã JavaScript của các "Smart Apps" ngay trên Server .NET để lấy dữ liệu (như thông tin từ Plex, Pi-hole, Docker) trước khi đẩy kết quả xuống giao diện.
*   **Giao diện & Style:** **SCSS/Sass** được biên dịch và gom nhóm (bundle) tự động bằng **LigerShark.WebOptimizer**.
*   **Hệ sinh thái nhúng:** Sử dụng các thư viện mạnh mẽ như **SSH.NET** (cho terminal), **Docker.DotNet** (quản lý container), **MailKit/MimeKit** (xử lý email).

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của Fenrus hướng tới sự **tự chủ (self-hosted)** và **mở rộng qua plugin**:

*   **Kiến trúc Plugin (Smart App Architecture):** Thay vì code cứng logic cho từng ứng dụng (Plex, Sonarr...), Fenrus định nghĩa một cấu trúc thư mục trong `/Apps/Smart/`. Mỗi ứng dụng gồm một file `app.json` (cấu hình) và `code.js` (logic xử lý). Điều này cho phép cộng đồng thêm ứng dụng mới mà không cần biên dịch lại toàn bộ dự án C#.
*   **Hybrid Rendering:** Sử dụng Blazor cho các trang quản trị/cấu hình phức tạp để tận dụng tính Type-safety của C#, nhưng lại dùng JavaScript thuần ở trang Dashboard chính để tối ưu hóa hiệu ứng đồ họa (như các hình nền động Vanta.js) và giảm tải cho Server Hub.
*   **Bảo mật đa tầng:** Dữ liệu nhạy cảm (mật khẩu SSH, API Keys) được đóng gói trong kiểu dữ liệu `EncryptedString` và tự động mã hóa/giải mã khi lưu vào LiteDB bằng khóa riêng được tạo lúc cài đặt.
*   **Multi-tenancy (Đa người dùng):** Hệ thống phân tách rõ rệt giữa `SystemSettings` (toàn hệ thống) và `UserSettings` (cá nhân), hỗ trợ phân quyền Admin và chế độ Guest Dashboard.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **JS Interop & Sandbox:** Kỹ thuật cầu nối giữa C# và JS được sử dụng triệt để. Jint tạo ra một môi trường "Sandbox" an toàn để thực thi mã JS của Smart Apps, giới hạn các hàm mà JS có thể gọi (như `fetch`, `log`, `liveStats`) thông qua class `AppHelper`.
*   **Worker Pattern:** Sử dụng các lớp Worker chạy nền (`CalendarEventWorker`, `MailWorker`) để quét thông tin lịch và email định kỳ, sau đó gửi thông báo trực tiếp đến người dùng qua WebSocket.
*   **Proxy & Sanitize:** Hệ thống có `ProxyController` để vượt qua lỗi CORS khi Smart Apps cần lấy dữ liệu từ các server khác, đồng thời tích hợp **DOMPurify** (trong JS) và **Markdig** (trong C#) để làm sạch nội dung HTML/Markdown, tránh tấn công XSS.
*   **Kỹ thuật Virtualization:** Trong quản lý file (`VirtualizeList.js`), hệ thống chỉ render các item hiển thị trên màn hình, giúp xử lý hàng ngàn file trong Cloud Drive mà không làm chậm trình duyệt.

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Khởi động (Startup):** `StartUpHelper` kiểm tra thư mục dữ liệu, khởi tạo khóa mã hóa và nạp danh sách plugin Apps từ ổ đĩa vào bộ nhớ.
2.  **Xác thực (Authentication):** Hỗ trợ cả đăng nhập truyền thống (Forms Auth) lẫn **OAuth/OpenID Connect** (SSO). Người dùng đầu tiên đăng ký sẽ mặc định là Admin.
3.  **Xử lý Smart App:**
    *   Dashboard gửi yêu cầu cập nhật App qua SignalR hoặc HTTP.
    *   Server khởi tạo engine Jint -> Nạp `code.js` của App -> Thực thi hàm `status()`.
    *   App có thể gọi `fetch()` (được C# proxy) để lấy data từ API bên ngoài.
    *   Kết quả trả về được định dạng thành HTML/JSON và đẩy xuống giao diện.
4.  **Terminal/Log:** Khi người dùng mở terminal SSH hoặc Docker, một WebSocket được thiết lập. Server .NET đóng vai trò là "cầu nối" (bridge), chuyển tiếp từng byte dữ liệu giữa browser (Xterm.js) và server đích, đảm bảo mật khẩu không bao giờ lộ ra phía client.
5.  **Thông báo thời gian thực:** Các sự kiện từ Worker (ví dụ: có email mới) được `NotificationHelper` phát tín hiệu, SignalR sẽ "bắn" thông báo đó đến chính xác browser của người dùng đang kết nối.

### Tổng kết
Fenrus không chỉ là một dashboard đơn thuần mà là một **Application Portal**. Kỹ thuật lập trình ở đây rất chuyên nghiệp trong việc kết hợp các ngôn ngữ khác nhau để tận dụng ưu điểm của từng loại: C# cho quản lý hệ thống/bảo mật và JS cho tính linh hoạt/plugin.