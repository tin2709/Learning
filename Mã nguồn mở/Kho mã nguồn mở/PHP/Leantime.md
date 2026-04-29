Dựa trên các tệp tin và thông tin kiến trúc bạn cung cấp, dưới đây là phân tích chi tiết về dự án **Leantime** — một hệ thống quản lý dự án nguồn mở hiện đại được tối ưu cho người dùng không chuyên và người có đặc điểm thần kinh đa dạng (neurodiverse).

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Backend:**
    *   **PHP 8.2+:** Sử dụng các tính năng mới nhất của PHP để đảm bảo hiệu năng và bảo mật.
    *   **Laravel 11:** Đóng vai trò là nền tảng (framework) cơ sở, nhưng được tùy biến sâu (custom) để phù hợp với kiến trúc riêng của Leantime.
    *   **MySQL 8.0+ / MariaDB 10.6+:** Cơ sở dữ liệu quan hệ chính.
    *   **JSON-RPC 2.0:** Giao thức chính cho API, cho phép gọi các phương thức dịch vụ (service methods) một cách nhất quán.
*   **Frontend:**
    *   **HTMX:** Công nghệ quan trọng đang được dùng để thay thế jQuery, giúp cập nhật giao diện mà không cần tải lại trang nhưng vẫn giữ logic ở phía server.
    *   **Blade & Legacy TPL:** Hệ thống template kép, đang chuyển đổi dần từ PHP thuần (.tpl.php) sang Laravel Blade.
    *   **Tailwind CSS & Bootstrap:** Đang chuyển dịch từ Bootstrap 2.x (legacy) sang Tailwind CSS (với tiền tố `tw-`) để hiện đại hóa giao diện.
    *   **JavaScript:** Sử dụng kiến trúc namespace toàn cục (`leantime`) với mô hình IIFE.
*   **Hạ tầng:**
    *   **Docker & Docker Compose:** Hỗ trợ môi trường phát triển và triển khai nhanh chóng.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Leantime áp dụng kiến trúc **Domain-Driven Design (DDD)** kết hợp với mô hình **Modular Monolith**:

*   **Phân tách Core và Domain:**
    *   `app/Core/`: Chứa các thành phần cốt lõi của framework (xử lý HTTP, DB abstraction, Event system, Translation).
    *   `app/Domain/`: Chia thành 56+ module (Tickets, Projects, Users...). Mỗi module là một "thế giới" riêng với đầy đủ các lớp: Controller, Service, Repository, Model, Template.
*   **Kiến trúc Dịch vụ (Service Layer):** Logic nghiệp vụ không nằm ở Controller mà nằm ở các Service. Các Service này đồng thời là bề mặt API cho JSON-RPC.
*   **Tư duy "Less is More":** Tập trung vào việc giảm tải nhận thức (cognitive load). Mọi tính năng được thiết kế để "vô hình", chỉ xuất hiện khi cần thiết, hỗ trợ tốt cho người dùng bị ADHD hoặc Dyslexia.
*   **Khả năng mở rộng (Extensibility):** Hệ thống Plugin mạnh mẽ cho phép mở rộng tính năng mà không can thiệp vào mã nguồn lõi.

### 3. Các kỹ thuật chính (Key Techniques)

*   **HTMX Migration:** Đây là kỹ thuật "hiện đại hóa" ứng dụng PHP cũ. Thay vì xây dựng một SPA (Single Page Application) phức tạp, Leantime dùng HTMX để tải các đoạn HTML (partials) từ server, giúp trải nghiệm mượt mà nhưng vẫn giữ codebase đơn giản.
*   **Event & Filter System (Hook-based):** Sử dụng hệ thống Hook tương tự như WordPress (Events để kích hoạt hành động, Filters để thay đổi dữ liệu). Điều này cho phép các Plugin can thiệp vào bất kỳ đâu trong luồng xử lý.
*   **Thiết kế cho Neurodiversity:** Sử dụng các font chữ đặc biệt (Atkinson Hyperlegible), tối ưu độ tương phản và tập trung vào phản hồi thị giác rõ ràng.
*   **JSON-RPC API Mapping:** Tự động ánh xạ các phương thức public trong Service lớp Domain thành các endpoint API thông qua Reflection PHP, giúp giảm thiểu việc viết code boilerplate cho API.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Một yêu cầu (request) điển hình trong Leantime sẽ đi qua các bước sau:

1.  **Entry Point:** Yêu cầu đi vào `public/index.php`.
2.  **Bootstrapping:** Hệ thống tải cấu hình từ `.env`, khởi tạo Container của Laravel và chạy các Service Providers của Leantime Core.
3.  **Routing:**
    *   Hệ thống kiểm tra các route của Laravel trước.
    *   Nếu không khớp, nó chuyển sang **Frontcontroller** (Legacy). Frontcontroller sẽ phân tích URL theo mẫu `/module/action/id` để tìm Controller tương ứng trong `app/Domain/`.
4.  **Middleware:** Chạy qua một chuỗi các lớp bảo vệ (xác thực session, kiểm tra cài đặt, ngôn ngữ, quyền truy cập dự án).
5.  **Execution:**
    *   **Controller** tiếp nhận yêu cầu, gọi các **Service** liên quan.
    *   **Service** thực hiện logic nghiệp vụ, gọi **Repository** để truy vấn DB (sử dụng Laravel Query Builder hoặc SQL thuần).
6.  **Response:**
    *   Nếu là yêu cầu thông thường: Trả về giao diện đầy đủ (Layout + Template).
    *   Nếu là yêu cầu **HTMX**: Chỉ trả về một đoạn HTML nhỏ (Partial) để cập nhật một phần trang web.
7.  **Events:** Sau khi xử lý xong, các sự kiện (như `ticket_updated`) được phát đi để các module khác hoặc Plugin có thể thực hiện các tác vụ phụ (như gửi email, thông báo Slack).

### Kết luận
Leantime là một sự kết hợp thú vị giữa **truyền thống (PHP Monolith)** và **hiện đại (HTMX, DDD)**. Nó cho thấy cách một hệ thống lớn có thể duy trì sự ổn định lâu dài trong khi vẫn liên tục cải tiến về mặt công nghệ và trải nghiệm người dùng đặc thù.