Dựa trên cấu trúc thư mục và nội dung mã nguồn của **Open Web Analytics (OWA)**, dưới đây là phân tích chi tiết về hệ thống này:

### 1. Công nghệ cốt lõi (Core Technology Stack)

OWA là một hệ thống lâu đời (từ 2006) nhưng đã được hiện đại hóa qua các phiên bản.
*   **Ngôn ngữ lập trình:** **PHP** là ngôn ngữ chủ đạo. Hệ thống yêu cầu PHP hiện đại (hỗ trợ các tính năng như `password_hash`, `filter_var`).
*   **Quản lý phụ viện:** Sử dụng **Composer**. Các thư viện quan trọng bao gồm `GuzzleHttp` (xử lý request HTTP), `Monolog` (ghi log), `PHPMailer` (gửi mail), và `ua-parser` (phân tích trình duyệt/thiết bị).
*   **Cơ sở dữ liệu:** Chủ yếu tối ưu cho **MySQL** (thể hiện qua `owa_db_mysql.php`). OWA sử dụng kiến trúc Fact/Dimension (giống Data Warehouse) để lưu trữ dữ liệu phân tích.
*   **Frontend:** Sử dụng **JavaScript** thuần cho tracker client. Dashboard sử dụng kết hợp giữa template PHP (`.tpl`) và các thư viện hiện đại như **jQuery**, **Flot** (vẽ biểu đồ), và **Webpack** để đóng gói tài nguyên.
*   **Tích hợp:** Có sẵn module hỗ trợ **WordPress** (`wp_plugin.php`).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của OWA rất đồ sộ và tuân thủ các nguyên tắc thiết kế phần mềm cổ điển:

*   **Modular Architecture (Kiến trúc Module):** OWA được thiết kế để mở rộng. Mọi tính năng (từ cache, geolocation đến việc ghi hình phiên làm việc - domstream) đều nằm trong thư mục `modules/`. Mỗi module có thể đăng ký Controller, Entity, Event Handler và View riêng.
*   **Service-Oriented & Singleton:** Sử dụng `owa_coreAPI` làm trung tâm điều phối (Service Locator). Hầu hết các thành phần quan trọng (DB, Auth, Cache, EventDispatch) đều được truy cập qua mô hình **Singleton** để đảm bảo duy nhất một instance trong suốt vòng đời của request.
*   **Data Warehouse Design:** Cách đặt tên các entity như `action_fact.php`, `location_dim.php` cho thấy OWA tổ chức dữ liệu theo mô hình **Star Schema** (Fact và Dimension tables), giúp tối ưu hóa việc truy vấn báo cáo trên lượng dữ liệu lớn.
*   **Multi-role Execution:** Hệ thống có thể chạy dưới nhiều vai trò khác nhau tùy theo endpoint: `logger` (chỉ ghi dữ liệu), `admin_web` (hiển thị báo cáo), `installer` (cài đặt), và `cli` (chạy lệnh từ terminal).

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Factory Pattern:** Sử dụng cực kỳ phổ biến (`owa_coreAPI::entityFactory`, `owa_coreAPI::supportClassFactory`). Kỹ thuật này giúp hệ thống khởi tạo các đối tượng dựa trên tên chuỗi, cho phép nạp các class từ các module khác nhau một cách linh hoạt.
*   **Observer & Event-Driven:** OWA có một hệ thống sự kiện mạnh mẽ (`owa_observer.php`, `owa_event.php`). Khi một sự kiện xảy ra (ví dụ: `new_session`), các "Handlers" đăng ký với sự kiện đó sẽ được kích hoạt để xử lý logic tương ứng.
*   **Active Record / ORM Layer:** Lớp `owa_entity.php` và `owa_db.php` đóng vai trò như một ORM tự chế. Các thực thể (Entity) tự biết cách `save()`, `update()`, và `delete()` chính mình, đồng thời quản lý định nghĩa cột (schema) ngay trong code PHP.
*   **Request Abstraction:** `owa_requestContainer.php` chuẩn hóa toàn bộ dữ liệu đầu vào từ GET, POST, Cookie và CLI vào một đối tượng duy nhất, giúp bảo mật (sanitization) và truy cập dữ liệu đồng nhất.
*   **Security & Anti-Exploit:**
    *   Sử dụng **Nonce** để chống CSRF.
    *   Hệ thống `owa_sanitize` lọc dữ liệu đầu vào nghiêm ngặt.
    *   Kỹ thuật `ignore_user_abort(true)` và `flush()` trong `log.php` để phản hồi request ảnh 1x1 cực nhanh cho client rồi mới xử lý lưu DB ở backend.

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng thu thập dữ liệu (Tracking Flow):
1.  **Client-side:** Javascript tracker gửi request (thường là GET) đến `log.php` kèm theo các tham số như `event_type`, `site_id`, `url`...
2.  **Server-side (`log.php`):** 
    *   Hệ thống ngay lập tức trả về một ảnh GIF 1x1 pixel và ngắt kết nối với trình duyệt người dùng (`ob_flush()`, `flush()`) để người dùng không cảm thấy lag.
    *   Sau đó, PHP tiếp tục chạy ngầm: `owa_requestContainer` phân giải các tham số.
    *   `owa_coreAPI::logEvent` được gọi, kích hoạt chuỗi sự kiện.
    *   Các Event Processors sẽ phân tích User-Agent để xác định thiết bị, gọi MaxMind để xác định vị trí (Geolocation), sau đó lưu vào các bảng Fact/Dimension.

#### B. Luồng báo cáo (Reporting Flow):
1.  Người dùng truy cập `index.php`.
2.  `owa_auth` kiểm tra quyền qua Cookie hoặc API Key (có kiểm tra chữ ký `signature` cho REST API).
3.  `owa_reportController` tiếp nhận yêu cầu, xác định khoảng thời gian (`timePeriod`) và site cần xem.
4.  Hệ thống gọi các **Metric** classes. Các class này sẽ thực hiện các câu lệnh SQL phức tạp (`SUM`, `COUNT`, `DISTINCT`) trên các bảng Fact.
5.  Dữ liệu được trả về dưới dạng JSON (cho API) hoặc render qua `owa_template` để hiển thị biểu đồ trên dashboard.

#### C. Luồng CLI (Maintenance Flow):
1.  Thông qua `cli.php`, quản trị viên có thể chạy các tác vụ nặng như cập nhật database schema (`updatesApplyCli`), đổi mật khẩu, hoặc xử lý hàng đợi (queue) mà không bị giới hạn bởi timeout của web server.

### Tổng kết
So với Fathom Lite (tinh gọn, hiện đại), **Open Web Analytics** là một "con quái vật" về tính năng và độ phức tạp. Nó mang tư duy của các hệ thống Enterprise cũ nhưng cực kỳ mạnh mẽ, cho phép can thiệp sâu vào mọi ngõ ngách của dữ liệu và quy trình xử lý. Đây là lựa chọn cho những ai muốn một hệ thống tự thụ hoàn toàn, có khả năng tùy biến vô hạn và báo cáo chuyên sâu như Google Analytics.