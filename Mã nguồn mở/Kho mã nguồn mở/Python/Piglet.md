Dưới đây là phân tích chi tiết về dự án **Piglet** dựa trên mã nguồn và tài liệu bạn đã cung cấp:

### 1. Công Nghệ Cốt Lõi (Core Technologies)

Dự án Piglet được xây dựng trên một ngăn xếp công nghệ (stack) mã nguồn mở phổ biến, tối ưu cho việc triển khai tự lưu trữ (self-hosting):

*   **Backend API:** Sử dụng **FastAPI** (Python). Đây là một framework hiện đại, hiệu suất cao, dùng để xây dựng các RESTful API với tính năng tự động tạo tài liệu OpenAPI (Swagger).
*   **Web Interface (Frontend):** Sử dụng **Flask** (Python) kết hợp với công cụ tạo mẫu **Jinja2**. Flask đóng vai trò là một "Client" gọi các API từ dịch vụ FastAPI để hiển thị dữ liệu cho người dùng.
*   **Cơ sở dữ liệu (Database):** Mặc định sử dụng **MariaDB** (thông qua Docker). Mã nguồn cũng cho thấy sự chuẩn bị cho việc hỗ trợ **SQLite** để tinh giản bộ máy.
*   **Quản lý tiến trình:** Sử dụng **Supervisor**. Công cụ này cho phép chạy đồng thời ứng dụng Flask, API FastAPI, và các trình lập lịch (scheduler) bên trong cùng một Docker container.
*   **Bộ nhớ đệm & Session:** **Redis**. Được sử dụng để lưu trữ phiên làm việc (Session) của Flask và đóng vai trò là "Broker" cho các tác vụ xếp hàng.
*   **Tác vụ nền & Lập lịch:** **Celery**. Dùng để xử lý các lệnh chi tiêu định kỳ (Recurring Orders) hoặc chi tiêu trong tương lai (Future Spends).
*   **Giao diện người dùng:** HTML/CSS/JS thuần, kết hợp với **Bootstrap 5**, **Chart.js** (để vẽ biểu đồ tài chính) và **DataTables**.

### 2. Tư Duy Kiến Trúc (Architectural Thinking)

Dự án áp dụng kiến trúc **Tách biệt mối quan tâm (Separation of Concerns)** rõ rệt:

*   **Kiến trúc hướng API (API-First Design):** Toàn bộ logic nghiệp vụ (tính toán báo cáo, quản lý người dùng, xử lý giao dịch) nằm tại thư mục `webapp/api`. Flask WebUI chỉ đóng vai trò là lớp hiển thị. Điều này cho phép mở rộng sang ứng dụng di động trong tương lai mà không cần viết lại logic.
*   **Cấu trúc đa người dùng & Chia sẻ (Multi-tenancy & Sharing):** Hệ thống được thiết kế quanh thực thể `Budget` (Ngân sách). Một người dùng có thể sở hữu nhiều ngân sách và một ngân sách có thể có nhiều người dùng cùng quản lý. Việc chia sẻ được thực hiện thông qua mã mời hoặc email.
*   **Tư duy Docker-Centric:** Dự án được đóng gói hoàn toàn trong Docker, giúp việc cài đặt các thành phần phức tạp (Redis, MariaDB, Python venv) trở nên đơn giản chỉ với một lệnh `docker-compose up`.

### 3. Các Kỹ Thuật Chính (Key Techniques)

*   **Xác thực bảo mật:** Sử dụng mã hóa **SHA256** kết hợp với **Salt** ngẫu nhiên cho mật khẩu. API sử dụng **JWT (JSON Web Token)** để xác thực quyền truy cập giữa WebUI và Backend.
*   **Xử lý dữ liệu nhập (Data Import):** Tích hợp kỹ thuật nhận diện dấu phân cách tự động và chuẩn hóa dữ liệu khi người dùng tải lên tệp CSV để nhập dữ liệu chi tiêu hàng loạt.
*   **Hệ thống thông báo thông minh:** Kết hợp giữa thông báo trên giao diện web (Web Notifications) và gửi email qua giao thức SMTP (STARTTLS). Người dùng có thể tùy chỉnh nhận thông báo cho từng loại sự kiện (thêm chi tiêu, xóa hạng mục...).
*   **Quản lý Database Schema:** Sử dụng cơ chế tự động nâng cấp schema (Database migration) thông qua các file SQL đánh số phiên bản trong thư mục `config/dbschema/update`.

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

1.  **Luồng Đăng nhập/Xác thực:** Người dùng đăng nhập qua Flask WebUI -> Flask gọi API `/token` của FastAPI -> FastAPI kiểm tra DB, trả về JWT -> Flask lưu JWT vào Session (lưu trữ tại Redis).
2.  **Luồng Thêm chi tiêu (Order):** 
    *   Người dùng điền form -> Flask gửi yêu cầu kèm JWT tới API `/order/new`.
    *   FastAPI kiểm tra quyền của người dùng đối với `budget_id` đó.
    *   Nếu hợp lệ, dữ liệu được ghi vào MariaDB.
    *   Hệ thống kích hoạt thông báo cho các thành viên khác trong cùng ngân sách.
3.  **Luồng Xử lý chi tiêu định kỳ:** Celery Worker chạy ngầm sẽ kiểm tra các lịch trình trong bảng `pig_schedules`. Khi đến hạn, nó sẽ tự động tạo một bản ghi chi tiêu mới mà không cần người dùng can thiệp.
4.  **Luồng Báo cáo & Phân tích:** Khi người dùng vào trang Analyze -> Flask gọi API `/reports` và `/graph`. Backend sẽ thực hiện các truy vấn SQL phức tạp (SUM, GROUP BY theo tháng/hạng mục) và tính toán nợ nần giữa các thành viên (Debt calculation), sau đó trả về dữ liệu JSON để Chart.js vẽ biểu đồ.

### Tổng kết
**Piglet** là một dự án quản lý tài chính gia đình có kiến trúc khá chặt chẽ cho một ứng dụng tự lưu trữ. Việc sử dụng FastAPI làm lõi API và Celery cho các tác vụ nền cho thấy tác giả chú trọng vào tính mở rộng và khả năng xử lý tự động. Tuy nhiên, việc chạy tất cả các tiến trình Python trong cùng một Docker container (qua Supervisor) là cách làm phù hợp cho hạ tầng nhỏ nhưng cần lưu ý nếu muốn scale lên các cụm máy chủ lớn.