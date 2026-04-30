Dựa trên cấu trúc mã nguồn và các tệp tin cấu hình của **Drupal** (phiên bản core hiện tại), dưới đây là phân tích chi tiết về công nghệ, kiến trúc và quy trình hoạt động của hệ quản trị nội dung (CMS) mã nguồn mở này:

### 1. Công nghệ cốt lõi (Core Technology)

Drupal là một hệ sinh thái PHP khổng lồ, chuyển mình mạnh mẽ từ các phiên bản cũ sang kiến trúc hiện đại dựa trên các thành phần tiêu chuẩn:

*   **Ngôn ngữ chính:** **PHP (89.6%)**. Yêu cầu phiên bản rất mới (PHP 8.3/8.5+) cho các bản cập nhật gần đây nhất.
*   **Nền tảng Framework:** Dựa trên các thành phần của **Symfony** (như DependencyInjection, EventDispatcher, Routing, Console, HttpFoundation). Drupal không phải là Symfony, nhưng nó "mượn" những phần tốt nhất của Symfony.
*   **Dependency Management:** Sử dụng **Composer** cực kỳ chặt chẽ. Thư mục `composer/` chứa các plugin tùy chỉnh (Scaffold, VendorHardening) để quản lý cấu trúc tệp tin của Drupal ngoài thư mục `vendor`.
*   **Template Engine:** **Twig (2.4%)** – giúp tách biệt hoàn toàn logic xử lý và giao diện hiển thị, đảm bảo an toàn (XSS protection).
*   **Database Agnostic:** Hỗ trợ đa cơ sở dữ liệu thông qua hệ thống Driver (MySQL/MariaDB, PostgreSQL, SQLite).
*   **Testing:** Hệ thống test đồ sộ sử dụng PHPUnit, Nightwatch.js (cho functional JS testing) và OpenTelemetry để đo lường hiệu suất.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Drupal được xây dựng để trở thành một "Platform" hơn là một "Product":

*   **Modular Architecture (Kiến trúc Module):** Nhân core chỉ chứa các tính năng tối thiểu. Mọi tính năng khác được thêm vào qua Modules.
*   **Entity API:** Drupal coi mọi thứ là "Entity" (Node - nội dung, User, Comment, Taxonomy Term). Cách tiếp cận này giúp xử lý dữ liệu nhất quán và linh hoạt.
*   **Service-Oriented Architecture (SOA):** Sử dụng **Dependency Injection Container**. Các chức năng chính (Database, Mail, Path, Cache) đều được định nghĩa là các "Services", cho phép các module khác ghi đè hoặc mở rộng dễ dàng.
*   **Hooks & Plugins:**
    *   *Hooks:* Cơ chế hướng sự kiện truyền thống của Drupal để các module can thiệp vào luồng xử lý.
    *   *Plugins:* Hệ thống hướng đối tượng mới hơn để quản lý các thành phần có thể hoán đổi (ví dụ: các loại Block, các kiểu định dạng văn bản).
*   **Configuration Management (CMI):** Toàn bộ cấu trúc trang web (nội dung kiểu gì, view hiển thị ra sao) được quản lý bằng các tệp **YAML**, cho phép đồng bộ cấu hình giữa các môi trường (Dev -> Staging -> Prod) mà không cần chuyển đổi database.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Composer Scaffolding:** Một kỹ thuật đặc thù của Drupal để quản lý các tệp tin "cứng" (như `.htaccess`, `index.php`) vốn thường bị Composer ghi đè hoặc bỏ qua.
*   **Advanced Caching:**
    *   *Render Cache:* Lưu các thành phần giao diện đã render.
    *   *Dynamic Page Cache:* Lưu trữ các trang cho người dùng đã đăng nhập nhưng vẫn đảm bảo tính cá nhân hóa.
    *   *BigPipe:* Kỹ thuật gửi các phần tĩnh của trang web trước, sau đó mới gửi các phần động (do Facebook khởi xướng), giúp giảm thời gian cảm nhận tải trang (First Meaningful Paint).
*   **Asset Management:** Tự động gộp (aggregate), nén (minify) CSS/JS và hỗ trợ nén **Brotli/Gzip** ngay từ tệp `.htaccess`.
*   **Security Hardening:** Tệp `VendorHardening` plugin tự động bảo vệ thư mục `vendor`, ngăn chặn việc thực thi code trái phép từ các thư viện bên thứ ba.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

**Luồng xử lý một Request (HTTP Request Lifecycle):**
1.  **Tiếp nhận:** Yêu cầu đi qua `index.php`.
2.  **Khởi tạo:** `DrupalKernel` được khởi động, nạp các Services từ Container và xác định Module nào đang hoạt động.
3.  **Routing:** Symfony Router đối chiếu URL với hệ thống đường dẫn (đã được cache).
4.  **Middleware:** Yêu cầu đi qua các tầng Middleware (xác thực, kiểm tra bảo trì, cache).
5.  **Controller & Logic:** Controller tương ứng xử lý logic, truy vấn Entity qua Entity API.
6.  **Rendering:** Dữ liệu được chuyển qua hệ thống Theme. Twig engine sẽ render dữ liệu thành HTML dựa trên các template và gợi ý (suggestions).
7.  **Response:** Kết quả được gửi lại trình duyệt, đồng thời lưu vào các tầng Cache nếu cần.

**Luồng phát triển (Development Workflow):**
*   Lập trình viên viết Code/Module -> Xuất cấu hình ra tệp YAML (`drush cex`) -> Commit lên Git -> Kéo về server và nhập cấu hình (`drush cim`).

**Kết luận:** Drupal hiện nay là một CMS cấp độ doanh nghiệp (Enterprise-grade) cực kỳ mạnh mẽ. Nó kết hợp giữa sự linh hoạt tuyệt đối của hệ thống Hook/Entity và tiêu chuẩn lập trình hiện đại của Symfony, biến nó thành lựa chọn hàng đầu cho các website chính phủ, đại học và tập đoàn lớn.