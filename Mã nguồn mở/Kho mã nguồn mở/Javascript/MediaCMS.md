Dưới đây là phân tích chi tiết về dự án **MediaCMS** dựa trên cấu trúc thư mục và nội dung mã nguồn bạn đã cung cấp:

---

### 1. Phân tích Công nghệ Cốt lõi (Core Technology Stack)

MediaCMS là một hệ thống quản lý nội dung video hiện đại, kết hợp giữa sức mạnh xử lý phía server của Python và tính tương tác linh hoạt của React.

*   **Backend (Hậu phương):**
    *   **Framework:** Django 5.x (Python 3.11+). Sử dụng Django Rest Framework (DRF) để cung cấp API.
    *   **Cơ sở dữ liệu:** PostgreSQL (Lưu trữ metadata), Redis (Làm cache và Message Broker cho Celery).
    *   **Xử lý tác vụ ẩn (Async Tasks):** Celery. Đây là "trái tim" xử lý việc chuyển mã (transcoding) video nặng nề mà không làm treo web.
    *   **Xử lý Media:** FFmpeg (Chuyển mã video/audio), Bento4 (Đóng gói chuẩn HLS để stream adaptive), ImageMagick (Tạo thumbnail/sprite).
*   **Frontend (Tiền phương):**
    *   **Thư viện chính:** React JS kết hợp với TypeScript (đang dần chuyển đổi từ JS sang TS).
    *   **Trình phát video:** Video.js (được tùy biến sâu để hỗ trợ đa độ phân giải, tốc độ phát, và phụ đề).
    *   **Styling:** SCSS và Tailwind CSS (trong các tool mới).
*   **DevOps & Deployment:**
    *   **Containerization:** Docker & Docker Compose (Hỗ trợ nhiều cấu hình từ dev đến production với SSL/LetsEncrypt).
    *   **Web Server:** Nginx (Phục vụ file tĩnh và làm Reverse Proxy), uWSGI (Application Server).
    *   **Giám sát:** Supervisord (Quản lý các tiến trình web, celery trong container).

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Engineering & Architecture)

MediaCMS không đi theo hướng Single Page Application (SPA) thuần túy mà chọn mô hình **Hybrid Architecture**:

*   **Mô hình kết hợp:** Django đóng vai trò quản lý Route và Template chính, sau đó nhúng các component React vào từng trang cụ thể (như trang xem video, trang upload). Điều này giúp tận dụng SEO và hệ thống Admin cực mạnh của Django trong khi vẫn có trải nghiệm mượt mà của React.
*   **Thiết kế Modular:**
    *   `files/`: Quản lý toàn bộ logic về media (models, views, tasks).
    *   `users/`: Quản lý người dùng và phân quyền.
    *   `actions/`: Xử lý Like, Dislike, Report.
    *   `rbac/` & `identity_providers/`: Hệ thống phân quyền dựa trên vai trò nâng cao và kết nối SSO (SAML/Entra ID).
*   **Tư duy Asynchronous (Bất đồng bộ):** Kiến trúc tách biệt hàng đợi tác vụ (`short_tasks` cho việc tạo thumbnail, `long_tasks` cho việc chuyển mã video) giúp hệ thống có thể mở rộng (scale) bằng cách thêm nhiều Worker máy chủ khác nhau.

---

### 3. Các kỹ thuật chính nổi bật (Key Technical Highlights)

1.  **Chiến lược Chuyển mã Thông minh (Smart Transcoding):**
    *   **Chunking:** Video lớn được chia thành các đoạn nhỏ (ví dụ 4 phút) để mã hóa song song, sau đó ghép lại. Kỹ thuật này giảm rủi ro khi lỗi giữa chừng và tăng tốc độ xử lý trên hệ thống đa nhân.
    *   **HLS (HTTP Live Streaming):** Tự động tạo ra các file m3u8 và các phân đoạn .ts, cho phép trình phát tự động thay đổi chất lượng video (360p, 720p, 1080p) dựa trên tốc độ mạng của người dùng.
2.  **Hệ thống Phân quyền RBAC & SAML:**
    *   Hỗ trợ đăng nhập một lần (SSO) qua SAML (như Microsoft Entra ID).
    *   Có khả năng ánh xạ (mapping) các thuộc tính từ SAML trực tiếp vào các nhóm quyền (Groups) và chuyên mục (Categories) trong CMS.
3.  **AI Integration:** Tích hợp OpenAI Whisper để tự động tạo phụ đề (transcription) từ giọng nói trong video ngay tại server local.
4.  **Sprite Thumbnail:** Kỹ thuật tạo một file ảnh lớn chứa nhiều khung hình nhỏ (sprites) và sử dụng CSS/JS để hiển thị preview khi người dùng rê chuột trên thanh progress bar (giống YouTube).
5.  **Video Editor & Trimmer:** Cung cấp bộ công cụ trực tiếp trên trình duyệt để cắt video, trích xuất đoạn nhỏ mà không cần phần mềm bên ngoài.

---

### 4. Tóm tắt luồng hoạt động (Project Workflow Summary)

Dưới đây là hành trình của một video từ khi upload đến khi tới tay người xem:

1.  **Giai đoạn Upload:**
    *   Người dùng upload file qua giao diện React (sử dụng `FineUploader` hỗ trợ resume nếu mất mạng).
    *   Django nhận file, lưu vào thư mục `original/` và tạo bản ghi trong DB với trạng thái `pending`.
2.  **Giai đoạn Xử lý (Hậu kỳ):**
    *   Một tín hiệu (Signal) được gửi đến Celery.
    *   **Task 1:** Kiểm tra định dạng, lấy thông tin bitrate, độ phân giải (ffprobe).
    *   **Task 2:** Tạo ảnh đại diện (Thumbnail) và file ảnh Sprite.
    *   **Task 3 (Nặng nhất):** Chạy FFmpeg để tạo các phiên bản độ phân giải thấp hơn (H.264/AAC).
    *   **Task 4:** Bento4 đóng gói các bản video đã chuyển mã sang chuẩn HLS.
    *   **Task 5 (Nếu bật AI):** Whisper quét âm thanh để tạo file phụ đề .vtt.
3.  **Giai đoạn Phân phối:**
    *   Người xem truy cập trang video. React component gọi API lấy link `.m3u8`.
    *   Video.js nhận luồng HLS, Nginx phục vụ các file nhỏ `.ts` từ đĩa cứng.
    *   Mọi tương tác (Like, Comment, Time-stamp) được xử lý qua API RESTful và cập nhật thời gian thực vào PostgreSQL.

**Kết luận:** MediaCMS là một dự án có độ hoàn thiện rất cao về mặt kỹ thuật, phù hợp cho các tổ chức muốn tự vận hành nền tảng video riêng biệt (như trường học, doanh nghiệp) thay vì phụ thuộc vào YouTube hay Vimeo.