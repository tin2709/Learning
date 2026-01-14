Dolibarr là một hệ thống ERP & CRM mã nguồn mở nổi tiếng dành cho doanh nghiệp vừa và nhỏ. Dựa trên cấu trúc thư mục và nội dung tệp tin bạn cung cấp, dưới đây là phân tích chi tiết về kiến trúc kỹ thuật và cách thức hoạt động của dự án này.

---

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ chính:** **PHP** chiếm ưu thế tuyệt đối (hơn 92%). Đây là lựa chọn chiến lược để đảm bảo tính tương thích cao với hầu hết các hosting server.
*   **Cơ sở dữ liệu:** Hỗ trợ đa nền tảng thông qua lớp trừu tượng hóa (Abstraction Layer), chủ yếu là **MySQL/MariaDB** và **PostgreSQL**. Dự án cung cấp sẵn các script migration (`htdocs/install/mysql/migration/`) để nâng cấp dữ liệu qua từng phiên bản.
*   **Frontend:** Sử dụng **Vanilla JS** kết hợp với **jQuery** và **jQuery UI**. Dolibarr không sử dụng các framework hiện đại như React hay Vue để tránh sự phụ thuộc vào build tools phức tạp và đảm bảo tính ổn định dài hạn (long-term stability).
*   **Thư viện bên thứ ba:** Tích hợp trực tiếp các thư viện lớn vào thư mục `htdocs/includes/` (thay vì phụ thuộc hoàn toàn vào Composer ở runtime), ví dụ: **TCPDF** (xuất PDF), **CKEditor** (soạn thảo văn bản), **Stripe** (thanh toán), **SabreDAV** (kết nối lịch/tệp tin).

### 2. Kỹ thuật và Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Dolibarr được thiết kế theo hướng **"Module-Centric"** và **"Long-term Maintainability"**:

*   **Tính mô-đun hóa cực cao:** Mọi tính năng (Hóa đơn, Khách hàng, Sản phẩm, Kho...) đều là một module độc lập. Người dùng có thể bật/tắt tùy ý. Mỗi module thường có cấu trúc:
    *   `class/`: Chứa logic nghiệp vụ (Business Logic).
    *   `admin/`: Trang cấu hình cho admin.
    *   `tpl/`: Các tệp giao diện (Template).
*   **Cơ chế Hook & Trigger (Event-Driven):** 
    *   **Triggers (`htdocs/core/triggers/`):** Cho phép thực hiện các hành động tự động khi có một sự kiện xảy ra (ví dụ: tự động gửi email khi hóa đơn được xác nhận).
    *   **Hooks:** Cho phép các lập trình viên bên thứ ba can thiệp vào luồng xử lý hoặc giao diện của core mà không cần sửa mã nguồn gốc.
*   **Tư duy không framework (No-Framework Policy):** Dolibarr tự xây dựng các thành phần cốt lõi (Router, DB Wrapper, User Auth). Điều này giúp dự án không bị lỗi thời khi các framework (như Laravel/Symfony) thay đổi phiên bản lớn, giúp mã nguồn có thể chạy liên tục hơn 10-20 năm.

### 3. Các kỹ thuật chính nổi bật

*   **DoliDB (Database Abstraction):** Hệ thống không viết SQL thuần cho một loại DB duy nhất mà thông qua tệp `htdocs/core/db/DoliDB.class.php` để đảm bảo lệnh SQL có thể chạy trên cả MySQL và PostgreSQL.
*   **Cấu trúc dữ liệu mở rộng (Extrafields):** Một kỹ thuật mạnh mẽ cho phép người dùng thêm các trường dữ liệu tùy chỉnh vào các đối tượng (Hóa đơn, Sản phẩm) ngay từ giao diện admin mà không cần can thiệp vào code.
*   **REST API:** Sử dụng thư viện **Restler** để tự động tạo ra các endpoint API từ các lớp PHP nghiệp vụ, giúp tích hợp dễ dàng với các hệ thống bên ngoài (như Zapier được nhắc tới trong thư mục `dev/examples/zapier/`).
*   **Bảo mật:** 
    *   Kiểm tra token CSRF (`MAIN_SECURITY_CSRF_WITH_TOKEN`).
    *   Lọc dữ liệu đầu vào nghiêm ngặt để chống SQL Injection và XSS.
    *   Hệ thống phân quyền (Permissions) chi tiết đến từng hành động (Xem, Sửa, Xóa) cho từng module.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng xử lý một yêu cầu (Request) thông thường trong Dolibarr:

1.  **Khởi tạo (Bootstrapping):** Mọi tệp tin PHP đều gọi `master.inc.php`. Tệp này thực hiện:
    *   Kiểm tra tệp cấu hình `conf.php`.
    *   Kết nối Database.
    *   Khởi tạo biến `$user` (thông tin người dùng), `$conf` (cấu hình hệ thống), và `$langs` (ngôn ngữ).
2.  **Kiểm tra quyền:** Hệ thống kiểm tra xem người dùng hiện tại có quyền truy cập vào chức năng/module đó hay không thông qua biến `$user->rights`.
3.  **Xử lý nghiệp vụ (Business Logic):** Gọi đến các class đối tượng. Ví dụ, nếu bạn tạo hóa đơn, code sẽ gọi `$facture = new Facture($db);` và sau đó là `$facture->create($user);`.
4.  **Kích hoạt Trigger:** Sau khi dữ liệu được lưu thành công, các trigger liên quan sẽ được kích hoạt để thực hiện các tác vụ phụ.
5.  **Hiển thị (Rendering):** Dữ liệu được đưa vào các tệp template (`.tpl.php`) kết hợp với HTML/CSS của theme để hiển thị cho người dùng.

### 5. Đánh giá thư mục `dev/` (Công cụ cho lập trình viên)

Thư mục `dev/` mà bạn cung cấp cho thấy quy trình phát triển rất chuyên nghiệp:
*   **`initdata/`:** Chứa các script để tạo dữ liệu mẫu số lượng lớn, phục vụ việc kiểm thử hiệu năng (Load test).
*   **`setup/`:** Chứa cấu hình cho các công cụ kiểm tra chất lượng code như **PHPCodeSniffer** (kiểm tra chuẩn coding style) và các file cấu hình cho web server (Apache, Nginx).
*   **`tools/`:** Các công cụ chuyển đổi DB (MySQL sang PostgreSQL) và sửa lỗi định dạng tệp tin.

**Kết luận:** Dolibarr là một ví dụ điển hình của việc xây dựng phần mềm doanh nghiệp bền bỉ. Nó ưu tiên tính ổn định, khả năng tương thích ngược và sự đơn giản trong triển khai hơn là sử dụng các công nghệ thời thượng nhưng mau thay đổi.