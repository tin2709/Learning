Dựa trên các tệp tin mã nguồn và cấu trúc thư mục của dự án **Pixelfed**, dưới đây là phân tích chi tiết về các khía cạnh công nghệ, kiến trúc và vận hành của nền tảng này:

### 1. Công nghệ cốt lõi (Core Technology)

Pixelfed là một ứng dụng web hiện đại được xây dựng trên hệ sinh thái PHP và JavaScript:

*   **Backend Framework:** Sử dụng **Laravel (phiên bản 12.x)** - một framework PHP mạnh mẽ. Pixelfed tận dụng tối đa các tính năng của Laravel như Eloquent ORM, Queues (hàng đợi), Events, và Middleware.
*   **Ngôn ngữ lập trình:** **PHP (v8.3+)** chiếm ưu thế (54.5%), kết hợp với **Vue.js (v2)** cho phần frontend (32.2%) và **Blade templates** (13.0%).
*   **Giao thức liên hợp (Federation):** Sử dụng **ActivityPub**. Đây là "trái tim" của Pixelfed, cho phép nó giao tiếp với các nền tảng khác trong Fediverse như Mastodon hoặc Lemmy.
*   **Cơ sở dữ liệu & Lưu trữ:**
    *   **MySQL/MariaDB:** Lưu trữ dữ liệu quan hệ chính.
    *   **Redis:** Dùng để làm Cache và làm Broker cho hàng đợi (Queue).
    *   **S3/Object Storage:** Hỗ trợ lưu trữ hình ảnh trên đám mây (AWS S3, DigitalOcean Spaces...).
*   **Xử lý hình ảnh & Video:** Sử dụng **FFmpeg** cho video, **libvips** và **Intervention Image** cho việc tối ưu hóa, cắt ảnh và tạo thumbnail.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Pixelfed được thiết kế để xử lý tải cao và tính phi tập trung:

*   **Kiến trúc hướng dịch vụ (Service Layer):** Các logic phức tạp được tách khỏi Controller và đưa vào các `Services` (trong thư mục `app/Services`). Ví dụ: `ActivityPubDeliveryService`, `MediaStorageService`, `StatusService`.
*   **Xử lý bất đồng bộ (Job-Driven):** Pixelfed sử dụng hàng đợi cực kỳ dày đặc (`app/Jobs`). Mọi hành động từ xử lý ảnh, gửi thông báo đến việc gửi dữ liệu sang server khác (Federation) đều được thực hiện dưới nền thông qua **Laravel Horizon**.
*   **Kiến trúc Snowflake ID:** Sử dụng `HasSnowflakePrimary` trait để tạo ra các ID không trùng lặp theo thời gian, hỗ trợ tốt cho việc mở rộng cơ sở dữ liệu phân tán.
*   **Khả năng tương thích API:** Pixelfed không chỉ có API riêng mà còn triển khai các endpoint tương thích với **API của Mastodon** (`app/Http/Controllers/Api/MastoApi`), cho phép người dùng sử dụng các ứng dụng di động có sẵn của Mastodon.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Blurhash:** Lưu trữ một chuỗi hash ngắn của ảnh để hiển thị vùng màu mờ (blurred) trong khi ảnh gốc đang tải, giúp cải thiện trải nghiệm người dùng (UX).
*   **Content Lexer:** Hệ thống Lexer (`app/Util/Lexer`) dùng để quét văn bản, tự động nhận diện Hashtag, Mentions và các đường link (Autolink).
*   **Http Signature:** Kỹ thuật ký tên số để xác thực các yêu cầu giữa các máy chủ trong Fediverse, đảm bảo tính bảo mật và danh tính của người gửi.
*   **Tối ưu hóa hình ảnh:** Tự động nén và chuyển đổi định dạng ảnh (sang WebP) để tiết kiệm băng thông nhưng vẫn giữ được chất lượng cao cho một mạng xã hội chuyên về nhiếp ảnh.
*   **Shadow Filtering & Autospam:** Tích hợp các bộ lọc tự động để phát hiện và ngăn chặn nội dung rác hoặc tài khoản spam ngay lập tức.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

**Luồng Đăng bài (Post Creation):**
1.  Người dùng tải ảnh lên qua `ComposeController`.
2.  Hệ thống tạo một bản ghi `Media` tạm thời, thực hiện tối ưu hóa (resize, nén) thông qua `ImageOptimize` Job.
3.  Khi người dùng nhấn "Post", một `Status` được tạo.
4.  **Fan-out:** Một Job sẽ được đẩy vào hàng đợi để "phát tán" bài đăng này:
    *   Đưa vào Timeline của người theo dõi cùng server.
    *   Gửi Activity qua giao thức ActivityPub tới Inbox của các server từ xa (remote servers) mà người theo dõi đang ở đó.

**Luồng Liên hợp nhận (Federation Inbound):**
1.  Một server khác gửi một Activity (ví dụ: Like, Follow hoặc Post mới) tới `FederationController@inbox`.
2.  Hệ thống kiểm tra `Http Signature` để xác minh máy chủ gửi.
3.  Activity được đưa vào `InboxWorker` để xử lý bất đồng bộ.
4.  Dữ liệu được cập nhật vào database địa phương và thông báo cho người dùng liên quan.

**Kết luận:** Pixelfed là một dự án có kiến trúc rất bài bản, tận dụng tối đa sức mạnh của Laravel để xây dựng một hệ thống phân tán phức tạp nhưng vẫn đảm bảo được hiệu suất xử lý hình ảnh và tính mở của mạng xã hội liên hợp.