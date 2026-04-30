Dựa trên các tệp mã nguồn và cấu trúc thư mục của dự án **Lemmy**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và quy trình hoạt động của nền tảng mạng xã hội phi tập trung này:

### 1. Công nghệ cốt lõi (Core Technology)

Lemmy được xây dựng với mục tiêu hiệu suất cao và an toàn bộ nhớ, sử dụng hệ sinh thái Rust hiện đại:

*   **Ngôn ngữ lập trình:** **Rust** (chiếm ~78%) là ngôn ngữ chính cho backend, đảm bảo tốc độ và tránh các lỗi crash phổ biến.
*   **Web Framework:** **Actix-web**, một trong những framework web nhanh nhất hiện nay, xử lý các request API và các endpoint ActivityPub.
*   **Database & ORM:** **PostgreSQL** kết hợp với **Diesel ORM**. Dự án sử dụng Diesel để đảm bảo tính an toàn kiểu dữ liệu (type-safe) khi truy vấn cơ sở dữ liệu.
*   **Giao thức phi tập trung:** **ActivityPub** (thông qua thư viện `activitypub-federation`). Đây là "xương sống" cho phép Lemmy kết nối với các server khác trong Fediverse (như Mastodon, Pleroma).
*   **Frontend (phần client đi kèm):** Sử dụng **Inferno** (một framework giống React nhưng cực nhẹ) và **TypeScript**.
*   **Xử lý hình ảnh:** Tích hợp với **pict-rs**, một dịch vụ lưu trữ và xử lý ảnh riêng biệt.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Lemmy theo mô hình **Monorepo** với các Crates được chia nhỏ theo chức năng:

*   **Tách biệt View và Logic:** Điểm đặc biệt của Lemmy là việc sử dụng rất nhiều **SQL Views** (trong `crates/db_views`). Thay vì thực hiện các phép Join phức tạp trong code Rust, dự án đẩy logic tổng hợp dữ liệu (ví dụ: đếm số like, comment, thông tin người dùng) xuống database thông qua View để tối ưu hóa hiệu suất.
*   **Kiến trúc dựa trên Activity (Activity-centric):** Mọi hành động của người dùng (đăng bài, sửa bài, like) không chỉ là một dòng trong DB mà còn là một **Activity**. Kiến trúc này được thiết kế để phục vụ việc đồng bộ hóa dữ liệu giữa các server.
*   **Phân lớp API:**
    *   `api_common`: Chứa các cấu trúc dữ liệu dùng chung.
    *   `api_crud`: Xử lý các thao tác Create, Read, Update, Delete cơ bản.
    *   `routes_v3`: Lớp định tuyến cho các phiên bản API cụ thể.
*   **Hệ thống Plugin:** Lemmy hỗ trợ các plugin viết bằng **WebAssembly (Wasm)** (sử dụng Extism), cho phép mở rộng tính năng mà không cần sửa đổi mã nguồn lõi.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Federation Queue (Hàng đợi liên hợp):** Sử dụng hàng đợi bền bỉ (persistent queue) để gửi tin nhắn đến các server khác. Nếu một server đối tác offline, Lemmy sẽ lưu lại và gửi sau.
*   **HTTP Signatures:** Kỹ thuật ký tên HTTP để xác thực các request giữa các server phi tập trung, đảm bảo rằng một bài đăng từ "Server A" thực sự là của người dùng trên "Server A".
*   **Ltree (Hierarchy Trees):** Sử dụng extension `ltree` của PostgreSQL để quản lý cấu trúc cây của các bình luận (nested comments), giúp việc truy vấn toàn bộ một "thread" bình luận trở nên cực nhanh.
*   **Rate Limiting:** Hệ thống giới hạn tần suất request (trong `crates/utils/src/rate_limit`) rất chặt chẽ để chống spam và tấn công DoS.
*   **Automated Testing:** Hệ thống kiểm thử rất đồ sộ, bao gồm cả Unit test trong Rust và các bài test liên hợp (federation tests) bằng TypeScript trong thư mục `api_tests` để mô phỏng sự tương tác giữa nhiều server (alpha, beta, gamma...).

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Một hành động đăng bài (Post) diễn ra như sau:

1.  **Client gửi Request:** Người dùng gửi một JSON request đến endpoint API (được xử lý bởi `lemmy_api`).
2.  **Xác thực & Kiểm tra:** Backend kiểm tra JWT (Token), kiểm tra quyền (người dùng có bị ban không) và lọc từ ngữ nhạy cảm (slurs filter).
3.  **Lưu trữ cục bộ:** Hành động được thực thi qua `api_crud`, dữ liệu lưu vào PostgreSQL. Tại đây, các Trigger của DB sẽ tự động cập nhật các bảng thống kê (aggregates).
4.  **Tạo Activity:** Hệ thống tạo ra một đối tượng `Create(Post)` theo chuẩn ActivityPub.
5.  **Phát tán (Broadcasting):**
    *   `lemmy_apub_send` lấy danh sách các server đang theo dõi (followers) cộng đồng đó.
    *   Worker chạy ngầm ký tên số vào Activity và gửi đến Inboxes của các server từ xa.
6.  **Xử lý phía Server nhận:** Server đối tác nhận được Activity -> xác thực chữ ký -> lưu bản sao của bài đăng vào DB của họ -> hiển thị cho người dùng của họ.

**Kết luận:** Lemmy là một hệ thống phức tạp, kết hợp giữa sự khắt khe về hiệu suất của Rust và tính mở của giao thức phi tập trung. Việc chuyển dịch logic nặng sang SQL Views và sử dụng kiến trúc ActivityPub khiến nó trở thành một trong những giải pháp diễn đàn có khả năng mở rộng tốt nhất hiện nay.