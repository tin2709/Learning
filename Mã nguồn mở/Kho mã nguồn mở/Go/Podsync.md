Dựa trên mã nguồn của dự án **Podsync** (phiên bản cập nhật tháng 3/2026), dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)
Podsync được thiết kế như một công cụ trung gian (middleware) chuyển đổi nội dung video/audio từ các nền tảng lớn sang chuẩn RSS Podcast.

*   **Ngôn ngữ:** **Go (Golang) 1.25**, tận dụng tối đa khả năng xử lý song song (concurrency) để tải và xử lý nhiều feed cùng lúc.
*   **Xử lý Media:** Dựa trên hai công cụ dòng lệnh mạnh mẽ là `yt-dlp` (tải nội dung) và `ffmpeg` (chuyển đổi định dạng, nén audio).
*   **Lưu trữ Metadata:** **BadgerDB** - một thư viện key-value store thuần Go, nhúng trực tiếp vào ứng dụng để lưu trạng thái feed, ID episode và lịch sử tải, tránh việc phải cài đặt các hệ quản trị CSDL phức tạp.
*   **Lưu trữ Blob (Media):** Hỗ trợ song song **Local Filesystem** và **Amazon S3 (S3-compatible)**.
*   **Giao thức:** Phục vụ nội dung qua HTTP/HTTPS và định dạng RSS XML chuẩn iTunes.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Hệ thống đi theo hướng **Modular & Interface-driven** (Lập trình dựa trên giao diện), giúp tách biệt hoàn toàn giữa logic nghiệp vụ và hạ tầng.

*   **Tính trừu tượng hóa (Abstraction):** Tầng lưu trữ được định nghĩa qua interface `Storage`. Điều này cho phép ứng dụng hoán đổi giữa việc lưu file trên ổ cứng local hoặc đẩy lên đám mây (S3) mà không cần thay đổi logic của trình cập nhật (Updater).
*   **Cơ chế luân chuyển Key (Key Rotation):** Để đối phó với giới hạn (rate limiting) của YouTube/Vimeo API, dự án triển khai `KeyProvider`. Nó cho phép nạp nhiều API key và tự động xoay vòng khi thực hiện yêu cầu.
*   **Tách biệt trạng thái và nội dung:** Trạng thái hệ thống nằm trong BadgerDB, trong khi tệp XML và Media (nặng) được quản lý bởi tầng `fs`. Điều này tối ưu hóa hiệu suất đọc ghi.
*   **Headless & Server mode:** Kiến trúc cho phép chạy như một server bền bỉ (với cron scheduler) hoặc chạy một lần rồi thoát (headless mode), phù hợp cho cả Docker và các script tự động hóa.

### 3. Kỹ thuật lập trình (Programming Techniques)
*   **Graceful Shutdown:** Sử dụng `errgroup` và `context.WithCancel` kết hợp với tín hiệu hệ điều hành (`SIGINT`, `SIGTERM`). Khi dừng ứng dụng, hệ thống sẽ đợi các tiến trình tải dở kết thúc hoặc đóng DB an toàn trước khi thoát hẳn.
*   **Template Engine:** Implement tính năng `filename_template` cho phép người dùng tùy biến tên file tải về bằng các token như `{{title}}`, `{{id}}`, `{{pub_date}}`.
*   **Multi-error handling:** Sử dụng gói `go-multierror` để thu thập lỗi từ nhiều feed khác nhau. Nếu một feed bị lỗi (do API key hết hạn hoặc video bị xóa), nó sẽ không làm treo toàn bộ tiến trình cập nhật của các feed khác.
*   **Middleware Bảo mật:** Trong `server.go`, dự án sử dụng một `ServeMux` tùy chỉnh thay vì `DefaultServeMux` của Go để tránh lộ các endpoint debug (`expvar`) ra ngoài internet, một lỗi bảo mật phổ biến trong các ứng dụng Go.
*   **Self-healing:** Tự động cập nhật `yt-dlp` định kỳ để đảm bảo tính tương thích khi các nền tảng video thay đổi thuật toán.

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng Cập nhật (Update Loop):
1.  **Scheduler (Cron):** Kích hoạt việc cập nhật theo chu kỳ (ví dụ: mỗi 12 giờ).
2.  **API Query:** `Builder` gửi yêu cầu đến YouTube/Vimeo để lấy danh sách video mới nhất.
3.  **Filter:** `Matcher` kiểm tra các tiêu chí (độ dài video, từ khóa trong tiêu đề, ngày đăng) để quyết định có tải hay không.
4.  **Download & Transcode:** `yt-dlp` tải video, `ffmpeg` chuyển đổi sang MP3 (nếu cấu hình audio).
5.  **Storage:** Lưu tệp vào đích (Local/S3) và cập nhật Metadata vào BadgerDB.
6.  **Generator:** Tạo lại tệp `.xml` cho feed đó và tệp `.opml` tổng hợp.

#### B. Luồng Phục vụ (Serving Loop):
1.  **Request:** Ứng dụng Podcast (như Apple Podcasts, Overcast) gửi yêu cầu lấy file `.xml`.
2.  **Static Serving:** Web server trả về tệp XML tĩnh đã được generate sẵn (tốc độ cực nhanh).
3.  **Enclosure Redirect:** Khi người dùng nhấn Play, ứng dụng Podcast sẽ theo đường dẫn `enclosure` trong XML để tải/stream file media từ server hoặc S3.

### Tổng kết
Podsync là một ví dụ điển hình về việc xây dựng **"Cloud-native Tool"** bằng Go: nhỏ gọn, hiệu suất cao, cấu hình đơn giản qua TOML/Env và có khả năng chạy ổn định trên các hệ thống tài nguyên thấp như NAS (QNAP, Synology) thông qua Docker. Dự án tập trung vào tính ổn định của luồng dữ liệu (Data pipeline) hơn là sự phức tạp của giao diện người dùng.