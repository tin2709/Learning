Dựa trên nội dung mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về **OmniGet** - một dự án phần mềm nguồn mở hiện đại, được xây dựng với tư duy ưu tiên hiệu suất và trải nghiệm người dùng học tập.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án sử dụng mô hình **Modern Desktop Hybrid**:
*   **Backend: Rust & Tauri 2.x:** Sử dụng Rust để đảm bảo an toàn bộ nhớ và hiệu suất tối đa. Tauri 2.0 đóng vai trò là "cầu nối" (bridge) giữa hệ điều hành và giao diện web, giúp ứng dụng nhẹ hơn nhiều so với Electron (do dùng Webview bản địa).
*   **Frontend: Svelte 5 (Runes):** Một bước đi tiên phong khi sử dụng **Svelte 5** với cơ chế **Runes** (`$state`, `$derived`, `$effect`). Đây là kỹ thuật reactivity mới nhất, giúp quản lý trạng thái phức tạp một cách tường minh và hiệu quả hơn.
*   **Download Engine:** Tích hợp `yt-dlp` (thông qua Rust) để xử lý hơn 1000 trang web video. Ngoài ra còn có các bộ nạp tùy chỉnh cho HLS (`.m3u8`) và DASH (`.mpd`).
*   **Persistence:** Sử dụng **SQLite (via sqlx)** để lưu trữ cục bộ toàn bộ dữ liệu ghi chú, thẻ nhớ (flashcards), và tiến độ học tập.
*   **Browser Extension:** Viết bằng Vanilla JS/Manifest V3, giao tiếp với app thông qua **Native Messaging API**.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của OmniGet được thiết kế theo hướng **"Plug-and-Play"** và **"Local-First"**:
*   **Trait-based Platform System:** Trong `src-tauri/platforms/`, dự án sử dụng `traits.rs` để định nghĩa một interface chung (`PlatformDownloader`). Mỗi nền tảng (Hotmart, Udemy, YouTube...) là một module riêng biệt thực thi trait này. Điều này giúp việc mở rộng thêm nền tảng mới cực kỳ dễ dàng mà không làm ảnh hưởng đến lõi hệ thống.
*   **Decoupled Frontend:** Giao diện được thiết kế theo hướng **Domain-driven**. Các thành phần UI được chia theo miền chức năng (như `omnibox/`, `hotmart/`, `mascot/`), giúp mã nguồn dễ bảo trì khi quy mô dự án tăng lên.
*   **Offline-First Strategy:** Tư duy cốt lõi là "Download once, own forever". Toàn bộ dữ liệu (video, PDF, ghi chú) đều nằm trên ổ đĩa người dùng. App đóng vai trò là một lớp quản lý và tương tác (Player/Reader) trên nền dữ liệu đó.
*   **CSS Architecture:** Không dùng Tailwind hay CSS-in-JS. Dự án sử dụng **CSS Variables (Tokens)** thuần túy. Điều này giúp việc thay đổi theme (14 themes có sẵn) trở nên cực kỳ nhanh chóng và không gây overhead cho runtime.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Svelte 5 Runes:** Thay vì dùng `store` truyền thống, dự án dùng `$state` cho các đối trạng thái sâu (như danh sách download) và `$derived` để tự động tính toán các giá trị phụ thuộc (như tốc độ trung bình, ETA).
*   **Optimistic UI:** Khi người dùng nhấn Pause/Resume, giao diện cập nhật trạng thái ngay lập tức (Optimistic), sau đó mới gửi lệnh xuống backend Rust. Nếu lệnh thất bại, UI mới rollback. Điều này tạo cảm giác ứng dụng phản hồi tức thì.
*   **Sentinel Values cho Download Phase:** Backend sử dụng các giá trị sentinel (như -1.0 cho "Starting", -2.0 cho "Connecting") để truyền trạng thái từ Rust sang Svelte mà không cần cấu trúc dữ liệu phức tạp qua IPC.
*   **Type-safe i18n:** Dự án có một script (`generate-i18n-keys.js`) để tự động tạo ra các TypeScript types từ file JSON ngôn ngữ, đảm bảo không bao giờ gọi sai khóa dịch trong code.
*   **HLS Grouping:** Thuật toán trong `hls-grouping.js` thông minh trong việc nhận diện các biến thể chất lượng khác nhau của cùng một video để gom nhóm chúng lại, tránh việc hiển thị trùng lặp cho người dùng.

### 4. Luồng hoạt động hệ thống (System Workflow)

Có hai luồng chính (The Loops):

#### A. Luồng Tải xuống (The Download Loop):
1.  **Capture:** Người dùng paste URL vào Omnibox hoặc nhấn `Alt+O` trên Browser Extension.
2.  **Detection:** Module `detect.js` nhận diện nền tảng và loại nội dung (Video, Course, Playlist).
3.  **Authentication:** Nếu cần (như Udemy/Hotmart), app sử dụng cookies từ trình duyệt (truyền qua Native Messaging) để xác thực.
4.  **Enqueued:** Nhiệm vụ được đưa vào `queue.rs` trong Rust.
5.  **Processing:** Rust gọi `yt-dlp` hoặc `hls_downloader` để tải các phân đoạn (segments). Nếu là HLS, nó sẽ tự động dùng FFmpeg để merge thành file MP4 hoàn chỉnh.

#### B. Luồng Học tập (The Study Loop):
1.  **Library Sync:** Ứng dụng quét các thư mục đã chọn, trích xuất metadata (ISBN cho sách, Lesson ID cho khóa học).
2.  **Indexing:** Dữ liệu được lưu vào SQLite.
3.  **Interaction:** Người dùng mở video hoặc tài liệu.
    *   **Video:** Trình phát video tùy chỉnh cho phép tạo ghi chú gắn kèm timestamp. Khi nhấn vào ghi chú, video nhảy đến giây đó.
    *   **Reader:** Trình đọc PDF/EPUB cho phép highlight và lưu lại vào database dưới dạng Markdown.
4.  **Spaced Repetition:** Các ghi chú có thể được chuyển thành Flashcards theo thuật toán SM2 (giống Anki) để ôn tập định kỳ.

### Tổng kết
OmniGet không chỉ là một công cụ tải về (downloader), mà là một **Hệ sinh thái học tập cục bộ**. Sự kết hợp giữa **Rust** (hiệu năng, xử lý file) và **Svelte 5** (giao diện phản hồi cực nhanh) tạo ra một công cụ chuyên nghiệp, vượt xa các giải pháp tải video thông thường.