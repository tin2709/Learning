Dựa trên cấu trúc thư mục và mã nguồn của **Nextcloud Server**, một nền tảng lưu trữ và cộng tác mã nguồn mở hàng đầu, dưới đây là phân tích chi tiết:

---

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ lập trình:**
    *   **Backend:** PHP (chiếm ~57%). Yêu cầu PHP 8.2+ (xem tệp `composer.json`).
    *   **Frontend:** JavaScript/TypeScript (chiếm ~37%) và Vue.js. Đang chuyển dịch mạnh mẽ sang Vue 3.
*   **Giao thức Real-time & Truyền tải:**
    *   **SabreDAV:** Đây là thành phần quan trọng nhất, xử lý toàn bộ các giao thức WebDAV (quản lý file), CalDAV (lịch), và CardDAV (danh bạ).
    *   **OCS (Open Collaboration Services):** Một tiêu chuẩn API để các ứng dụng cộng tác giao tiếp với nhau.
    *   **OCM (Open Cloud Mesh):** Cho phép kết nối (Federation) giữa các máy chủ Nextcloud khác nhau trên toàn cầu.
*   **Cơ sở dữ liệu & Lưu trữ:**
    *   Hỗ trợ đa dạng: MySQL/MariaDB, PostgreSQL, SQLite và Oracle.
    *   Hỗ trợ lưu trữ đối tượng (Object Storage): Amazon S3, Swift, Azure.
*   **Công cụ build & Chất lượng code:**
    *   **Composer:** Quản lý thư viện PHP.
    *   **NPM/Vite:** Build frontend.
    *   **Psalm & PHPUnit:** Phân tích tĩnh và kiểm thử tự động cho PHP.
    *   **Cypress:** Kiểm thử E2E (end-to-end) cho giao diện người dùng.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Kiến trúc App-centric (Hướng ứng dụng):**
    *   Nextcloud Core (`core/`) chỉ đóng vai trò là khung (framework). Hầu hết các tính năng (Files, Sharing, Activity, Settings...) đều được thiết kế như những ứng dụng độc lập nằm trong thư mục `apps/`.
    *   Mỗi ứng dụng có vòng đời riêng, API riêng và có thể được bật/tắt tùy ý.
*   **Phân lớp API (OCP vs OC):**
    *   **OCP (OC Public - `lib/public`):** Là lớp API công khai, ổn định dành cho các nhà phát triển App. Nextcloud cam kết bảo trì lớp này để đảm bảo tính tương thích.
    *   **OC (OC Private - `lib/private`):** Các logic nội bộ của hệ thống, có thể thay đổi mà không báo trước.
*   **Trừu tượng hóa hệ thống tệp (Filesystem Abstraction):**
    *   Nextcloud không lưu file trực tiếp một cách đơn giản. Nó sử dụng một lớp Virtual Filesystem (thông qua `lib/private/Files`) để quản lý quyền, mã hóa (Encryption) và versioning trước khi ghi vào bộ lưu trữ vật lý.
*   **Federation (Liên bang hóa):**
    *   Tư duy thiết kế không tập trung. Người dùng máy chủ A có thể chia sẻ file trực tiếp cho người dùng máy chủ B như thể họ đang ở cùng một hệ thống.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Mã hóa đầu cuối & Phân quyền:** Sử dụng kỹ thuật mã hóa phía máy chủ (Server-side encryption) và hỗ trợ mã hóa đầu cuối (E2EE) cho các thư mục nhạy cảm. Hệ thống phân quyền (`apps/workflowengine`) cho phép tự động hóa quy trình xử lý file dựa trên điều kiện.
*   **Background Jobs (Cron):** Quản lý qua `cron.php`. Nextcloud đẩy các tác vụ nặng (quét file, gửi thông báo, dọn dẹp phiên bản cũ) vào hàng chờ xử lý dưới nền để không làm chậm trải nghiệm người dùng.
*   **Brute-force Protection:** Tích hợp sẵn cơ chế chống tấn công dò mật khẩu ngay tại tầng ứng dụng (`apps/settings/lib/SetupChecks/BruteForceThrottler.php`).
*   **Versioning & Trashbin:** Kỹ thuật CoW (Copy on Write) hoặc lưu trữ delta để quản lý các phiên bản cũ của tệp tin mà không tốn quá nhiều dung lượng.
*   **Theming & Customization:** Khả năng thay đổi toàn bộ thương hiệu (logo, màu sắc) thông qua `apps/theming` mà không cần sửa code lõi.

---

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Tiếp nhận yêu cầu (Entry Points):**
    *   Nếu là giao diện web: Chạy qua `index.php`.
    *   Nếu là đồng gửi file/API (WebDAV): Chạy qua `remote.php`.
    *   Nếu là liên kết chia sẻ công khai: Chạy qua `public.php`.
2.  **Khởi tạo (Bootstrapping):**
    *   `lib/base.php` được gọi để nạp trình tự tải tự động (autoloader), kiểm tra cấu hình (`config.php`), và khởi tạo Container DI (Dependency Injection).
3.  **Xác thực & Kiểm tra bảo mật:**
    *   Hệ thống kiểm tra Session, Token hoặc App Password. Sau đó áp dụng các chính sách CSP (Content Security Policy) để chặn XSS.
4.  **Điều hướng (Routing):**
    *   Router tìm khớp URL với Controller trong các ứng dụng (`apps/`).
5.  **Xử lý nghiệp vụ:**
    *   Controller gọi các Service. Nếu liên quan đến file, yêu cầu sẽ đi qua lớp `Storage` -> `Cache` (để lưu metadata vào DB) -> `Encryption` (nếu có) -> `Filesystem`.
6.  **Phản hồi (Response):**
    *   Kết quả được trả về dưới dạng JSON (cho API) hoặc HTML (được render từ các template PHP/Vue).

**Kết luận:** Nextcloud là một hệ thống PHP cực kỳ đồ sộ và mẫu mực trong việc áp dụng thiết kế hướng đối tượng (OOP) và tính module hóa. Nó biến một ngôn ngữ scripting như PHP thành một nền tảng Enterprise Cloud mạnh mẽ và bảo mật.