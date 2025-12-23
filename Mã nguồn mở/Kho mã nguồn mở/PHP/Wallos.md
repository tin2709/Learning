Dựa trên mã nguồn và cấu trúc thư mục của kho lưu trữ **Wallos**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của dự án:

### 1. Công nghệ cốt lõi (Tech Stack)

Dự án được xây dựng theo phong cách truyền thống nhưng tối ưu hóa cho việc tự triển khai (self-hosted).

*   **Ngôn ngữ lập trình:**
    *   **PHP 8.3:** Đóng vai trò xử lý logic phía máy chủ (Backend chính).
    *   **JavaScript (Vanilla):** Xử lý tương tác phía người dùng (Frontend) mà không dùng framework nặng như React/Vue, giúp ứng dụng nhẹ và nhanh.
    *   **SQL (SQLite):** Hệ quản trị cơ sở dữ liệu mặc định. Lựa chọn này cực kỳ phù hợp cho các ứng dụng cá nhân vì không cần cài đặt server DB phức tạp, chỉ cần một file `.db`.
*   **Máy chủ & Hạ tầng:**
    *   **Web Server:** Nginx (được cấu hình hỗ trợ cả IPv4 và IPv6).
    *   **Containerization:** Docker & Docker Compose (giúp triển khai nhanh chóng).
    *   **Hệ điều hành cơ sở:** Alpine Linux (nhẹ, bảo mật).
*   **Thư viện bên thứ ba:**
    *   **Chart.js:** Vẽ biểu đồ thống kê tài chính.
    *   **PHPMailer:** Gửi email thông báo.
    *   **OTPHP:** Xử lý mã xác thực hai lớp (2FA/TOTP).
    *   **Fixer API / APILayer:** Lấy tỷ giá hối đoái thực tế.
    *   **AI Integrations:** Hỗ trợ OpenAI (ChatGPT), Google (Gemini) và Local LLM (Ollama).

---

### 2. Các kỹ thuật và Tư duy kiến trúc chính

Kiến trúc của Wallos là sự kết hợp giữa **MPA (Multi-Page Application)** và các **RESTful Endpoints**.

*   **Quản lý Database (Migration System):** Dự án sử dụng một hệ thống migration tự chế (thư mục `/migrations`). Khi ứng dụng khởi chạy hoặc cập nhật, nó sẽ chạy các script PHP được đánh số thứ tự (000001.php, 000002.php...) để thay đổi cấu trúc bảng mà không làm mất dữ liệu người dùng.
*   **Kiến trúc API-Centric:** 
    *   Thư mục `/api`: Chứa các endpoint cung cấp dữ liệu định dạng JSON cho ứng dụng hoặc bên thứ ba.
    *   Thư mục `/endpoints`: Chứa logic xử lý các hành động cụ thể (thêm/sửa/xóa). Điều này tách biệt logic xử lý dữ liệu khỏi giao diện hiển thị (`.php` files ở thư mục gốc).
*   **Bảo mật:**
    *   **CSRF Protection:** Chống giả mạo yêu cầu từ phía máy khách.
    *   **Xác thực đa tầng:** Hỗ trợ cả đăng nhập truyền thống, 2FA (TOTP) và OIDC/OAuth2 (cho phép dùng tài khoản Google, Authelia, v.v.).
    *   **Phân quyền:** Có hệ thống User và Admin riêng biệt.
*   **PWA (Progressive Web App):** Có `manifest.json` và `service-worker.js`, cho phép người dùng "cài đặt" Wallos lên điện thoại như một ứng dụng di động thực thụ và hoạt động mượt mà trong môi trường mạng yếu.
*   **Đa ngôn ngữ (i18n):** Hệ thống dịch thuật được quản lý qua các file PHP và JS riêng biệt cho từng ngôn ngữ trong `includes/i18n`, giúp việc đóng góp từ cộng đồng rất dễ dàng.

---

### 3. Tóm tắt luồng hoạt động (Workflow)

Luồng hoạt động của Wallos có thể chia làm 3 giai đoạn chính:

#### Giai đoạn 1: Khởi tạo (Initialization)
1.  **Startup:** Script `startup.sh` chạy khi container khởi động: kiểm tra quyền thư mục, khởi tạo file SQLite (`wallos.db`) từ file trống nếu chưa có.
2.  **Migration:** Tự động chạy các bản cập nhật database mới nhất.
3.  **Setup:** Người dùng truy cập lần đầu để tạo tài khoản Admin, thiết lập tiền tệ chính (Main Currency) và các danh mục chi tiêu.

#### Giai đoạn 2: Quản lý đăng ký (Core Loop)
1.  **Nhập liệu:** Người dùng thêm một dịch vụ (ví dụ: Netflix).
2.  **Xử lý logo:** Hệ thống có script `search.php` để tìm kiếm logo trên web nếu người dùng không tải lên.
3.  **Tính toán:** Dựa trên chu kỳ (hàng tháng, hàng năm) và tỷ giá hối đoái (nếu khác đơn vị tiền tệ chính), hệ thống tính toán chi phí quy đổi.

#### Giai đoạn 3: Tự động hóa và Báo cáo (Automation & Reporting)
1.  **Cronjobs:** Đây là "trái tim" của ứng dụng. Các tác vụ chạy ngầm định kỳ để:
    *   Cập nhật tỷ giá hối đoái mới nhất.
    *   Kiểm tra các đăng ký sắp đến hạn thanh toán.
    *   Gửi thông báo qua Telegram, Discord, Email...
    *   Tự động gia hạn (cập nhật ngày thanh toán tiếp theo) cho các dịch vụ.
2.  **Visualization:** Trang `stats.php` và `index.php` lấy dữ liệu đã xử lý để hiển thị biểu đồ chi tiêu, giúp người dùng nhận biết các khoản phí "ẩn" hoặc đề xuất cắt giảm qua AI.

**Kết luận:** Wallos là một dự án có tư duy thiết kế thực dụng, tập trung vào khả năng tùy biến cao và quyền riêng tư (self-hosted), với hệ thống tự động hóa mạnh mẽ thông qua Cronjobs và API.