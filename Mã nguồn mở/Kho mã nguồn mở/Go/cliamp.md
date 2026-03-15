Chào bạn, đây là bản phân tích chi tiết về dự án **cliamp**, một trình phát nhạc terminal (TUI) đầy tham vọng được viết bằng ngôn ngữ Go, lấy cảm hứng từ trình phát nhạc Winamp huyền thoại.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Ngôn ngữ lập trình:** **Go (Golang)**, tận dụng khả năng xử lý đồng thời (concurrency) tốt để quản lý luồng âm thanh và giao diện terminal.
*   **Giao diện người dùng (TUI):**
    *   **Bubble Tea:** Framework xây dựng TUI theo mô hình kiến trúc Elm (Model-View-Update).
    *   **Lip Gloss:** Thư viện định nghĩa style (màu sắc, viền, bố cục) cho terminal.
*   **Engine âm thanh:**
    *   **Beep:** Thư viện xử lý âm thanh cốt lõi, quản lý việc giải mã, mix nhạc và các hiệu ứng (Volume, EQ).
    *   **FFmpeg:** Được sử dụng như một engine phụ thuộc bên ngoài để giải mã các định dạng phức tạp (AAC, Opus, WMA).
    *   **yt-dlp:** Công cụ "cầu nối" để stream nhạc từ YouTube, SoundCloud và Bilibili.
*   **Tích hợp dịch vụ bên thứ ba:**
    *   **go-librespot:** Thư viện mã nguồn mở để stream trực tiếp từ Spotify (không cần App chính thức).
    *   **Subsonic/Navidrome API:** Giao tiếp với các máy chủ nhạc cá nhân.
    *   **D-Bus (Linux):** Triển khai giao thức MPRIS2 để điều khiển nhạc bằng phím media hệ thống.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Provider Pattern (Mẫu nhà cung cấp):** Hệ thống trừu tượng hóa các nguồn nhạc (Local, Spotify, YouTube, Radio) thành các `Provider`. Mỗi Provider có nhiệm vụ trả về danh sách `Track` đồng nhất, giúp UI không cần quan tâm nhạc đến từ đâu.
*   **Audio Pipeline (Đường ống âm thanh):** Nhạc được xử lý qua một chuỗi các "Streamer" lồng nhau:
    `[Nguồn nhạc (Decoder)] -> [Resampler] -> [10-band EQ] -> [Volume Control] -> [Tap (Capture data cho Visualizer)] -> [Ctrl (Play/Pause)] -> [Speaker]`.
    Tư duy này giúp các hiệu ứng âm thanh áp dụng được cho mọi nguồn nhạc, kể cả nhạc stream từ Spotify.
*   **Asynchronous Processing (Xử lý bất đồng bộ):** Các tác vụ nặng như tìm kiếm nhạc trên YouTube, tải lời nhạc (Lyrics), hoặc tải trước (Preload) bài tiếp theo được đẩy vào các Goroutine thông qua cơ chế `tea.Cmd` của Bubble Tea để không làm treo giao diện (UI lag).
*   **Local-First & Config-Driven:** Ưu tiên lưu trữ cấu hình dưới dạng file TOML cục bộ, cho phép người dùng tùy biến sâu từ theme đến các dải tần EQ.

---

### 3. Các kỹ thuật chính (Key Technical Techniques)

*   **Chained OGG Streamer:** Kỹ thuật xử lý các luồng Icecast radio (OGG/Vorbis). Khi một bài hát kết thúc trong một luồng stream liên tục, hệ thống tự động tái khởi động bộ giải mã mà không đóng kết nối HTTP, đảm bảo chuyển bài mượt mà.
*   **NavBuffer (Background Buffering):** Một cơ chế tùy chỉnh cho Navidrome/Subsonic. Nhạc được tải về trong một bộ đệm ẩn dưới nền (background). Bộ giải mã có thể bắt đầu đọc dữ liệu ngay khi những byte đầu tiên vừa về tới, thay vì phải đợi tải xong toàn bộ file.
*   **Manual Biquad Filter:** Tự triển khai các bộ lọc kỹ thuật số (second-order IIR peaking filters) để làm EQ 10 dải tần. Đây là kỹ thuật xử lý tín hiệu số (DSP) chuyên sâu để giả lập Winamp EQ.
*   **Visualizer Sampling:** Sử dụng một thành phần gọi là `Tap` trong pipeline để copy dữ liệu thô (raw samples) vào một Ring Buffer. Sau đó, UI sẽ đọc dữ liệu này để vẽ các chế độ Visualizer (Bars, Flame, Matrix...) bằng các ký tự ASCII/Braille.
*   **Mojibake Sanitization:** Kỹ thuật nhận diện và sửa lỗi mã hóa ký tự (thường gặp ở các file MP3 cũ dùng bảng mã Windows-125x thay vì UTF-8) giúp hiển thị tên bài hát chính xác cho nhiều ngôn ngữ.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Khởi động:** Ứng dụng đọc cấu hình `config.toml`, quét thư mục nhạc hoặc URL từ tham số dòng lệnh.
2.  **Phân giải (Resolution):** `resolve.Args` kiểm tra đầu vào. Nếu là link YouTube, nó gọi `yt-dlp` để lấy link stream thực tế. Nếu là file local, nó đọc metadata (ID3 tags).
3.  **Xây dựng Pipeline:**
    *   Hệ thống xác định định dạng (MP3, FLAC, hay FFmpeg-needed).
    *   Tạo luồng giải mã (`decoder`).
    *   Áp dụng EQ và Volume đã lưu.
    *   Gửi luồng cuối cùng đến thiết bị đầu ra (Speaker) thông qua thư viện `speaker`.
4.  **Vòng lặp sự kiện (Event Loop):**
    *   Người dùng nhấn phím (ví dụ: `+` tăng âm lượng) -> `Update` function thay đổi trạng thái Model -> UI vẽ lại (`View`).
    *   Song song đó, `Tap` liên tục cập nhật mẫu âm thanh cho `Visualizer`.
5.  **Chuyển bài (Gapless Transition):** Khi bài hiện tại gần hết, `gaplessStreamer` sẽ chuẩn bị sẵn bài tiếp theo (`Preload`). Khi bài cũ kết thúc, bài mới được swap vào vị trí ngay lập tức, không có khoảng lặng (zero gap).
6.  **Cập nhật ngoại vi:** Nếu chạy trên Linux, hệ thống gửi thông tin bài hát đang phát qua D-Bus để các widget hệ thống (như GNOME/KDE media player) hiển thị và điều khiển được.

### Kết luận
**cliamp** là một sự kết hợp hoàn hảo giữa hoài cổ (Winamp) và hiện đại (Go, TUI). Nó không chỉ đơn thuần gọi một trình phát nhạc bên ngoài mà tự xây dựng engine xử lý tín hiệu âm thanh riêng, tạo ra một trải nghiệm nghe nhạc chuyên nghiệp ngay trong môi trường dòng lệnh.