Dưới đây là bản phân tích chi tiết về dự án **LocalStream – Private Lecture Transcriber** (còn gọi là **MP3 Grabber**), một hệ thống chuyển đổi bài giảng video/audio thành văn bản tập trung vào tính riêng tư và chạy hoàn toàn trên máy cục bộ.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án là sự kết hợp giữa hệ sinh thái Python (AI) và Node.js (Hệ thống/Mạng):

*   **Transcription Engine (AI):** **faster-whisper** (một bản implementation tối ưu của OpenAI Whisper). Công nghệ này cho phép nhận dạng giọng nói nhanh gấp 4 lần phiên bản gốc và tiêu tốn ít RAM hơn.
*   **Audio/Stream Processing:**
    *   **yt-dlp:** Thư viện mạnh mẽ để tải các luồng HLS (m3u8) và DASH (mpd) từ các nền tảng giáo dục như Panopto, Kaltura.
    *   **ffmpeg:** Xử lý hậu kỳ, tách luồng âm thanh từ video.
*   **Backend & Relay:** **Node.js (Express & ws)**. Đóng vai trò làm máy chủ trung gian (Relay Server) để nhận dữ liệu từ trình duyệt và điều phối các tiến trình Python.
*   **Frontend & Capture:** **Chrome Extension (Manifest V3)**. Sử dụng API `webRequest` để "ngửi" (sniffing) lưu lượng mạng nhằm tìm link stream ẩn.
*   **Acceleration:** Hỗ trợ **NVIDIA CUDA** để tăng tốc xử lý qua GPU, tự động chuyển về CPU nếu không có phần cứng tương thích.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án thể hiện tư duy thiết kế thực dụng, giải quyết bài toán "vượt rào" bảo mật của các trang web trường học:

*   **Passive Network Sniffing (Thay thế DOM Scraping):** Thay vì cố gắng đọc mã nguồn trang web (vốn dễ thay đổi giao diện), ứng dụng đứng ở tầng mạng để bắt các tệp manifest (`.m3u8`, `.mpd`). Đây là cách tiếp cận bền vững (robust) hơn để lấy link bài giảng từ Canvas hay Panopto.
*   **Kiến trúc Relay (Relay Pattern):** Do trình duyệt không thể trực tiếp chạy mã Python nặng nề, hệ thống sử dụng một WebSocket Relay. Trình duyệt gửi URL + Session Cookies về Node.js; Node.js sau đó gọi `yt-dlp` (kèm cookie để giả lập người dùng đã đăng nhập) và chuyển tiếp file cho Python.
*   **Tách biệt logic (Decoupling):**
    *   `start.js`: Quản lý cài đặt, môi trường và menu người dùng.
    *   `relay.js`: Xử lý giao tiếp thời gian thực và quản lý hàng đợi công việc.
    *   `transcribe.py`: Chỉ tập trung vào tác vụ AI (SST - Speech to Text).
*   **Local-First & Privacy:** Kiến trúc loại bỏ hoàn toàn Cloud API (như OpenAI API hay Google Speech). Dữ liệu không bao giờ rời khỏi máy tính của sinh viên, đảm bảo tính riêng tư tuyệt đối.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Cookie Extraction & Handover:** Kỹ thuật trích xuất cookie từ phiên làm việc hiện tại của trình duyệt và chuyển sang định dạng `Netscape cookie` để `yt-dlp` có thể tải được các video nằm sau lớp tường lửa đăng nhập của trường học.
*   **Intelligent Stream Filtering:** Trong một trang web có thể có nhiều tệp stream (các độ phân giải khác nhau). Mã nguồn (`test_extension_filter.js`) có logic phân loại ưu tiên: tệp `master.m3u8` có điểm cao nhất (100), sau đó là các playlist chất lượng thấp hơn.
*   **Inter-process Communication (IPC):** Node.js sử dụng `child_process` để thực thi script Python. Nó lắng nghe luồng `stdout` của Python để cập nhật phần trăm tiến độ về giao diện `viewer.html` qua WebSocket.
*   **Crash-safe Transcription:** Lưu kết quả trung gian vào các tệp "sidecar" (.json). Nếu quá trình transcription bị sập giữa chừng, hệ thống có thể khôi phục hoặc giữ lại phần đã làm xong.
*   **Debouncing:** Tránh việc bắt link trùng lặp khi người dùng chuyển đổi chất lượng video trên trình duyệt bằng cách sử dụng cơ chế chờ (delay) và kiểm tra ID duy nhất của stream.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Hệ thống vận hành qua 2 luồng chính:

#### Luồng A: File nội bộ (File Mode)
1.  Người dùng bỏ file MP3/MP4 vào thư mục `media/`.
2.  `start.js` quét thư mục và hiện danh sách.
3.  Python khởi tạo model Whisper (Tiny/Base/Medium tùy cấu hình).
4.  Kết quả xuất ra thư mục `transcriptions/` dưới dạng file `.txt` có timestamp.

#### Luồng B: Trình duyệt (Extension Mode)
1.  Người dùng mở trang bài giảng trên Chrome.
2.  **Extension** phát hiện yêu cầu mạng chứa `.m3u8`.
3.  **Extension** lấy URL này cùng với Cookies của trang web gửi qua WebSocket về **Relay Server** (Node.js).
4.  **Relay Server** dùng `yt-dlp` để tải stream về thư mục `uploads/`.
5.  **Relay Server** gọi Python để chuyển âm thanh thành văn bản.
6.  Tiến độ được cập nhật trực tiếp lên trang `http://localhost:8787` để sinh viên theo dõi.

---

### Tổng kết
Dự án là một công cụ **Data Pipeline** mini hoàn chỉnh. Nó không chỉ đơn thuần là gọi một thư viện AI, mà còn giải quyết triệt để các vấn đề về **lấy dữ liệu có xác thực** (Authenticated data fetching), **xử lý luồng** (Stream processing) và **giao tiếp đa ngôn ngữ** (Node.js - Python) để tạo ra trải nghiệm người dùng cuối đơn giản nhất cho sinh viên.