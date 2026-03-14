Dựa trên mã nguồn và cấu trúc thư mục của dự án **Kimai**, đây là bản phân tích chi tiết về công nghệ, kiến trúc và cách thức hoạt động của hệ thống này:

### 1. Công nghệ cốt lõi (Core Technologies)

Kimai là một ứng dụng Web truyền thống nhưng được xây dựng với các tiêu chuẩn hiện đại (Modern Monolith):

*   **Backend (Linh hồn của hệ thống):**
    *   **PHP 8.1+:** Tận dụng các tính năng mới như Attributes, Typed Properties, và Enums.
    *   **Symfony Framework (v6.4+):** Sử dụng các Component mạnh mẽ như Security (xác thực), Form (xử lý dữ liệu), Validator, và Messenger.
    *   **Doctrine ORM:** Công cụ ánh xạ thực thể (Entities) vào DB (MySQL/MariaDB), sử dụng Repository Pattern để quản lý truy vấn.
    *   **Twig:** Công cụ template engine để render giao diện phía server.
*   **Frontend (Giao diện người dùng):**
    *   **Tabler UI:** Một Dashboard template dựa trên **Bootstrap 5**, tạo giao diện chuyên nghiệp, sạch sẽ.
    *   **Webpack Encore:** Để quản lý và biên dịch Assets (JS, SCSS).
    *   **JavaScript (Vanilla & Custom Plugins):** Thay vì dùng React/Vue, Kimai sử dụng một hệ thống "Plugin JS" tự xây dựng (xem `assets/js/KimaiLoader.js`) để điều khiển các hành vi động như Modal, Autocomplete, và Datatable.
*   **Hạ tầng & Công cụ:**
    *   **Docker:** Cung cấp môi trường chạy chuẩn hóa (FPM, Apache).
    *   **Composer:** Quản lý thư viện PHP.
    *   **Yarn/NPM:** Quản lý thư viện CSS/JS.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Kimai tập trung vào tính **mở rộng** và **vững chắc**:

*   **Kiến trúc Plugin (Extensibility):** Thư mục `var/plugins/` và các lớp quản lý Plugin (`src/Plugin/`) cho phép cộng đồng viết thêm tính năng mà không can thiệp vào mã nguồn lõi.
*   **Phân tách nghiệp vụ (Domain Separation):** Dữ liệu được tổ chức theo các Domain chính: `Customer` (Khách hàng) -> `Project` (Dự án) -> `Activity` (Hoạt động) -> `Timesheet` (Bảng chấm công).
*   **Hệ thống Phân quyền tinh vi (Granular Permissions):** Sử dụng **Symfony Voters** (`src/Voter/`) để kiểm tra quyền truy cập đến từng bản ghi cụ thể (ví dụ: Team lead chỉ thấy Timesheet của nhân viên trong đội).
*   **Đa quốc gia (i18n):** Hỗ trợ hơn 30 ngôn ngữ thông qua hệ thống dịch thuật mạnh mẽ (`translations/`).

### 3. Các kỹ thuật chính (Key Techniques)

*   **Event-Driven Architecture:** Kimai sử dụng rất nhiều Events và Subscribers (`src/EventSubscriber/`). Khi một Timesheet được lưu, hàng loạt sự kiện được kích hoạt để tính toán tiền lương, ngân sách dự án hoặc gửi thông báo.
*   **Custom JS Framework:** Dự án không dùng thư viện lớn nhưng xây dựng các "Widgets" JS (`assets/js/widgets/`) để quản lý trạng thái giao diện, giúp trang web mượt mà như một ứng dụng Single Page App (SPA) ở một số khu vực (như Modal forms).
*   **Xử lý dữ liệu phức tạp:**
    *   **Invoicing:** Hệ thống tạo hóa đơn (`src/Invoice/`) hỗ trợ nhiều định dạng (PDF, XLSX, DOCX) và các thuật toán tính giá khác nhau (Hourly rate, Fixed rate).
    *   **Reporting:** Các báo cáo phức tạp được tối ưu bằng các truy vấn SQL đặc thù thông qua Repository.
*   **Security & Auth:** Tích hợp sẵn LDAP, SAML, và xác thực 2 lớp (2FA) - rất hiếm thấy ở các dự án mã nguồn mở quy mô trung bình.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng đi của một yêu cầu (Request) trong Kimai thường diễn ra như sau:

1.  **Entry Point:** Yêu cầu đi vào `public/index.php`.
2.  **Routing:** Symfony Router xác định Controller tương ứng (ví dụ `TimesheetController`).
3.  **Security Layer:** Hệ thống kiểm tra User đã đăng nhập chưa và có quyền thực hiện hành động đó không (Voters).
4.  **Logic xử lý:**
    *   Controller gọi **Service** tương ứng để xử lý nghiệp vụ (ví dụ: `TimesheetService` để bắt đầu bấm giờ).
    *   **Repository** thực hiện lưu trữ vào Database.
5.  **Event Dispatching:** Một sự kiện "Timesheet started" được phát đi để các bộ phận khác (như Dashboard widget) cập nhật thời gian thực.
6.  **Response:** Giao diện được render qua **Twig** (kèm theo các CSS/JS đã được đóng gói bởi Webpack) và trả về cho trình duyệt.

**Kết luận:** Kimai là một ví dụ điển hình về cách xây dựng ứng dụng doanh nghiệp bằng PHP một cách chuyên nghiệp. Nó không chạy theo các xu hướng JS framework hào nhoáng mà tập trung vào **hiệu năng, bảo mật và khả năng bảo trì lâu dài** trên nền tảng Symfony.