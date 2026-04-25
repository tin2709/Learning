Dựa trên toàn bộ mã nguồn bạn cung cấp, đây là phân tích chi tiết về dự án **AdPlay** - một hệ thống Digital Signage (biển hiệu kỹ thuật số) vận hành nội bộ (local-first).

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án sử dụng một Stack hiện đại, tập trung vào hiệu suất xử lý media và tính đơn giản trong triển khai:

*   **Frontend:**
    *   **Angular (v21):** Sử dụng các tính năng mới nhất như **Signals** để quản lý trạng thái (state management), Standalone Components, và `provideHttpClient`.
    *   **Tailwind CSS:** Xử lý giao diện (UI) với khả năng tùy biến Dark/Light mode linh hoạt.
    *   **Hls.js:** Thư viện giúp trình phát (Player) có thể chạy các luồng video Adaptive Streaming (HLS) ngay cả trên các trình duyệt không hỗ trợ bản xứ.
    *   **RxJS:** Quản lý các luồng dữ liệu bất đồng bộ (API calls, polling).
*   **Backend:**
    *   **Node.js & Express:** Framework phía máy chủ xử lý API và Streaming.
    *   **TypeScript:** Đảm bảo an toàn kiểu dữ liệu cho toàn bộ hệ thống backend.
    *   **Lowdb (JSON Database):** Sử dụng tệp JSON để lưu trữ dữ liệu. Đây là lựa chọn tối ưu cho hệ thống chạy local, không cần cài đặt SQL phức tạp.
    *   **FFmpeg (ffmpeg-static & ffprobe-static):** "Trái tim" của hệ thống xử lý media, dùng để tối ưu hóa video, tạo ảnh poster và phân tách luồng HLS.
    *   **Multer:** Xử lý việc tải lên (upload) tệp tin.
    *   **JWT (JSON Web Token) & Bcrypt:** Xử lý xác thực và bảo mật mật khẩu.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của AdPlay được thiết kế theo hướng **"Plug-and-Play" (Cắm là chạy)** và **"Local-first"**:

*   **Zero-Config Startup:** Sử dụng tệp `launch-adplay.cjs` để tự động hóa mọi thứ: kiểm tra môi trường, cài đặt dependencies, build frontend/backend, tạo file `.env` và mở trình duyệt. Người dùng phổ thông chỉ cần chạy 1 file `.bat` hoặc `.sh`.
*   **Separation of Concerns (SoC):**
    *   **Service Layer:** Tách biệt logic nghiệp vụ (Business Logic) ra khỏi Route. Ví dụ: `video.service.ts` xử lý logic lưu trữ, trong khi `media.service.ts` xử lý FFmpeg.
    *   **Repository Pattern:** `db.ts` đóng vai trò là lớp trung gian duy nhất tương tác với tệp JSON, giúp dễ dàng thay thế bằng SQL (Postgres/SQLite) sau này nếu cần.
*   **Feature-based Structure (Frontend):** Chia mã nguồn theo tính năng (`dashboard`, `player`, `auth`) thay vì chia theo loại tệp (component, service), giúp dễ quản lý và mở rộng.

---

### 3. Các kỹ thuật chính (Key Technical Implementation)

Dự án sở hữu một số kỹ thuật xử lý chuyên sâu:

*   **Resumable Chunked Upload (Tải lên có thể tạm dừng):**
    *   Frontend chia nhỏ tệp video thành các "chunks" (mặc định 8MB).
    *   Nếu mạng lỗi, hệ thống sẽ chỉ tải lại phần còn thiếu dựa trên `fileKey` (fingerprint của tệp).
*   **Media Pipeline (Luồng xử lý truyền thông):**
    *   **Optimization:** Sau khi upload, backend chạy ngầm FFmpeg để chuyển đổi video về cấu hình chuẩn (H.264 Baseline, 480p/1080p) để đảm bảo các TV đời cũ cũng có thể phát được.
    *   **HLS Generation:** Tự động tạo tệp `.m3u8` và các đoạn video nhỏ (`.ts`) để hỗ trợ streaming mượt mà, giảm tải cho bộ nhớ trình duyệt.
*   **Hybrid Player Strategy:**
    *   Player ưu tiên phát qua **HLS** để tối ưu hiệu suất. Nếu thất bại, nó tự động chuyển sang **Direct MP4 Streaming** (HTTP Range Requests).
    *   **Cache API:** Sử dụng trình duyệt Cache để lưu trữ các video nhỏ, giúp Player vẫn có thể chạy ổn định khi mất kết nối mạng tạm thời.
*   **Pairing & Heartbeat:**
    *   Sử dụng Token đặc biệt gắn với mỗi Profile để "ghép đôi" TV với server.
    *   Cơ chế Heartbeat (30 giây/lần) giúp Admin biết chính xác màn hình nào đang online/offline.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

#### A. Luồng Quản trị (Admin Workflow):
1.  **Xác thực:** Admin đăng nhập qua JWT.
2.  **Tải lên:** Video/Ảnh được tải lên theo từng phần (chunks).
3.  **Xử lý:** Backend nhận đủ chunks -> Ghép file -> Chạy FFmpeg tối ưu video + Tạo ảnh Poster + Tạo playlist HLS.
4.  **Cấu hình:** Admin tạo **Profile** (Màn hình), chọn các nội dung từ thư viện để tạo thành một Playlist.

#### B. Luồng Trình phát (Player Workflow):
1.  **Kết nối:** TV truy cập đường dẫn `/player/:slug?token=...`. Token này sau đó được lưu vào `localStorage` và ẩn khỏi thanh địa chỉ để bảo mật.
2.  **Đồng bộ:** Player gọi API lấy thông tin Profile và danh sách Playlist.
3.  **Phát nội dung:**
    *   Vòng lặp (Loop) bắt đầu.
    *   Nếu là video: Ưu tiên HLS -> MP4.
    *   Nếu là ảnh: Hiển thị trong 10 giây (mặc định).
4.  **Giám sát:** Player gửi tín hiệu "Heartbeat" về server định kỳ để cập nhật trạng thái "Last Seen".
5.  **Cập nhật nóng:** Player thực hiện "Manual Sync" mỗi phút. Nếu Admin thay đổi Playlist, Player sẽ nhận diện sự thay đổi và tự động cập nhật nội dung mới mà không cần load lại trang.

**Kết luận:** AdPlay là một hệ thống hoàn chỉnh, được viết rất chỉn chu, kết hợp giữa sự đơn giản của NodeJS và sức mạnh xử lý video của FFmpeg, cực kỳ phù hợp cho các giải pháp biển hiệu quảng cáo nội bộ chất lượng cao.