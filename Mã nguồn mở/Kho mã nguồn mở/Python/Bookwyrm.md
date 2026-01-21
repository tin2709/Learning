Dựa trên cấu trúc thư mục và nội dung các file mã nguồn của dự án **BookWyrm**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống bằng tiếng Việt:

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

BookWyrm là một ứng dụng Web phức tạp kết hợp giữa mạng xã hội và quản lý dữ liệu thư viện.
*   **Ngôn ngữ chính:** Python 3.11+.
*   **Framework Web:** **Django 5.x** (Sử dụng kiến trúc MVT - Model-View-Template).
*   **Cơ sở dữ liệu:**
    *   **PostgreSQL:** Lưu trữ dữ liệu quan hệ (Người dùng, sách, bài viết). Sử dụng `django-pgtrigger` để quản lý các trigger trong DB.
    *   **Redis:** Đóng hai vai trò:
        1.  **Broker cho Celery:** Điều phối các tác vụ nền.
        2.  **Activity Stream Manager:** Lưu trữ và quản lý dòng thời gian (feeds) của người dùng để truy xuất cực nhanh.
*   **Xử lý tác vụ nền (Asynchronous Tasks):** **Celery** & **Celery Beat**. Đây là trái tim của việc gửi email, xử lý hình ảnh và truyền tin liên minh (federation).
*   **Giao thức liên minh:** **ActivityPub** (Giao thức chuẩn W3C cho mạng xã hội phi tập trung).
*   **Frontend:** Django Templates phối hợp với **Bulma CSS** và JavaScript thuần (Vanilla JS).
*   **Container hóa:** **Docker** và **Docker Compose** (Quản lý toàn bộ stack gồm Nginx, Web, DB, Redis, Celery).

---

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của BookWyrm được thiết kế theo hướng **"Federated Monolith"** (Khối thống nhất nhưng có khả năng liên minh):

1.  **Tính phi tập trung (Decentralization):** Thay vì một server trung tâm như Goodreads, BookWyrm cho phép nhiều thực thể (instances) chạy độc lập nhưng có thể giao tiếp với nhau qua ActivityPub. Bạn ở server A có thể theo dõi và bình luận bài viết của người ở server B.
2.  **Kiến trúc hướng sự kiện (Event-driven):** Các hành động của người dùng (đăng bài, theo dõi) thường kích hoạt các Signal trong Django, sau đó đẩy vào hàng đợi Celery để xử lý bất đồng bộ, giúp giao diện người dùng luôn mượt mà.
3.  **Mô hình hóa dữ liệu sách phức tạp:** Sử dụng khái niệm **Work** (Tác phẩm) và **Edition** (Phiên bản xuất bản). Một "Work" có thể có nhiều "Edition" (bìa cứng, bìa mềm, ngôn ngữ khác nhau).
4.  **Tách biệt logic kết nối dữ liệu (Connectors):** Hệ thống có một module riêng (`bookwyrm/connectors/`) để lấy thông tin sách từ các nguồn bên ngoài (OpenLibrary, Inventaire, Finna) mà không làm ảnh hưởng đến logic lõi.

---

### 3. Các kỹ thuật then chốt (Key Techniques)

*   **Xử lý ActivityPub (`bookwyrm/activitypub/`):** Chuyển đổi các Model của Django thành định dạng JSON-LD (ActivityStreams) để gửi đi và ngược lại (Deserialize JSON nhận được thành dữ liệu local).
*   **Chữ ký số (Signatures - `bookwyrm/signatures.py`):** Mọi request gửi đi giữa các server trong mạng lưới liên minh đều được ký bằng cặp khóa RSA (Private/Public Key) để đảm bảo tính xác thực.
*   **Quản lý luồng tin bằng Redis (`bookwyrm/redis_store.py`):** Thay vì truy vấn SQL phức tạp để tạo Newfeed, BookWyrm lưu danh sách ID bài viết vào Redis Sorted Sets. Việc lấy feed chỉ đơn giản là đọc từ Redis.
*   **Hệ thống Connectors (`bookwyrm/connectors/`):** Sử dụng các lớp trừu tượng (Abstract Classes) để định nghĩa cách ánh xạ (mapping) dữ liệu từ các API khác nhau về định dạng chuẩn của BookWyrm.
*   **Xử lý hình ảnh:** Sử dụng `django-imagekit` để tự động tạo các kích thước ảnh bìa (thumbnail) khác nhau ngay khi ảnh được tải lên hoặc lấy về từ URL.

---

### 4. Tóm tắt luồng hoạt động (Project Activity Flow)

Dựa trên file `urls.py` và các module chính, luồng hoạt động diễn ra như sau:

#### A. Luồng Đăng bài (Status/Review Flow):
1.  **User action:** Người dùng viết một Review trên giao diện Web.
2.  **Processing:** View xử lý lưu vào PostgreSQL.
3.  **Signal:** Một Django Signal được kích hoạt (`add_status_on_create`).
4.  **Async Task:** Celery nhận tác vụ:
    *   Cập nhật Redis feed cho những người theo dõi cùng server.
    *   Ký tên bài viết và gửi bản tin (Activity) tới Inbox của các server khác nơi có người theo dõi.
5.  **Preview Image:** Một tác vụ Celery khác (`generate_preview_images`) tạo ảnh preview (OG image) để chia sẻ lên Twitter/Facebook.

#### B. Luồng Tìm kiếm và Nhập liệu sách (Search/Connector Flow):
1.  **Search:** Người dùng tìm một cuốn sách theo tên hoặc ISBN.
2.  **Connector Manager:** Hệ thống gọi đồng thời (async) tới các Connectors (OpenLibrary, Inventaire...).
3.  **Mapping:** Dữ liệu JSON từ bên ngoài được ánh xạ về Model `Book`/`Author` local thông qua lớp `Mapping`.
4.  **Local Copy:** Một bản sao dữ liệu được tạo trong DB local để người dùng có thể tương tác (shelve, review) mà không cần gọi API bên ngoài liên tục.

#### C. Luồng Liên minh (Federation Flow):
1.  **Incoming Activity:** Một server khác gửi một bản tin "Follow" tới `inbox/`.
2.  **Verification:** BookWyrm kiểm tra chữ ký HTTP để xác nhận danh tính server gửi.
3.  **Processing:** Nếu hợp lệ, hệ thống tự động tạo/cập nhật bản ghi người dùng từ xa (Remote User) trong DB local và gửi lại bản tin "Accept".

---

### Kết luận
Dự án BookWyrm là một ví dụ điển hình về việc sử dụng **Django** để xây dựng hệ thống phức tạp, có tính mở rộng cao và tuân thủ các tiêu chuẩn web hiện đại. Điểm mạnh nhất của nó nằm ở việc tích hợp **ActivityPub** và cách tối ưu hiệu năng dòng tin bằng **Redis**, giúp nó không chỉ là một ứng dụng web đơn lẻ mà là một phần của mạng lưới mạng xã hội rộng lớn (Fediverse).