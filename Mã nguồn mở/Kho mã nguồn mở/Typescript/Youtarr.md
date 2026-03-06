Dựa trên cấu trúc thư mục và nội dung mã nguồn của dự án **Youtarr**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc, kỹ thuật lập trình và luồng hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Tech Stack)
Youtarr là một ứng dụng Full-stack hiện đại, được đóng gói hoàn toàn bằng Docker.
*   **Backend:** Node.js với framework **Express**. Sử dụng **Sequelize ORM** để tương tác với cơ sở dữ liệu (MariaDB/MySQL).
*   **Frontend:** **React** kết hợp với **TypeScript**. Sử dụng **Vite** làm công cụ build (thay thế cho CRA cũ), **Material UI (MUI)** cho giao diện, và **Axios** để gọi API.
*   **Cơ sở dữ liệu:** **MariaDB 10.3** (được cấu hình mặc định trong Docker Compose).
*   **Công cụ thực thi chính:**
    *   **yt-dlp:** "Trái tim" của hệ thống, dùng để tải video và trích xuất dữ liệu từ YouTube.
    *   **FFmpeg:** Xử lý hậu kỳ video/âm thanh.
    *   **Apprise:** Gửi thông báo đa nền tảng (Discord, Telegram, Slack...).
    *   **SponsorBlock:** Tự động loại bỏ quảng cáo tích hợp trong video.
*   **Real-time:** Sử dụng **WebSockets (ws)** để cập nhật trạng thái tải xuống và thông báo từ server lên giao diện người dùng ngay lập tức.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Dự án được thiết kế theo hướng **"Docker-First"** và **"Media Server Friendly"**:
*   **Kiến trúc Client-Server tách biệt:** Backend cung cấp RESTful API và quản lý các công việc chạy ngầm (cron jobs), trong khi Frontend là một SPA (Single Page Application) độc lập.
*   **Cấu trúc Module-based:** Backend được chia thành các module chuyên biệt (`server/modules`) như `downloadModule`, `plexModule`, `ytdlpModule`, `notificationModule`... giúp dễ dàng bảo trì và mở rộng.
*   **Quản lý trạng thái (State Management):** Sử dụng các Custom Hooks (như `useConfig`, `useChannelList`) ở Frontend để đóng gói logic xử lý dữ liệu, giúp các component giao diện gọn gàng hơn.
*   **Khả năng di động (Portability):** Toàn bộ cấu hình và dữ liệu được lưu trữ trong các volumes (`/app/config`, `/app/data`, `/app/database`), cho phép người dùng nâng cấp image Docker mà không mất dữ liệu.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)
*   **Quản lý Di trú dữ liệu (Database Migrations):** Sử dụng `umzug` (thư viện đi kèm Sequelize) để quản lý phiên bản database. Mỗi thay đổi cấu trúc bảng đều có file migration riêng (`migrations/`), đảm bảo tính nhất quán giữa các môi trường.
*   **Hệ thống xử lý lỗi đa tầng:**
    *   **Backend:** Middleware xử lý lỗi tập trung và kiểm tra sức khỏe DB (`databaseHealthModule`).
    *   **Frontend:** Sử dụng **Error Boundaries** để chặn lỗi render và **DatabaseErrorOverlay** để hiển thị thông báo khi mất kết nối DB.
*   **Kiểm thử (Testing Strategy):**
    *   **Unit/Integration Test:** Sử dụng **Jest** cho cả Backend và Frontend.
    *   **UI Testing:** Sử dụng **Storybook** để phát triển component trong môi trường cô lập và **MSW (Mock Service Worker)** để giả lập API.
*   **Bảo mật:**
    *   Xác thực người dùng qua JWT/Session.
    *   Hỗ trợ **API Keys** cho các tích hợp bên ngoài.
    *   Chế độ **Headless setup** qua biến môi trường (`AUTH_PRESET_USERNAME/PASSWORD`).

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng Đăng ký Kênh (Channel Subscription)
1.  Người dùng nhập URL/Handle kênh vào giao diện.
2.  Backend sử dụng `ytdlpModule` để lấy metadata (tên uploader, thumbnail, mô tả).
3.  Thông tin kênh được lưu vào DB.
4.  Hệ thống quét các tab (Videos, Shorts, Streams) dựa trên cấu hình người dùng để tìm video mới.

#### B. Luồng Tải xuống tự động (Auto-Download Workflow)
1.  **Cron Job:** Một tác vụ lập lịch chạy định kỳ quét các kênh đã đăng ký.
2.  **Lọc dữ liệu:** Video được kiểm tra qua các bộ lọc: Độ phân giải, thời lượng, từ khóa (Regex), định dạng âm thanh.
3.  **Hàng đợi (Queue):** Tạo một `Job` tải xuống trong database.
4.  **Thực thi:** `ytDlpRunner` gọi `yt-dlp` để tải file về thư mục tạm.
5.  **Hậu xử lý:**
    *   FFmpeg gộp file video/audio.
    *   Nhúng metadata vào file MP4.
    *   Tạo file `.nfo` và tải ảnh poster cho các Media Server (Plex/Kodi).
6.  **Hoàn tất:** Di chuyển file vào thư mục đích, cập nhật trạng thái video trong DB và gửi thông báo qua Apprise.

#### C. Luồng Tích hợp Plex
1.  Khi video tải xong, `plexModule` được kích hoạt.
2.  Backend sử dụng Plex Token để gọi API của Plex Server.
3.  Yêu cầu Plex cập nhật thư viện (Library Scan) chỉ cho thư mục cụ thể chứa video mới, giúp cập nhật nhanh chóng mà không cần scan toàn bộ ổ đĩa.

### 5. Điểm nổi bật trong mã nguồn
*   **Script khởi chạy thông minh:** Các file `.sh` (như `start.sh`, `scripts/_start_template.sh`) có khả năng tự phát hiện kiến trúc phần cứng (ARM vs x86) để điều chỉnh file Docker Compose phù hợp (ví dụ: sửa lỗi mount volume trên ARM).
*   **Tính tùy biến cao:** Dự án hỗ trợ cả DB nội bộ lẫn DB bên ngoài, cho phép người dùng linh hoạt trong việc triển khai trên NAS (Synology, Unraid) hoặc Cloud.

Tóm lại, **Youtarr** là một dự án có tính kỹ thuật cao, chú trọng vào trải nghiệm người dùng tự lưu trữ (self-hosted) và khả năng tích hợp sâu vào hệ sinh thái giải trí gia đình.