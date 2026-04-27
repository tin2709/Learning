Dựa trên mã nguồn của dự án **Shlink** (phiên bản v5.x), một nền tảng rút gọn URL tự điều hành (self-hosted) viết bằng PHP, dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống:

### 1. Công nghệ cốt lõi (Core Technology)

Shlink không đi theo các framework phổ biến như Laravel hay Symfony mà chọn một hướng tiếp cận tinh gọn và hiệu suất cao hơn:

*   **Framework Base:** Sử dụng **Mezzio** (trước đây là Zend Expressive). Đây là một framework dựa trên chuẩn **PSR-15 (Middleware)**, cho phép xây dựng ứng dụng dưới dạng các lớp xử lý (pipeline) lồng nhau, rất phù hợp cho các microservices hoặc ứng dụng API.
*   **Ngôn ngữ:** **PHP 8.4+**. Tận dụng tối đa các tính năng hiện đại như *Constructor Property Promotion*, *Readonly properties*, và *Strict types*.
*   **Database & ORM:** Sử dụng **Doctrine ORM (v3.x)**. Shlink nổi bật với khả năng hỗ trợ đa cơ sở dữ liệu: MySQL, MariaDB, PostgreSQL, Microsoft SQL Server và SQLite.
*   **High Performance Runtime:** 
    *   **RoadRunner:** Sử dụng làm máy chủ ứng dụng chính (PHP Application Server) để duy trì ứng dụng trong bộ nhớ, giúp tốc độ phản hồi cực nhanh (thay thế cho PHP-FPM truyền thống).
    *   **FrankenPHP:** Bắt đầu hỗ trợ máy chủ ứng dụng hiện đại dựa trên Caddy.
*   **Real-time & Messaging:** Tích hợp **Mercure Hub** để đẩy thông báo thời gian thực, **RabbitMQ** và **Redis Pub/Sub** để xử lý các sự kiện bất đồng bộ.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Shlink phản ánh tư duy của một hệ thống có tính tùy biến và mở rộng cao:

*   **Modular Monolith (Khối thống nhất chia module):** Mã nguồn được chia thành 3 module chính trong thư mục `module/`:
    *   `Core`: Chứa logic nghiệp vụ cốt lõi, thực thể (Entities), và truy cập dữ liệu.
    *   `Rest`: Xử lý giao tiếp API.
    *   `CLI`: Xử lý các lệnh điều khiển qua terminal.
*   **Architectural Decision Records (ADR):** Dự án duy trì một thư mục `docs/adr/` cực kỳ bài bản. Đây là nơi lưu trữ lý do tại sao các quyết định kỹ thuật được đưa ra (ví dụ: tại sao chọn RoadRunner, tại sao hỗ trợ multi-segment slugs). Điều này thể hiện tư duy quản trị dự án chuyên nghiệp.
*   **Pipeline-Centric:** Luồng xử lý request được cấu hình như một đường ống (pipeline) trong `middleware-pipeline.global.php`. Mỗi request đi qua các "trạm" như xác thực, định danh IP, định vị địa lý trước khi đến logic cuối cùng.
*   **Configuration Aggregation:** Sử dụng `laminas-config-aggregator` để gộp cấu hình từ nhiều nguồn (global, module, env vars), giúp hệ thống cực kỳ linh hoạt trong việc cấu hình qua Docker hoặc file.

### 3. Kỹ thuật lập trình chính (Primary Programming Techniques)

*   **Specification Pattern:** Shlink sử dụng `happyr/doctrine-specification` để xây dựng các câu lệnh truy vấn phức tạp. Ví dụ: thay vì viết SQL dài dòng, hệ thống sử dụng các class Specification để lọc URL theo Tag, Domain hoặc API Key một cách sạch sẽ.
*   **Dependency Injection (DI):** Sử dụng `laminas-servicemanager` để quản lý các service. Hầu hết các thành phần được khởi tạo thông qua các Factory class để đảm bảo tính lỏng lẻo (loose coupling).
*   **Dynamic Redirect Rules:** Một kỹ thuật nâng cao được giới thiệu gần đây là hệ thống quy tắc điều hướng động. Cho phép điều hướng người dùng dựa trên: Thiết bị (Mobile/Desktop), Ngôn ngữ trình duyệt, Tham số Query, hoặc Địa lý (Quốc gia/Thành phố).
*   **Hashing API Keys:** Kể từ v4.3, Shlink chuyển sang lưu trữ API Keys dưới dạng mã hóa SHA256 thay vì văn bản thuần, tăng cường bảo mật cấp độ doanh nghiệp.

### 4. Luồng hoạt động hệ thống (System Workflow)

Hệ thống hoạt động theo các luồng chính rất tối ưu:

1.  **Luồng Điều hướng (Redirection - Quan trọng nhất):**
    *   Khách truy cập URL ngắn -> RoadRunner nhận request -> Chuyển qua Pipeline Middleware.
    *   **Middleware định danh:** Lấy IP, User-Agent.
    *   **Middleware xử lý logic:** Tìm kiếm mã ngắn trong Cache (Redis). Nếu không có, truy vấn Database.
    *   **Rule Engine:** Kiểm tra các quy tắc động (Ví dụ: Nếu là iPhone thì chuyển hướng đến App Store).
    *   **Async Tracking:** Ghi lại lượt truy cập (Visit) một cách bất đồng bộ qua RoadRunner Jobs để không làm chậm quá trình điều hướng của người dùng.
    *   **Kết quả:** Trả về mã Redirect (301, 302, 307, 308) cho trình duyệt.

2.  **Luồng Quản trị (Rest API):**
    *   Client gửi request (kèm API Key trong header) -> `AuthenticationMiddleware` kiểm tra quyền (Role-based access).
    *   Xử lý logic (Tạo URL, thống kê lượt click) -> Trả về kết quả JSON theo chuẩn **Problem Details (RFC 7807)** nếu có lỗi.

3.  **Luồng Đồng bộ (Real-time Updates):**
    *   Khi có lượt click mới -> Một sự kiện được bắn ra -> Các listener gửi dữ liệu đến Mercure (để hiển thị trên Dashboard ngay lập tức) hoặc đẩy vào Matomo để phân tích sâu hơn.

**Tổng kết:** Shlink là một dự án PHP mẫu mực về việc áp dụng các tiêu chuẩn hiện đại, ưu tiên hiệu suất cao thông qua các runtime không chặn (non-blocking) và cấu trúc code cực kỳ chặt chẽ, dễ bảo trì.