Dựa trên mã nguồn và cấu trúc thư mục của dự án **SupoClip**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và quy trình hoạt động:

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

**Backend (Python-centric):**
*   **Framework:** `FastAPI` (Async) để xây dựng API hiệu năng cao.
*   **Xử lý tác vụ nền:** `arq` kết hợp với `Redis` để quản lý hàng đợi (job queue) và xử lý video không đồng bộ.
*   **Cơ sở dữ liệu:** `PostgreSQL` sử dụng `SQLAlchemy` và `asyncpg` để tương tác async.
*   **Xử lý Video & Ảnh:** 
    *   `MoviePy`: Cắt ghép, chèn hiệu ứng, xử lý âm thanh.
    *   `OpenCV` & `MediaPipe`: Phát hiện khuôn mặt (Face Detection) để crop video 9:16 tự động (Auto-reframe).
    *   `yt-dlp`: Tải video từ YouTube với chất lượng cao.
*   **AI & NLP:**
    *   `AssemblyAI`: Chuyển đổi âm thanh thành văn bản (Speech-to-Text) với độ chính xác cao và timestamp cấp độ từ.
    *   `Pydantic AI`: Sử dụng LLM (OpenAI, Gemini, Claude, Ollama) để phân tích kịch bản, chấm điểm "viral" và tìm đoạn cắt tiềm năng.
*   **Khác:** `FFmpeg` (engine cốt lõi cho mọi thao tác video).

**Frontend (Modern Web):**
*   **Framework:** `Next.js 15` (App Router) với `React 19`.
*   **Styling:** `Tailwind CSS v4` và `ShadCN UI`.
*   **ORM:** `Prisma` để quản lý schema và truy vấn database từ phía Next.js.
*   **Authentication:** `Better Auth` (một giải pháp auth mới, bảo mật và linh hoạt).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án đã trải qua một đợt tái cấu trúc (Refactoring) quan trọng từ dạng Monolith sang **Layered Architecture (Kiến trúc phân lớp)**:

*   **Tách biệt tiến trình (Decoupling):** API Server và Worker xử lý video là hai thực thể riêng biệt. Điều này giúp API phản hồi cực nhanh (<100ms) trong khi các tác vụ nặng (tải video, render) chạy ngầm trong Worker.
*   **Kiến trúc 3 lớp (Layered Pattern):**
    *   *Routes:* Tiếp nhận HTTP request, xác thực người dùng.
    *   *Services:* Chứa logic nghiệp vụ (business logic) như điều phối việc cắt clip, xử lý B-roll.
    *   *Repositories:* Thao tác trực tiếp với Database bằng SQL thuần (Raw SQL) qua `asyncpg` để tối ưu tốc độ.
*   **Real-time Communication:** Sử dụng **SSE (Server-Sent Events)** thay vì Polling. Server sẽ "đẩy" tiến độ xử lý (downloading, transcribing, rendering) xuống Frontend theo thời gian thực qua Redis Pub/Sub.
*   **Tính mở rộng (Scalability):** Nhờ sử dụng Redis Queue, hệ thống có thể tăng số lượng Worker lên nhiều lần để xử lý hàng nghìn video cùng lúc mà không làm treo API.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **AI Virality Scoring:** Hệ thống không cắt video ngẫu nhiên. Nó gửi transcript cho LLM để đánh giá dựa trên các tiêu chí: *Hook strength, Engagement, Value, Shareability*. Mỗi đoạn cắt đều có lý giải (reasoning) từ AI.
*   **Smart Auto-Cropping:** Sử dụng chuỗi fallback (MediaPipe -> OpenCV DNN -> Haar cascade) để tìm chủ thể trong video ngang (16:9) và tự động căn giữa khi chuyển sang dọc (9:16).
*   **Word-Level Subtitle Sync:** Đồng bộ hóa phụ đề từng từ một. Kỹ thuật này sử dụng dữ liệu timestamp từ AssemblyAI để tạo hiệu ứng chữ nhảy (pop-up) hoặc karaoke giống như phong cách của Alex Hormozi hay MrBeast.
*   **B-Roll Integration:** AI tự động gợi ý các từ khóa từ transcript, sau đó gọi API của `Pexels` để tải video minh họa (B-roll) chèn vào những đoạn thiếu hình ảnh hấp dẫn.
*   **Template System:** Cung cấp các preset sẵn có (Hormozi, MrBeast, TikTok style) bao gồm cấu hình font, màu sắc, vị trí và hiệu ứng animation cho phụ đề.

---

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Input:** Người dùng nhập link YouTube hoặc upload file video từ Frontend.
2.  **Queueing:** Backend tạo một `Task` trong Postgres với trạng thái `queued` và đẩy một job vào Redis.
3.  **Processing (Worker):**
    *   **Tải video:** Sử dụng `yt-dlp`.
    *   **Transcribe:** Gửi audio đến AssemblyAI để lấy transcript có timestamp từng từ.
    *   **Analysis:** Gửi transcript cho LLM để xác định 3-7 đoạn cắt "viral" nhất.
    *   **Generation:** Worker sử dụng MoviePy để cắt video, căn giữa khuôn mặt, và chèn phụ đề theo template đã chọn.
4.  **Notification:** Trong suốt quá trình, Worker gửi message qua Redis Pub/Sub, API Server nhận được và đẩy qua SSE để người dùng thấy thanh progress bar chạy trên màn hình.
5.  **Output:** Sau khi hoàn tất, các file `.mp4` được lưu vào thư mục `clips/` và người dùng có thể xem, chỉnh sửa thêm hoặc tải về.

**Kết luận:** SupoClip là một dự án kết hợp xuất sắc giữa sức mạnh của các thư viện xử lý media truyền thống (FFmpeg, MoviePy) và trí tuệ nhân tạo hiện đại (LLM, STT) để tự động hóa quy trình sáng tạo nội dung ngắn.