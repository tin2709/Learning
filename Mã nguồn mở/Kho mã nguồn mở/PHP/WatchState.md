Dựa trên cấu trúc thư mục và nội dung các file bạn đã cung cấp cho dự án **WatchState**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Technology Stack)

Hệ thống được xây dựng trên một ngăn xếp công nghệ hiện đại, tối ưu cho việc tự lưu trữ (self-hosting):

*   **Backend (PHP 8.4+):** Sử dụng phiên bản PHP mới nhất, tận dụng các tính năng hiện đại như Attributes, Typed Properties, và Readonly classes.
    *   **Framework Components:** Không dùng một Framework cồng kềnh (như Laravel/Symfony full-stack) mà kết hợp các thành phần rời rạc từ Symfony (Console, Cache, HttpClient, EventDispatcher), League (Container, Route) để tối ưu hiệu năng.
    *   **Server:** Sử dụng **FrankenPHP** (viết bằng Go, tích hợp Caddy), hỗ trợ PHP-Server hiệu năng cao và hiện đại hơn thay cho Nginx/PHP-FPM truyền thống.
*   **Frontend (Nuxt 4 & Vue 3):**
    *   **SPA Mode:** Frontend được xây dựng dưới dạng Single Page Application (SSR disabled), biên dịch thành các file tĩnh để nhúng trực tiếp vào backend PHP.
    *   **UI:** Sử dụng **Nuxt UI** (dựa trên Tailwind CSS) mang lại giao diện hiện đại, hỗ trợ Dark Mode và Responsive tốt.
*   **Cơ sở dữ liệu & Lưu trữ:**
    *   **SQLite:** Lựa chọn hoàn hảo cho dự án tự lưu trữ, không cần cài đặt server DB phức tạp, dữ liệu nằm gọn trong một file `.db`.
    *   **Redis:** Dùng làm Cache layer để tăng tốc các yêu cầu API và quản lý hàng đợi/trạng thái.
*   **Containerization:** Docker (Rootless) đảm bảo an toàn và dễ dàng triển khai trên các hệ thống như Unraid, Synology hoặc Linux Server.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của WatchState được thiết kế theo hướng **Modular** và **Adapter Pattern**:

*   **Backend Abstraction (Lớp trừu tượng Backend):** Hệ thống có một interface chung (`ClientInterface.php`) cho các Media Server. Mỗi loại backend (Plex, Emby, Jellyfin) có một "Adapter" riêng trong thư mục `src/Backends/`. Điều này giúp việc thêm một loại backend mới (như Kodi hay Stremio) trở nên dễ dàng mà không ảnh hưởng đến logic cốt lõi.
*   **Identity-based Multi-tenancy:** Hỗ trợ đa người dùng thông qua khái niệm `identities`. Mỗi identity có cấu hình và cơ sở dữ liệu riêng, cho phép đồng bộ hóa trạng thái xem cho nhiều tài khoản khác nhau trên cùng một instance.
*   **State Mapping (Ánh xạ trạng thái):** Sử dụng các `Mappers` (như `DirectMapper`, `MemoryMapper`) để chuyển đổi dữ liệu từ các định dạng API khác nhau của Plex/Jellyfin về một cấu trúc dữ liệu chuẩn của WatchState (`StateEntity`).
*   **Event-Driven (Kiến trúc hướng sự kiện):** Sử dụng `EventDispatcher` để xử lý các tác vụ như cập nhật tiến trình (progress), đẩy dữ liệu (push), giúp tách biệt logic xử lý chính và các tác vụ phụ.

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **Attribute-based Routing:** Sử dụng PHP Attributes (ví dụ: `#[Get('/v1/api/history')]`) để định nghĩa route ngay tại class xử lý (Action/Controller). Đây là cách làm của Symfony/FastAPI, giúp code cực kỳ tường minh.
*   **Dependency Injection (DI):** Sử dụng `league/container` để quản lý sự phụ thuộc. Các dịch vụ như Database, Logger, HttpClient được khởi tạo một lần và "tiêm" vào nơi cần thiết, giúp dễ dàng viết Unit Test.
*   **Asynchronous-like Processing:** Mặc dù PHP là đơn luồng, WatchState sử dụng `Symfony/HttpClient` để thực hiện các yêu cầu API song song (parallel) tới các media server để tăng tốc độ đồng bộ.
*   **Schema Migrations:** Tự quản lý phiên bản DB thông qua các file SQL trong thư mục `migrations/`, cho phép nâng cấp cơ sở dữ liệu tự động khi người dùng cập nhật phiên bản phần mềm.
*   **Strict Typing:** Toàn bộ code sử dụng `declare(strict_types=1)`, đảm bảo tính an toàn về kiểu dữ liệu, giảm thiểu bug logic.

### 4. Luồng hoạt động của hệ thống (System Workflow)

Hệ thống vận hành thông qua 3 luồng chính:

#### A. Luồng Khởi tạo (Initialization)
1. Docker khởi chạy script `init-container.sh`.
2. Kiểm tra quyền ghi vào thư mục `/config`.
3. Chạy các tiến trình phụ: Redis (Cache server) và `runner.sh` (Task Scheduler).
4. Thực hiện Migration để đảm bảo SQLite có cấu trúc mới nhất.
5. FrankenPHP bắt đầu lắng nghe tại cổng 8080.

#### B. Luồng Đồng bộ hóa (Synchronization)
Có 3 phương thức đồng bộ:
*   **Scheduled Tasks (Cron):** Chạy định kỳ (ví dụ: mỗi giờ). Tác vụ `Import` sẽ quét các backend để lấy trạng thái xem mới nhất, `Export` sẽ đẩy trạng thái từ local ra các backend khác.
*   **Webhooks:** Khi người dùng nhấn "Play" hoặc "Finish" trên Plex/Jellyfin, server đó gửi một tín hiệu (POST request) tới WatchState. WatchState xử lý ngay lập tức để cập nhật database local và các backend liên quan.
*   **On-demand:** Người dùng kích hoạt thủ công qua giao diện WebUI.

#### C. Luồng Xử lý Dữ liệu (Data Processing)
1. **Fetch:** Lấy dữ liệu từ Backend API (định dạng JSON).
2. **Parse/Map:** Dùng `Gid` (Global ID) như IMDB ID, TMDB ID để khớp nối phim/tập phim giữa các server khác nhau (vì ID nội bộ của Plex khác Jellyfin).
3. **Compare:** So sánh `last_watched_at` (thời gian xem cuối). Dữ liệu nào có timestamp mới hơn sẽ được ưu tiên.
4. **Conflict Resolution:** Nếu có xung đột (ví dụ: một bên báo đã xem, một bên báo chưa), hệ thống sẽ đánh dấu `tainted` để người dùng xử lý hoặc tự động ghi đè dựa trên cấu hình.

### Tổng kết
**WatchState** là một dự án có kỹ thuật lập trình rất chuyên nghiệp. Nó không chỉ đơn thuần là một script đồng bộ mà là một hệ thống quản lý trạng thái (State Management System) hoàn chỉnh, có khả năng mở rộng tốt và chú trọng vào hiệu suất thông qua việc tối ưu hóa giao tiếp HTTP và tận dụng tối đa các tính năng mới nhất của ngôn ngữ PHP.