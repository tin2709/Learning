Dưới đây là phân tích chi tiết về dự án **MeTube** dựa trên mã nguồn bạn đã cung cấp:

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án được xây dựng trên mô hình **Client-Server** hiện đại, tách biệt rõ ràng giữa Backend và Frontend:

*   **Backend (Python):**
    *   **Framework:** `aiohttp` - Một thư viện HTTP server/client bất đồng bộ (asynchronous) mạnh mẽ.
    *   **Real-time Communication:** `python-socketio` - Sử dụng WebSockets để cập nhật trạng thái tải xuống theo thời gian thực cho người dùng.
    *   **Download Engine:** `yt-dlp` - Trình tải video mạnh mẽ nhất hiện nay (fork từ youtube-dl).
    *   **Persistence (Lưu trữ dữ liệu):** `shelve` và `dbm` - Lưu trữ hàng đợi (queue) và lịch sử tải xuống dưới dạng file cơ sở dữ liệu đơn giản trên ổ đĩa.
    *   **Task Management:** `multiprocessing` - Mỗi tiến trình tải xuống được chạy trong một process riêng biệt để không làm treo server chính.

*   **Frontend (Angular):**
    *   **Framework:** Angular 19+ (sử dụng các tính năng mới nhất như `signal-like` viewChild, `standalone components`).
    *   **Package Manager:** `pnpm` - Giúp cài đặt dependencies nhanh và tiết kiệm dung lượng.
    *   **UI/UX:** Bootstrap 5, FontAwesome (icons), Ng-select (dropdown nâng cao).
    *   **Real-time:** `ngx-socket-io` để kết nối với Backend.

*   **DevOps & Deployment:**
    *   **Containerization:** Docker (Multi-stage build giúp giảm dung lượng image).
    *   **CI/CD:** GitHub Actions (Tự động cập nhật `yt-dlp` hàng ngày và build image đa nền tảng amd64/arm64).

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của MeTube thể hiện tư duy **"Resilient & Real-time"** (Kiên cố và Thời gian thực):

*   **Cô lập tiến trình (Process Isolation):** Thay vì dùng thread, MeTube dùng `multiprocessing.Process`. Điều này cực kỳ quan trọng vì `yt-dlp` tiêu tốn nhiều tài nguyên và đôi khi có thể gây lỗi memory. Nếu một tiến trình tải bị crash, nó không ảnh hưởng đến toàn bộ ứng dụng web.
*   **Trạng thái nhất quán (Queue Persistence):** Sử dụng `PersistentQueue` dựa trên thư viện `shelve`. Khi server khởi động lại, các link đang tải dở hoặc đang chờ sẽ được nạp lại tự động, không bị mất dữ liệu.
*   **Giao tiếp hướng sự kiện (Event-driven):** Backend không bắt Frontend phải "refresh" trang. Mọi thay đổi về tốc độ, phần trăm (%), hay lỗi đều được "emit" qua Socket.io.
*   **Cấu hình linh hoạt (Configuration-centric):** Hầu hết logic (đường dẫn, user ID, template tên file) đều được điều khiển qua biến môi trường (Environment Variables), rất phù hợp với môi trường Docker/Kubernetes.

### 3. Các kỹ thuật chính nổi bật (Key Highlighted Techniques)

*   **Xử lý lỗi cơ sở dữ liệu tự động:** Trong file `ytdl.py`, hàm `repair()` có khả năng tự phát hiện và sửa lỗi file database (`gdbm` hoặc `sqlite3`) nếu bị hỏng (corrupted) do tắt nguồn đột ngột.
*   **Template linh hoạt:** Hỗ trợ `OUTPUT_TEMPLATE` của `yt-dlp`, cho phép người dùng tự định nghĩa cấu trúc thư mục và tên file (ví dụ: đưa video vào thư mục theo tên Playlist).
*   **Hỗ trợ iOS Compatibility:** Một kỹ thuật thông minh trong `dl_formats.py` để tự động chọn hoặc chuyển đổi định dạng video sang H264/AAC để xem được trực tiếp trên iPhone/iPad.
*   **Cơ chế Watch File:** Sử dụng `watchfiles` để theo dõi file cấu hình `YTDL_OPTIONS_FILE`. Khi người dùng sửa file JSON bên ngoài, ứng dụng tự động cập nhật cấu hình mà không cần restart container.
*   **Xử lý Chapter:** Kỹ thuật tự động tách video thành từng chương (chapters) bằng `FFmpegSplitChapters` và hiển thị chúng như các file riêng lẻ trong UI.

### 4. Luồng hoạt động của dự án (Project Workflow)

1.  **Tiếp nhận yêu cầu (Add URL):**
    *   Người dùng dán URL vào UI (hoặc qua extension/bookmarklet).
    *   Frontend gửi request POST tới `/add`.
    *   Backend sử dụng `yt-dlp` với option `extract_info` (không tải) để lấy metadata (tiêu đề, định dạng, playlist hay video đơn).

2.  **Quản lý hàng đợi (Queue Management):**
    *   Nếu là Playlist, MeTube sẽ bóc tách từng video.
    *   Các item được đưa vào `PersistentQueue` (trạng thái `pending` hoặc `queued`).
    *   Hệ thống kiểm tra `MAX_CONCURRENT_DOWNLOADS`. Nếu còn slot, tiến trình tải sẽ bắt đầu.

3.  **Thực thi tải xuống (Downloading):**
    *   Một `multiprocessing.Process` được tạo ra.
    *   Backend lắng nghe `progress_hooks` từ `yt-dlp`. Các thông tin (tốc độ, dung lượng, ETA) được đẩy qua `status_queue`.
    *   Socket.io gửi thông tin này lên trình duyệt để cập nhật thanh tiến trình (Progress Bar).

4.  **Hậu xử lý (Post-processing):**
    *   Sau khi tải xong, FFmpeg (nếu cần) sẽ trộn audio/video hoặc convert định dạng.
    *   Nếu có thumbnail, nó sẽ được nhúng vào file.
    *   File được di chuyển từ thư mục `TEMP_DIR` sang `DOWNLOAD_DIR`.

5.  **Hoàn tất và Lưu trữ:**
    *   Thông tin video được chuyển từ database `queue` sang database `completed`.
    *   Người dùng có thể xem trực tiếp hoặc tải file về máy qua link Web server được cấu hình sẵn.

### Tóm tắt
MeTube không chỉ là một giao diện web đơn giản mà là một **hệ thống quản lý tải xuống chuyên nghiệp**. Nó tập trung vào sự ổn định (qua multiprocessing), tính liên tục (qua persistent queue) và trải nghiệm người dùng hiện đại (qua WebSockets). Việc đóng gói Docker cực tốt giúp nó trở thành một trong những ứng dụng tự host (self-hosted) phổ biến nhất cho việc lưu trữ video cá nhân.