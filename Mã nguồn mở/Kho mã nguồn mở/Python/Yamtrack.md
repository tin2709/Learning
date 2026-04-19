Yamtrack là một hệ thống theo dõi đa phương tiện (Media Tracker) tự lưu trữ (self-hosted) rất tinh tế, được xây dựng trên nền tảng **Django (Python)**. Qua việc phân tích mã nguồn bạn cung cấp, dưới đây là cái nhìn sâu sắc về kiến trúc và kỹ thuật của dự án:

### 1. Công nghệ cốt lõi (Core Stack)
*   **Framework:** Django 5.x (Sử dụng tính năng mới nhất như `MonitorField`, `GeneratedField`).
*   **Database:** Hỗ trợ linh hoạt SQLite (cho cá nhân) và PostgreSQL (cho quy mô lớn).
*   **Giao diện:** Tailwind CSS kết hợp với **HTMX**. Đây là lựa chọn thông minh giúp ứng dụng có cảm giác như một Single Page Application (SPA) nhưng vẫn giữ được sự đơn giản của Server-side Rendering.
*   **Xử lý nền (Background Tasks):** Celery + Redis + Celery Beat. Dùng để đồng bộ metadata, gửi thông báo và dọn dẹp dữ liệu.
*   **Lịch sử & Audit:** `django-simple-history` được dùng cực kỳ triệt để để theo dõi từng thay đổi nhỏ nhất trong tiến trình xem/đọc của người dùng.

### 2. Tư duy Kiến trúc (Architecture Thinking)

#### A. Mô hình Dữ liệu Phân cấp (The Item-Media Pattern)
Yamtrack tách biệt rõ ràng giữa "Thông tin định danh" và "Dữ liệu người dùng":
*   **`Item`:** Lưu trữ thông tin chung từ API (TMDB ID, Title, Image, Source). Một `Item` có thể được dùng chung bởi nhiều người dùng.
*   **`BasicMedia` (và các lớp con như `Anime`, `Movie`, `Game`...):** Lưu trữ trạng thái của riêng từng người dùng (Score, Status, Progress, Start/End Date).
*   **Cấu trúc TV Show đặc thù:** Hệ thống xử lý phân cấp `TV` -> `Season` -> `Episode`. Điều này cho phép theo dõi tiến độ đến từng tập phim một cách chi tiết.

#### B. Chiến lược Provider (Abstraction Layer)
Thay vì gọi trực tiếp API ở Views, Yamtrack sử dụng một lớp trung gian trong `app/providers/`:
*   `services.py` đóng vai trò là **Gateway**.
*   Các file như `tmdb.py`, `mal.py`, `bgg.py` là các **Adapters** riêng biệt. 
*   Thiết kế này cho phép dễ dàng thêm nguồn dữ liệu mới (ví dụ: vừa thêm BoardGameGeek trong migration gần nhất) mà không làm ảnh hưởng đến logic hiển thị.

### 3. Kỹ thuật Lập trình Chính

#### A. Tối ưu hóa Hiệu năng (Performance Tuning)
*   **Cache:** Sử dụng Django Cache (thường là Redis) để lưu kết quả tìm kiếm từ API ngoại vi, tránh bị rate limit và tăng tốc độ phản hồi.
*   **Database Query:** Sử dụng `select_related` và `prefetch_related` rất kỹ lưỡng trong `statistics.py` và `helpers.py` để tránh lỗi N+1 query khi hiển thị danh sách media lớn.
*   **Bulk Operations:** Trong các file migration (ví dụ `0045`), tác giả sử dụng `bulk_create` và `iterator()` để xử lý hàng ngàn dòng dữ liệu mà không làm tràn bộ nhớ RAM.

#### B. UI/UX động với HTMX
Mã nguồn trong `views.py` thường xuyên kiểm tra `request.headers.get("HX-Request")`. Nếu là yêu cầu từ HTMX, server chỉ trả về một đoạn mã HTML nhỏ (Partial Template) thay vì toàn bộ trang, giúp trải nghiệm cực kỳ mượt mà.

#### C. Hệ thống Lịch sử (Timeline Logic)
`history_processor.py` là một module thú vị. Nó phân tích sự khác biệt (diff) giữa các phiên bản của một bản ghi để tạo ra các dòng mô tả tự nhiên như: *"Bắt đầu xem vào ngày..."* hoặc *"Đã thay đổi điểm số từ 8 thành 9"*.

### 4. Luồng hoạt động của hệ thống (System Flow)

1.  **Luồng Tìm kiếm:** 
    *   User nhập từ khóa -> View gọi `services.search` -> Gọi API (TMDB/MAL...) -> `enrich_items_with_user_data` kiểm tra xem trong DB user đã theo dõi mục này chưa -> Trả về kết quả kèm trạng thái "Đã theo dõi" nếu có.
    
2.  **Luồng Theo dõi (Tracking):**
    *   User nhấn "Track" -> HTMX gọi `track_modal` -> Trả về Form tùy chỉnh theo loại media -> User Save -> `media_save` xử lý -> Tạo/Cập nhật `Item` và `BasicMedia` -> Cập nhật UI.

3.  **Luồng Đồng bộ (Syncing):**
    *   Celery Beat định kỳ quét các đầu mục -> Gọi API cập nhật thông tin mới (như ngày phát hành tập mới) -> Lưu vào bảng `Event` -> Gửi thông báo qua Apprise nếu có thay đổi.

### 5. Đánh giá chất lượng Code
*   **Tính ổn định:** Migration được quản lý rất chặt chẽ, có cả các migration "sửa lỗi" cho các migration trước đó (ví dụ `0060` sửa lỗi của `0057`), cho thấy dự án có quy trình kiểm soát dữ liệu tốt.
*   **Khả năng mở rộng:** Cấu trúc lớp base `MediaForm` và `BasicMedia` giúp việc thêm một loại phương tiện mới (như Podcast hay Music) chỉ mất rất ít công sức.
*   **Bảo mật:** Sử dụng `defusedxml` để tránh tấn công XML, cấu hình Nginx với các header bảo mật, và quản lý quyền truy cập dữ liệu theo `user_id` trong mọi query.

**Kết luận:** Đây là một dự án Django mẫu mực về cách kết hợp giữa logic backend phức tạp và giao diện hiện đại (Tailwind/HTMX) mà vẫn duy trì được hiệu năng tốt trên các thiết bị cấu hình thấp (như Raspberry Pi).