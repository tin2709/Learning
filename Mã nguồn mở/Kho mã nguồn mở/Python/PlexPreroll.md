Đây là bản phân tích chi tiết về dự án **Plex Prerolls** dựa trên mã nguồn bạn đã cung cấp. Dự án này là một công cụ tự động hóa mạnh mẽ dành cho những người dùng Plex muốn quản lý các video "pre-roll" (video ngắn phát trước phim) một cách thông minh và linh động.

---

### 1. Công nghệ Cốt lõi (Core Tech Stack)

*   **Ngôn ngữ lập trình:** Python 3.11 (Chạy trên nền Alpine Linux để tối ưu dung lượng).
*   **Framework Web:** **Flask** được sử dụng để xây dựng API tiếp nhận Webhook từ Plex.
*   **Thư viện Plex chuyên dụng:** `plexapi` để tương tác trực tiếp với server Plex, thay đổi cấu hình hệ thống mà không cần can thiệp thủ công.
*   **Xử lý Media:**
    *   **FFmpeg (ffmpeg-python):** Dùng để cắt, ghép, chèn overlay (poster, text) và render video pre-roll mới.
    *   **yt-dlp & youtube-search-python:** Tìm kiếm và tải trailer/soundtrack từ YouTube để làm nguyên liệu dựng video.
*   **Quản lý tiến trình:** **PM2 (Process Manager 2)** chạy trong Docker để quản lý song song hai tiến trình: API Webhook và Script chạy định kỳ (Runner).
*   **Cấu hình:** `confuse` (hỗ trợ YAML) giúp quản lý cấu hình phân cấp phức tạp.
*   **Lập lịch:** `croniter` để xử lý các biểu thức Cron.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Engineering & Architecture)

Dự án được thiết kế theo mô hình **Hybrid Automation (Tự động hóa lai)**:

*   **Kiến trúc Đa tiến trình (Decoupled Architecture):**
    *   **Runner (`run.py`):** Chạy vòng lặp vô hạn, kiểm tra lịch trình (cron) để cập nhật danh sách pre-roll định kỳ vào Plex.
    *   **API Server (`api.py`):** Lắng nghe các sự kiện Webhook (như khi có phim mới được thêm vào) để thực hiện các tác vụ thời gian thực (real-time).
    *   **PM2** đóng vai trò là "nhạc trưởng" đảm bảo nếu một tiến trình bị crash, nó sẽ tự khởi động lại mà không ảnh hưởng tiến trình kia.

*   **Quản lý Cấu hình (Configuration Management):**
    *   Sử dụng Class-based config (`ConfigSection`, `YAMLElement`) để bọc dữ liệu YAML. Điều này giúp code an toàn hơn (type-safe) và dễ bảo trì khi cấu hình phình to.

*   **Tư duy Container-first:**
    *   Mọi đường dẫn đều được tính toán để tương thích với Docker Volumes.
    *   Hỗ trợ `PUID/PGID` để xử lý vấn đề phân quyền file trên Linux/Unraid.

---

### 3. Các kỹ thuật chính nổi bật (Key Techniques)

#### A. Kỹ thuật Path Globbing (Ánh xạ đường dẫn)
Đây là giải pháp cho vấn đề kinh điển trong Docker: Đường dẫn file bên trong container khác với đường dẫn Plex nhìn thấy.
*   Hệ thống cho phép khai báo `root_path` (local) và `plex_path` (remote).
*   Script tự động quét các file bằng pattern (vd: `*.mp4`), sau đó "dịch" đường dẫn sang định dạng mà Plex hiểu được trước khi gửi API cập nhật.

#### B. Thuật toán Lập lịch Linh hoạt (Advanced Scheduling)
Không chỉ là Cron, hệ thống hỗ trợ:
*   **Wildcard Dates:** Hỗ trợ định dạng `xxxx-07-04` (mỗi năm vào ngày 4/7).
*   **Weighting (Trọng số):** Nếu một video có `weight: 2`, nó sẽ xuất hiện 2 lần trong danh sách gửi sang Plex, làm tăng gấp đôi tỉ lệ được chọn ngẫu nhiên.
*   **Disable Always:** Khả năng tạm dừng các pre-roll mặc định khi có sự kiện đặc biệt (ví dụ: Giáng sinh thì chỉ chiếu phim về Noel).

#### C. Tự động dựng Video (Dynamic Video Rendering)
Đây là tính năng cao cấp nhất:
1.  Nhận tín hiệu "Phim mới" từ Plex.
2.  Tự động lên YouTube tìm trailer và nhạc nền.
3.  Dùng FFmpeg để "dán" Poster phim, vẽ Text (Tên phim, đánh giá Critic/Audience) lên một template video (`overlay.mov`).
4.  Kết quả là một video pre-roll "nóng hổi" giới thiệu chính bộ phim vừa được thêm vào thư viện.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

**Luồng 1: Cập nhật theo lịch (Scheduled Update)**
1.  `run.py` kiểm tra file `config.yaml`.
2.  `ScheduleManager` lọc ra các video hợp lệ theo thời gian hiện tại (Tháng/Tuần/Ngày lễ).
3.  Tổng hợp tất cả đường dẫn, nhân bản theo trọng số (weight).
4.  Gọi `PlexConnector` gửi chuỗi đường dẫn (cách nhau bởi dấu `;`) vào cài đặt `cinemaTrailersPrerollID` của Plex.

**Luồng 2: Webhook phim mới (Recently Added)**
1.  Người dùng thêm phim "Inception" vào Plex.
2.  Plex gửi Webhook POST đến `/recently-added`.
3.  `WebhookProcessor` tạo một luồng (thread) mới để không làm nghẽn API.
4.  `RecentlyAddedPrerollRenderer` tải poster từ Plex, tải trailer từ YouTube.
5.  FFmpeg kết hợp tất cả lại thành video `recently-added-preroll.mp4`.
6.  Hệ thống tự xóa các video cũ (giữ lại X video gần nhất) để tiết kiệm bộ nhớ.

---

### Đánh giá chung
Đây là một dự án có tính **hoàn thiện kỹ thuật cao**, giải quyết triệt để bài toán cá nhân hóa trải nghiệm xem phim gia đình. Việc sử dụng kết hợp giữa Webhook, xử lý ảnh/video động và quản lý tiến trình bằng PM2 cho thấy tác giả có tư duy hệ thống rất tốt.

**Lưu ý cho bạn:** Nếu muốn tính năng tự động render chạy tốt, bạn **bắt buộc** phải có file `yt_dlp_cookies.txt` vì YouTube hiện tại chặn rất gắt các công cụ tải trailer tự động.