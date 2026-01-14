Chào bạn, dưới đây là phân tích chi tiết về dự án **InvoicePlane** dựa trên cấu trúc thư mục và nội dung mã nguồn bạn đã cung cấp.

---

### 1. Công nghệ cốt lõi (Core Technology)

Dự án InvoicePlane được xây dựng trên một ngăn xếp công nghệ (stack) truyền thống nhưng được cập nhật để tương thích với các tiêu chuẩn hiện đại:

*   **Backend Framework:** **CodeIgniter 3.1.13**. Đây là một PHP framework theo mô hình MVC, nổi tiếng với sự nhẹ nhàng và hiệu suất cao. Tuy nhiên, dự án đã tùy biến thêm **Modular Extensions (MX)** để hỗ trợ cấu trúc module (HMVC).
*   **Ngôn ngữ:** **PHP 8.1+**. Mã nguồn cho thấy sự xuất hiện của các Attribute như `#[AllowDynamicProperties]` (một tính năng của PHP 8.2 để tương thích ngược).
*   **Database:** **MariaDB/MySQL**. Hệ thống quản trị cơ sở dữ liệu quan hệ, được quản lý qua `CI_DB` (Query Builder của CodeIgniter).
*   **Frontend & UI:**
    *   **Bootstrap 3**: Sử dụng SASS để quản lý giao diện.
    *   **jQuery & jQuery UI**: Xử lý các tương tác phía client, datepicker, và kéo thả.
    *   **Select2**: Cho các dropdown tìm kiếm thông minh.
    *   **Dropzone.js**: Hỗ trợ tải lên tệp tin (attachments).
*   **Công cụ build & Quản lý thư viện:**
    *   **Composer**: Quản lý thư viện PHP (mPDF, PHPMailer, Stripe SDK, Dotenv).
    *   **Yarn/npm**: Quản lý thư viện Javascript.
    *   **Grunt**: Tự động hóa việc biên dịch SASS, gộp (concat) và nén (minify) các tệp CSS/JS.
*   **Môi trường:** Hỗ trợ **Docker** toàn diện với các container: Nginx, PHP-FPM, MariaDB và phpMyAdmin.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Technical & Architectural Thinking)

Kiến trúc của InvoicePlane thể hiện tư duy thiết kế thực dụng, tập trung vào khả năng mở rộng và bảo mật:

*   **Mô hình HMVC (Hierarchical Model-View-Controller):** Thay vì để tất cả Controller/Model vào một thư mục lớn, dự án chia thành các module riêng biệt trong `application/modules/` (như `clients`, `invoices`, `quotes`). Mỗi module tự quản lý logic, view và model của riêng mình, giúp code dễ bảo trì và cô lập lỗi.
*   **Hệ thống Controller phân tầng (Inheritance):**
    *   `Base_Controller`: Thiết lập các cấu hình chung, load helper và cấu hình hệ thống.
    *   `User_Controller`: Kiểm tra quyền đăng nhập cơ bản.
    *   `Admin_Controller`: Dành cho các tác vụ quản trị, tích hợp thêm logic lọc đầu vào (XSS cleaning).
    *   `Guest_Controller`: Dành riêng cho giao diện khách hàng (Client Portal).
*   **Tư duy cấu hình tập trung:** Sử dụng thư viện `phpdotenv` để đọc cấu hình từ tệp `ipconfig.php` (tương đương `.env`). Điều này giúp tách biệt mã nguồn và thông tin cấu hình nhạy cảm (DB, URL, Encryption Key).
*   **Tách biệt Logic Tính toán:** Có sự phân chia rõ ràng giữa dữ liệu lưu trữ và dữ liệu hiển thị thông qua các Model Amount (như `Mdl_invoice_amounts`), giúp việc tính toán thuế, giảm giá trở nên chính xác và dễ kiểm tra (audit).

---

### 3. Các kỹ thuật chính nổi bật (Key Technical Highlights)

*   **Hệ thống e-Invoicing (Hóa đơn điện tử):** Đây là điểm mạnh vượt trội. Dự án hỗ trợ các tiêu chuẩn hóa đơn điện tử Châu Âu như **ZUGFeRD** và **Factur-X**. Kỹ thuật này bao gồm việc tạo file XML tương ứng và nhúng trực tiếp vào file PDF (PDF/A-3).
*   **Xử lý File an toàn (Secure File Handling):** Qua tệp `file_security_helper.php`, dự án áp dụng các kỹ thuật:
    *   Kiểm tra **Path Traversal**: Ngăn chặn truy cập file ngoài thư mục cho phép bằng cách dùng `realpath` và kiểm tra tiền tố đường dẫn.
    *   **Hóa băm (Hashing)**: Sử dụng SHA256 để định danh file trong log thay vì ghi trực tiếp tên file do người dùng cung cấp (tránh Log Injection).
    *   **Sanitize Filenames**: Loại bỏ các ký tự điều khiển và ký tự đặc biệt khỏi tiêu đề HTTP để tránh tấn công Response Splitting.
*   **Custom Fields & Custom Values:** Cho phép người dùng thêm các trường dữ liệu tùy chỉnh vào Clients, Invoices mà không cần thay đổi cấu trúc bảng database gốc. Hệ thống sử dụng một bảng meta-data để lưu trữ các giá trị này.
*   **Hệ thống Template linh hoạt:** Sử dụng PHP thuần trong các View Template cho PDF và Email, cho phép người dùng tùy biến giao diện hóa đơn sâu sắc qua thư mục `invoice_templates`.
*   **Đa ngôn ngữ (i18n):** Tích hợp sâu thông qua helper `trans_helper.php` và quản lý bản dịch qua Crowdin, hỗ trợ chuyển đổi ngôn ngữ dựa trên cài đặt của từng khách hàng.

---

### 4. Tóm tắt luồng hoạt động của dự án (Project Workflow)

1.  **Giai đoạn Khởi tạo (Initialization):**
    *   Người dùng truy cập, `index.php` load thư viện tự động (Autoload) và cấu hình từ `ipconfig.php`.
    *   Hệ thống kiểm tra trạng thái cài đặt (`SETUP_COMPLETED`). Nếu chưa, nó sẽ chuyển hướng đến module `setup`.

2.  **Giai đoạn Nghiệp vụ (Business Logic):**
    *   **Quản lý Danh mục:** Người dùng tạo Clients, Products, Tax Rates.
    *   **Tạo Chứng từ:** Khi tạo một Invoice, hệ thống sẽ gọi `Ajax.php` để xử lý các tương tác thời gian thực (như chọn khách hàng, tính toán lại tổng tiền khi thêm item).
    *   **Tính toán (Calculation):** Khi lưu hóa đơn, hệ thống thực hiện tính toán qua các Model nghiệp vụ, áp dụng các quy tắc thuế (Tax rules) và giảm giá (Discount logic) được cấu hình trong hệ thống.

3.  **Giai đoạn Xuất bản & Gửi đi (Output & Distribution):**
    *   **Tạo PDF:** Sử dụng thư viện **mPDF** để chuyển đổi giao diện HTML thành PDF. Nếu khách hàng có kích hoạt e-invoice, hệ thống sẽ chạy thêm logic tạo XML và nhúng vào PDF.
    *   **Gửi Email:** Qua helper `mailer_helper`, hệ thống lấy template email, thay thế các tag (như `{{{client_name}}}`) bằng dữ liệu thật và gửi kèm file PDF qua SMTP hoặc PHPMailer.

4.  **Giai đoạn Thanh toán (Payment):**
    *   Khách hàng nhận được liên kết đến Guest Portal. Họ có thể thanh toán qua các cổng tích hợp sẵn như **Stripe** hoặc **PayPal**.
    *   Sau khi thanh toán thành công, hệ thống cập nhật trạng thái hóa đơn và ghi log vào bảng `ip_payments`.

Tóm lại, InvoicePlane là một dự án PHP truyền thống nhưng có cấu trúc rất chặt chẽ, đặc biệt chú trọng vào các tiêu chuẩn kế toán và bảo mật tệp tin, phù hợp cho việc tự triển khai (self-hosted) cho các doanh nghiệp vừa và nhỏ.