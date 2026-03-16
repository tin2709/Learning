Dựa trên cấu trúc thư mục và nội dung mã nguồn bạn cung cấp, dưới đây là phân tích chi tiết về dự án **AutoClip (AI Video Intelligent Clipping System)**:

### 1. Công nghệ cốt lõi (Core Tech Stack)

Hệ thống được xây dựng trên một ngăn xếp công nghệ hiện đại, tập trung vào khả năng xử lý bất đồng bộ và xử lý tệp tin lớn:

*   **Backend (Python Ecosystem):**
    *   **FastAPI:** Framework web hiện đại, hiệu suất cao dùng để xây dựng RESTful API và xử lý WebSocket.
    *   **Celery & Redis:** Đây là "xương sống" của hệ thống xử lý video. Celery quản lý các tác vụ nặng (tải video, cắt ghép, chạy AI) trong hàng đợi, Redis đóng vai trò là Message Broker.
    *   **SQLAlchemy & SQLite:** Quản lý dữ liệu quan hệ (Project, Clip, Task). Hệ thống hỗ trợ chuyển đổi sang PostgreSQL cho môi trường sản xuất.
    *   **FFmpeg:** Công cụ dòng lệnh mạnh mẽ dùng để cắt, ghép và xử lý luồng video/audio.
    *   **yt-dlp:** Thư viện hàng đầu để tải video từ YouTube, Bilibili và hàng trăm nền tảng khác.
*   **AI & Language Processing:**
    *   **LLM (Qwen, OpenAI, Gemini):** Sử dụng các mô hình ngôn ngữ lớn để hiểu nội dung video thông qua bản dịch (subtitle/transcript).
    *   **Whisper (OpenAI) / Bcut ASR:** Chuyển đổi giọng nói trong video thành văn bản (Speech-to-Text) để làm đầu vào cho AI.
*   **Frontend (Modern Web):**
    *   **React 18 & TypeScript:** Xây dựng giao diện người dùng tin cậy và dễ bảo trì.
    *   **Ant Design (AntD):** Thư viện UI component chuyên nghiệp.
    *   **Zustand:** Quản lý trạng thái (state management) nhẹ nhàng và hiệu quả hơn Redux.
    *   **Vite:** Công cụ build frontend cực nhanh.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án áp dụng kiến trúc **Microservices-lite** và **Pipeline-based processing**:

*   **Tách biệt Tiền - Hậu (Frontend-Backend Separation):** Backend chỉ cung cấp API và xử lý dữ liệu, Frontend đảm nhận việc hiển thị và tương tác người dùng.
*   **Kiến trúc hướng Pipeline (Pipeline Pattern):** Quy trình xử lý video được chia thành các bước (steps) riêng biệt trong thư mục `backend/pipeline/`:
    *   `Step 1: Outline` (Trích xuất đại ý)
    *   `Step 2: Timeline` (Phân tích mốc thời gian)
    *   `Step 3: Scoring` (Chấm điểm mức độ hấp dẫn/giá trị)
    *   ... đến `Step 6: Video` (Xuất bản video hoàn chỉnh).
    *   *Lợi ích:* Dễ dàng debug từng giai đoạn và có thể chạy lại một bước cụ thể mà không cần bắt đầu từ đầu.
*   **Xử lý bất đồng bộ (Event-Driven/Async):** Do việc xử lý video tốn nhiều thời gian và tài nguyên, hệ thống không bắt người dùng chờ đợi trên HTTP request. Thay vào đó, nó gửi phản hồi "đã nhận tác vụ" và cập nhật tiến độ qua **WebSocket** hoặc **Simple Progress System**.
*   **Quản lý tài nguyên (Storage Strategy):** Dữ liệu được tổ chức theo ID dự án (`data/projects/{uuid}`), tách biệt giữa dữ liệu thô (raw), dữ liệu trung gian (metadata) và kết quả cuối cùng (output).

### 3. Các kỹ thuật chính (Key Techniques)

*   **AI-Driven Highlighting:** Không chỉ cắt video ngẫu nhiên, hệ thống gửi transcript của video cho LLM cùng với các "Prompt" chuyên biệt (trong `backend/prompt/`) để nhận diện các phân đoạn "đắt giá" dựa trên ngữ cảnh (ví dụ: kiến thức, hài hước, kinh doanh).
*   **Cơ chế lưu trữ tối ưu (Optimized Storage):** Hệ thống chỉ lưu đường dẫn tệp trong Database thay vì lưu Blob dữ liệu, giúp DB nhẹ nhàng và tăng tốc độ truy xuất tệp lớn.
*   **Real-time Progress Tracking:** Kết hợp Redis Pub/Sub và WebSocket để đẩy trạng thái xử lý (ví dụ: "Đang tải 45%", "Đang chạy AI...") lên giao diện người dùng theo thời gian thực.
*   **Đa dạng hóa nhà cung cấp AI (LLM Provider Management):** Thông qua `llm_manager.py`, hệ thống có thể linh hoạt chuyển đổi giữa Tongyi Qwen, OpenAI, Gemini tùy theo cấu hình API Key của người dùng.
*   **Bilibili Integration:** Kỹ thuật đăng nhập qua QR Code và quản lý Cookie giúp tự động hóa việc tải lên và quản lý tài khoản trên nền tảng Bilibili mà không cần can thiệp thủ công.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Tiếp nhận (Input):** Người dùng gửi link YouTube/Bilibili hoặc upload file local.
2.  **Chuẩn bị (Preparation):**
    *   `yt-dlp` tải video về thư mục `raw/`.
    *   Nếu không có phụ đề, hệ thống chạy Whisper để tạo file `.srt`.
3.  **Phân tích (AI Analysis):**
    *   Bản dịch được chia nhỏ (chunking) và gửi cho LLM.
    *   LLM dựa trên Prompt để tóm tắt, xác định mốc thời gian (start/end) và chấm điểm (scoring) từng đoạn.
4.  **Hậu kỳ (Video Generation):**
    *   Dựa trên danh sách các đoạn được LLM chọn, `VideoEditor` gọi FFmpeg để cắt video gốc thành các đoạn clip nhỏ.
    *   Tự động tạo tiêu đề và lý do đề xuất cho từng clip.
5.  **Quản lý & Xuất bản (Management & Upload):**
    *   Người dùng xem lại các đoạn cắt trên UI.
    *   Hệ thống có thể gom các đoạn clip thành "Bộ sưu tập" (Collection).
    *   (Đang phát triển) Tự động đẩy video đã cắt lên Bilibili thông qua tài khoản đã liên kết.

**Đánh giá chung:** Đây là một hệ thống hoàn chỉnh, có tính ứng dụng cao cho những người làm sáng tạo nội dung (Content Creator) muốn tối ưu hóa quy trình làm video highlight từ các luồng livestream hoặc video dài. Kiến trúc này rất ổn định và dễ mở rộng sang các nền tảng video khác.