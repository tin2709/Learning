Dựa trên nội dung mã nguồn của dự án **TasmoAdmin** (phiên bản v4.x), dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống.

---

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Backend:** PHP 8.2+ (tiệm cận PHP 8.5 trong các bản dev). Sử dụng kiến trúc hướng đối tượng (OOP) kết hợp với PSR-4 autoloading.
*   **Framework Components:** Không dùng một Framework lớn (như Laravel/Symfony full-stack) mà sử dụng các thành phần rời rạc từ **Symfony** để giữ hệ thống nhẹ:
    *   `symfony/routing`: Quản lý đường dẫn.
    *   `symfony/http-foundation`: Xử lý Request/Response.
    *   `selective/container`: Dependency Injection (DI) Container để quản lý dịch vụ.
*   **Frontend:**
    *   **Giao diện:** Bootstrap 4/5 (đang trong quá trình nâng cấp), FontAwesome.
    *   **Build Tool:** `esbuild` để biên dịch SCSS và JavaScript, `node-minify` để tối ưu hóa tài nguyên.
    *   **Client Logic:** jQuery kết hợp với các thư viện bổ trợ như `tablesaw` (cho bảng phản hồi) và `js-cookie`.
*   **Lưu trữ (Storage):** **Flat-file database**. Sử dụng `devices.csv` để lưu danh sách thiết bị và `MyConfig.json` cho cấu hình hệ thống. Điều này giúp ứng dụng chạy được trên các thiết bị tài nguyên thấp (Raspberry Pi, ESP8266) mà không cần SQL Server.
*   **Giao tiếp thiết bị:** Guzzle HTTP Client để gửi lệnh tới firmware Tasmota qua giao thức HTTP/REST API.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Hệ thống đi theo mô hình **Hybrid Controller-Page Architecture**:

*   **Front Controller Pattern:** Tất cả yêu cầu đều đi qua `index.php`. Tại đây, hệ thống khởi tạo `bootstrap.php`, nạp Container và thực hiện Routing.
*   **Dependency Injection (DI):** Toàn bộ các logic quan trọng (Config, DeviceRepository, Sonoff API) được đăng ký trong `includes/container.php`. Điều này giúp mã nguồn dễ kiểm thử (Unit Test) và dễ bảo trì.
*   **Dịch vụ hóa (Service Layer):**
    *   `DeviceRepository`: Chịu trách nhiệm CRUD dữ liệu từ file CSV.
    *   `Sonoff`: Lớp trừu tượng (Abstraction) để giao tiếp với thiết bị Tasmota (gửi lệnh, kiểm tra trạng thái).
    *   `UpdateChecker/FirmwareDownloader`: Xử lý logic nghiệp vụ về cập nhật Firmware.
*   **Cấu trúc thư mục rõ ràng:**
    *   `src/`: Chứa logic nghiệp vụ thuần túy (Classes).
    *   `pages/`: Chứa các "view-controllers" xử lý logic hiển thị cụ thể cho từng trang.
    *   `data/`: Phân tách hoàn toàn dữ liệu người dùng khỏi mã nguồn, tạo điều kiện cho Docker Volume dễ dàng.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Xử lý đồng thời (Concurrency):** Trong cấu hình có `REQUEST_CONCURRENCY`. Hệ thống sử dụng Guzzle để gửi các yêu cầu HTTP không đồng bộ (hoặc song song) khi cần kiểm tra trạng thái hoặc cập nhật firmware cho hàng loạt thiết bị cùng lúc.
*   **Tối ưu hóa tài nguyên:**
    *   Sử dụng cơ chế nén GZIP cho các gói dữ liệu.
    *   Minify JS/CSS tự động qua Makefile/npm scripts.
*   **Local OTA Server:** TasmoAdmin tự đóng vai trò là một máy chủ OTA địa phương. Nó tải firmware từ GitHub về, sau đó cung cấp URL nội bộ cho các thiết bị Tasmota tải về, giúp giảm băng thông quốc tế và tăng tính ổn định.
*   **Đa ngôn ngữ (i18n):** Sử dụng `php-i18n` nạp các file `.ini`. Điểm đặc biệt là dự án có `JsonLanguageHelper` để đổ dữ liệu ngôn ngữ ra JSON cho phía Client (JavaScript) sử dụng chung, đảm bảo đồng nhất dịch thuật.
*   **Hệ thống tự cập nhật (Self-Update):** Cho phép ứng dụng tự tải mã nguồn mới từ GitHub và ghi đè chính nó (không áp dụng cho Docker vì Docker quản lý qua Image).

### 4. Luồng hoạt động (Operational Flow Summary)

1.  **Khởi động (Bootstrapping):**
    *   `index.php` gọi `bootstrap.php`.
    *   Kiểm tra môi trường (PHP version, cURL, Zip extension).
    *   Khởi tạo `i18n` và nạp cấu hình từ `MyConfig.json`.
    *   Xác thực người dùng (Session/Login).

2.  **Định tuyến (Routing):**
    *   `Symfony\Component\Routing` khớp URL người dùng yêu cầu với danh sách trong `routes.php`.
    *   Nếu yêu cầu trang UI, nó dùng `render_template` để nạp trang tương ứng từ thư mục `pages/` lồng vào khung `header/footer`.

3.  **Giao tiếp thiết bị (Device Interaction):**
    *   Khi người dùng nhấn "Toggle" một công tắc trên giao diện:
    *   JavaScript gửi yêu cầu AJAX tới `actions.php`.
    *   Backend sử dụng dịch vụ `Sonoff` để gửi lệnh HTTP (`cm?cmnd=Power...`) tới IP của thiết bị.
    *   Kết quả trả về được Parse từ JSON và phản hồi lại giao diện.

4.  **Cập nhật Firmware (Firmware Update Flow):**
    *   Người dùng chọn thiết bị -> Chọn bản Firmware.
    *   Hệ thống kiểm tra phiên bản hiện tại -> Gửi lệnh cho thiết bị chuyển sang chế độ nạp Firmware.
    *   Thiết bị tải file từ thư mục `data/firmwares/` của TasmoAdmin.

### Tóm tắt đặc điểm nổi bật
TasmoAdmin là một dự án **"Lightweight nhưng Modern"**. Nó sử dụng các công nghệ hiện đại của PHP (DI Container, Composer, Symfony Components) nhưng lại chọn cách lưu trữ dữ liệu tối giản (Flat-file) và giao tiếp trực tiếp (HTTP) để phù hợp hoàn hảo với hệ sinh thái Smart Home nhỏ lẻ và các thiết bị nhúng.