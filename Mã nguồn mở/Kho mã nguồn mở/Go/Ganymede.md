Dưới đây là bản phân tích chi tiết về kho lưu trữ **Ganymede** - một nền tảng lưu trữ (archiving) Twitch VOD và Live Stream dựa trên các tệp tin bạn đã cung cấp.

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án sử dụng một "stack" công nghệ hiện đại, chú trọng vào hiệu suất và khả năng mở rộng:

*   **Backend (Golang):** Ngôn ngữ chủ đạo cho logic xử lý server (`api`) và các tác vụ chạy ngầm (`worker`). Go được chọn vì khả năng xử lý đồng thời (concurrency) cực tốt, phù hợp cho việc tải nhiều luồng video cùng lúc.
*   **Frontend (Next.js & TypeScript):** Sử dụng React framework (Next.js) để xây dựng giao diện người dùng (UI) hiện đại, mượt mà và tối ưu SEO/Performance.
*   **Database (PostgreSQL & Ent ORM):**
    *   **Postgres:** Cơ sở dữ liệu quan hệ mạnh mẽ.
    *   **Ent:** Một Entity Framework cho Go, giúp quản lý Schema và truy vấn database một cách "Type-safe" (an toàn về kiểu dữ liệu).
*   **Hệ thống Task Queue (River):** Sử dụng thư viện `riverqueue` để quản lý các tác vụ xếp hàng như: tải chat, render chat, tải video, tạo thumbnail.
*   **Công cụ xử lý Video/Media:**
    *   **FFmpeg:** "Xương sống" để chuyển đổi, cắt ghép và xử lý video.
    *   **yt-dlp:** Công cụ tải video từ các nền tảng (đã được tùy chỉnh bằng patch riêng cho Twitch).
    *   **TwitchDownloaderCLI:** Công cụ chuyên dụng để tải dữ liệu từ Twitch.
*   **Containerization (Docker & Supervisord):** Sử dụng Docker để đóng gói. Đặc biệt dùng **Supervisord** để quản lý đồng thời 3 tiến trình (`api`, `worker`, `frontend`) bên trong cùng một container.

---

### 2. Kỹ thuật & Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của Ganymede thể hiện tư duy phân tách trách nhiệm (Separation of Concerns) rõ rệt:

*   **Kiến trúc Server-Worker:**
    *   **Server (API):** Chịu trách nhiệm nhận yêu cầu từ người dùng, quản lý xác thực (OAuth/SSO), và cung cấp dữ liệu cho giao diện.
    *   **Worker:** Chuyên xử lý các tác vụ "nặng" (tốn CPU/Băng thông). Điều này giúp ứng dụng không bị treo khi đang tải các video dung lượng lớn.
*   **Tư duy "File-First" (Lưu trữ bền vững):** Ganymede không chỉ lưu dữ liệu vào DB mà còn tổ chức tệp tin theo cấu trúc thư mục rõ ràng. Mục tiêu là nếu Ganymede ngừng hoạt động, người dùng vẫn có thể xem lại video và chat bằng các trình phát phổ thông.
*   **Multi-stage Docker Build:** Dockerfile được chia làm nhiều giai đoạn (build api, build frontend, build tools) giúp giảm kích thước hình ảnh (image) cuối cùng và bảo mật mã nguồn.
*   **Tích hợp sâu với GQL & IRC:** Ứng dụng tương tác trực tiếp với Twitch GQL API để lấy metadata và sử dụng giao thức IRC để bắt (capture) chat trong thời gian thực khi stream đang diễn ra.

---

### 3. Các kỹ thuật chính nổi bật (Key Technical Highlights)

1.  **Real-time Chat Playback:** Kỹ thuật đồng bộ hóa dữ liệu chat (đã lưu) với mốc thời gian của video đang phát, tạo trải nghiệm như đang xem trực tiếp trên Twitch.
2.  **Custom yt-dlp Patching:** Dự án bao gồm một tệp `.patch` cho `yt-dlp` để xử lý các thay đổi đặc thù của Twitch (như hệ thống Playback Access Token), giúp vượt qua các rào cản kỹ thuật của nền tảng.
3.  **Hệ thống Quản lý "Watched Channels":** Sử dụng logic định kỳ (cron job/watchdog) để kiểm tra trạng thái live của các kênh được theo dõi và tự động kích hoạt tiến trình lưu trữ.
4.  **SSO/OIDC Integration:** Hỗ trợ đăng nhập một lần (Single Sign-On) qua OpenID Connect, cho phép tích hợp vào các hệ thống quản lý định danh có sẵn (như Keycloak, Authelia).
5.  **Tối ưu hóa tài nguyên hệ thống:** Cho phép cấu hình số lượng tác vụ song song tối đa (`MAX_VIDEO_DOWNLOAD_EXECUTIONS`, v.v.) thông qua biến môi trường để phù hợp với cấu hình phần cứng của người dùng.

---

### 4. Tóm tắt luồng hoạt động của dự án (Project Flow)

Luồng hoạt động chính có thể tóm tắt qua các bước sau:

1.  **Thiết lập:** Người dùng cấu hình thông qua `docker-compose`, cung cấp Twitch Client ID/Secret.
2.  **Theo dõi (Monitoring):** API Server gửi yêu cầu đến Twitch để theo dõi các kênh trong danh sách "Watched".
3.  **Kích hoạt tác vụ (Triggering):** Khi phát hiện kênh đang Live hoặc có VOD mới, một "Job" sẽ được đẩy vào **River Queue**.
4.  **Thực thi (Execution):**
    *   **Worker** nhận Job.
    *   Sử dụng `yt-dlp` hoặc `TwitchDownloader` để tải video gốc.
    *   Đồng thời tải hoặc capture chat stream.
    *   Sử dụng `FFmpeg` để xử lý hậu kỳ (nếu cần) và tạo ảnh thumbnail (sprite).
5.  **Lưu trữ & Phục vụ:**
    *   Video và metadata được lưu vào thư mục `/data/videos`.
    *   Thông tin được cập nhật vào database thông qua **Ent ORM**.
6.  **Người dùng cuối:** Truy cập giao diện web (Next.js), tìm kiếm video và xem lại với đầy đủ chat, chương (chapters) và tính năng phát lại mượt mà.

**Tổng kết:** Ganymede là một dự án có độ hoàn thiện kỹ thuật cao, kết hợp giữa sức mạnh xử lý hệ thống của **Go** và sự linh hoạt của **Next.js**, hướng tới mục tiêu lưu trữ dữ liệu truyền thông một cách chuyên nghiệp và bền vững.