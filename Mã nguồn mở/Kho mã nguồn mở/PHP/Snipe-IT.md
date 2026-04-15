Dựa trên cấu trúc thư mục và nội dung các tệp tin của dự án **Snipe-IT**, dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống quản trị tài sản mã nguồn mở hàng đầu này:

### 1. Công nghệ cốt lõi (Core Technologies)
Snipe-IT là một ứng dụng Web hiện đại chạy trên ngăn xếp (stack) PHP:
*   **Framework chính:** **Laravel 11** (phiên bản mới nhất tính đến thời điểm file được tạo), tận dụng tối đa các tính năng như Eloquent ORM, Service Providers, Migrations, và Artisan CLI.
*   **Frontend:**
    *   **Blade Template Engine:** Render phía server.
    *   **Livewire:** Cho phép tạo các thành phần giao diện tương tác cao (reactive) bằng PHP mà không cần viết quá nhiều JavaScript (ví dụ: `app/Livewire/CategoryEditForm.php`).
    *   **Laravel Mix (Webpack):** Quản lý asset (CSS, JS), sử dụng Less cho stylesheet.
*   **Cơ sở dữ liệu:** Hỗ trợ **MariaDB** và **MySQL** (thể hiện qua `docker-compose.yml` và các file `tests-mysql.yml`).
*   **API & Auth:**
    *   **Laravel Passport:** Cung cấp OAuth2 cho API.
    *   **SAML & LDAP:** Hỗ trợ xác thực doanh nghiệp tập trung (thể hiện qua `app/Services/Saml.php` và `app/Console/Commands/LdapSync.php`).
*   **Hạ tầng:** Docker (nhiều biến thể: Ubuntu, Alpine), Vagrant, và Ansible để tự động hóa triển khai.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Hệ thống được thiết kế theo tư duy **Modular và Action-Oriented**:

*   **Action Pattern:** Thay vì viết logic nghiệp vụ phức tạp trong Controller, Snipe-IT sử dụng thư mục `app/Actions`. Mỗi Action đại diện cho một tác vụ duy nhất (ví dụ: `DestroyCategoryAction`, `CreateCheckoutRequestAction`). Điều này giúp mã nguồn dễ đọc, dễ kiểm thử (Unit Test) và tái sử dụng.
*   **Polymorphic Relationships (Quan hệ đa hình):** Đây là "xương sống" của việc quản lý tài sản. Một tài sản (Asset) có thể được giao (Checkout) cho một Người dùng, một Địa điểm, hoặc một Tài sản khác. Kỹ thuật này thể hiện qua các cột `assigned_to` và `assigned_type` trong cơ sở dữ liệu.
*   **Company Scoping (Phạm vi công ty):** Snipe-IT hỗ trợ đa công ty (Multi-tenancy) bằng cách sử dụng các **Global Scopes** (như `CompanyableScope.php`). Điều này đảm bảo quản trị viên của công ty A không thể thấy tài sản của công ty B.
*   **Kiến trúc Sự kiện (Event-Driven):** Sử dụng Laravel Events/Listeners (ví dụ: `CheckoutableCheckedIn.php`) để xử lý các tác vụ phụ như gửi email thông báo hoặc ghi log sau khi một hành động chính hoàn tất.

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)
*   **Sử dụng Traits mạnh mẽ:** Snipe-IT dùng Traits để chia sẻ code giữa các Model khác nhau.
    *   `Searchable`: Tự động tạo truy vấn tìm kiếm cho Model.
    *   `Loggable`: Tự động ghi chép mọi thay đổi vào bảng `action_logs`.
    *   `Presentable`: Sử dụng **Presenter Pattern** (thư mục `app/Presenters`) để tách biệt logic hiển thị (định dạng tiền tệ, ngày tháng) ra khỏi Model.
*   **Database Migrations & Factories:** Hệ thống có lịch sử migration cực kỳ chi tiết (từ năm 2012 đến nay), cho thấy sự tiến hóa liên tục. Hệ thống Factory và Seeder rất hoàn thiện, phục vụ tốt cho việc kiểm thử tự động.
*   **Observers:** Sử dụng để theo dõi các thay đổi của Model (ví dụ: `AssetObserver.php`) để thực hiện các tác vụ như cập nhật số lượng tồn kho hoặc trạng thái liên quan một cách tự động.
*   **Custom Fields:** Hệ thống cho phép người dùng tự định nghĩa các trường dữ liệu cho tài sản, một kỹ thuật xử lý database linh hoạt (thư mục `app/Models/CustomField.php`).

### 4. Luồng hoạt động của hệ thống (System Workflow)

Luồng nghiệp vụ chính của Snipe-IT xoay quanh **Vòng đời tài sản**:

1.  **Nhập liệu (Importing):** Dữ liệu có thể vào hệ thống qua giao diện Web, API hoặc CLI (Artisan). Snipe-IT có bộ Importer rất mạnh (`app/Importer`) để xử lý file CSV lớn.
2.  **Xác thực & Đồng bộ:** Hệ thống đồng bộ người dùng từ LDAP/Active Directory hàng ngày.
3.  **Checkout (Giao tài sản):**
    *   User yêu cầu tài sản -> Admin phê duyệt.
    *   Hệ thống tạo một bản ghi trong `action_logs`.
    *   Nếu yêu cầu EULA, hệ thống tạo `CheckoutAcceptance` và gửi email cho User.
4.  **Audit (Kiểm kê):** Admin thực hiện quét tài sản định kỳ. Các lệnh Artisan (CLI) hỗ trợ tìm kiếm tài sản sắp hết hạn bảo hành hoặc cần bảo trì.
5.  **Thông báo (Notifications):** Hệ thống sử dụng Queue để gửi email thông báo (Mailable) nhằm không làm chậm trải nghiệm người dùng.
6.  **Báo cáo (Reporting):** Dữ liệu từ các bảng `Actionlog` và `Assets` được tổng hợp qua các `Transformers` để xuất ra các báo cáo dạng JSON/CSV/PDF.

### Tổng kết
Snipe-IT là một ví dụ điển hình về việc ứng dụng Laravel để giải quyết bài toán quản trị thực tế. Nó kết hợp giữa sự linh hoạt của **quan hệ đa hình**, tính bảo mật của **scopes**, và tính sạch sẽ của **Action pattern** để tạo ra một hệ thống có thể mở rộng từ một văn phòng nhỏ đến một tập đoàn đa quốc gia.