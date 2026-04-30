Dưới đây là phân tích chi tiết về dự án **Claude Video Vision** dựa trên mã nguồn bạn đã cung cấp. Đây là một plugin được thiết kế cực kỳ bài bản cho Claude Code, đóng vai trò là một "lớp tri giác" (perception layer) hơn là một lớp diễn giải.

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Ngôn ngữ & Runtime:** TypeScript chạy trên Node.js (v20+). Sử dụng bộ công cụ hiện đại như `zod` để kiểm tra kiểu dữ liệu (validation) và `vitest` để kiểm thử.
*   **Xử lý Video/Audio:** **FFmpeg** là "trái tim" của hệ thống. Nó được dùng để:
    *   Trích xuất khung hình (frames).
    *   Trích xuất âm thanh (WAV 16kHz mono).
    *   Phân tích kỹ thuật (phát hiện cảnh cắt, khoảng lặng, độ nhiễu, độ sáng).
*   **Model Context Protocol (MCP):** Sử dụng SDK của Anthropic để kết nối Claude với các công cụ (tools) cục bộ.
*   **AI Backends (Đa dạng):**
    *   **Cloud:** Google Gemini API (Gemini 1.5 Flash - tối ưu cho audio) và OpenAI Whisper API.
    *   **Local:** `whisper.cpp` (tối ưu cho hiệu suất) hoặc Python `openai-whisper`.
*   **Hệ thống phiên làm việc (Session System):** Lưu trữ cache các khung hình và kết quả phân tích dưới dạng JSON manifest để tránh xử lý lại cùng một video nhiều lần.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của dự án tuân thủ nguyên tắc **Phân tách trách nhiệm (Separation of Concerns)**:

*   **Kiến trúc Phân tầng:**
    *   **Skill Layer (`skills/`):** Dạy Claude cách tư duy, khi nào nên dùng công cụ nào.
    *   **Tool Layer (`mcp-server/src/tools/`):** Các điểm cuối API để Claude gọi.
    *   **Service Layer (`extractors/`, `backends/`):** Logic thực thi việc trích xuất và gọi AI.
    *   **Utility Layer:** Xử lý hệ thống, checksum, định dạng thời gian.
*   **Triết lý "Tri giác trước, Suy luận sau":** Thay vì gửi toàn bộ video lên một AI đắt đỏ, hệ thống chia nhỏ video thành các thành phần (hình ảnh + văn bản audio) rồi mới đưa vào ngữ cảnh của Claude.
*   **Khả năng thích ứng (Adaptability):** Hệ thống không trích xuất video một cách mù quáng. Nó tính toán FPS dựa trên độ dài video (video ngắn = FPS cao, video dài = FPS thấp) để tối ưu hóa giới hạn Token của Claude.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Phân tích hai giai đoạn (Two-Pass Analysis):**
    *   *Giai đoạn 1 (`video_analyze`):* Chạy các filter FFmpeg (`scdet`, `silencedetect`) để tìm "điểm nóng" (điểm cắt cảnh, điểm có tiếng nói).
    *   *Giai đoạn 2 (`video_watch`):* Chỉ trích xuất khung hình chi tiết tại các điểm quan trọng đã tìm thấy.
*   **Quản lý bộ nhớ & Token:**
    *   **View Sampling:** Kỹ thuật lấy mẫu N khung hình cách đều nhau thay vì lấy tất cả, giúp Claude không bị "ngợp" dữ liệu.
    *   **Frame Description Mode:** Một sub-agent (Haiku/Sonnet) mô tả hình ảnh thành văn bản để tiết kiệm token hình ảnh (vốn rất đắt).
*   **Bảo mật & Toàn vẹn:**
    *   **Streaming SHA-256 Checksum:** Khi tải model Whisper cục bộ, hệ thống kiểm tra mã hash theo thời gian thực để đảm bảo file không bị hỏng hoặc giả mạo.
    *   **Shell Injection Prevention:** Sử dụng `execFile` thay vì `exec` để truyền đối số dưới dạng mảng, chặn đứng các cuộc tấn công chèn lệnh qua tên file.
*   **Timestamp Alignment:** Kỹ thuật dịch chuyển (shift) mốc thời gian audio để khớp chính xác với khung hình khi người dùng yêu cầu xem một đoạn giữa video (ví dụ: bắt đầu từ 01:30).

### 4. Tóm tắt luồng hoạt động (Operational Flow)

Khi người dùng nói: *"Phân tích video demo.mp4 này"*:

1.  **Nhận diện:** Claude kích hoạt Skill `video-perception`.
2.  **Kiểm tra (`video_info`):** Lấy thông số kỹ thuật (độ dài, độ phân giải).
3.  **Phân tích cấu trúc (`video_analyze`):**
    *   FFmpeg quét video tìm các đoạn chuyển cảnh và khoảng lặng.
    *   Backend (Gemini/Whisper) chuyển audio thành văn bản kèm mốc thời gian (timestamps).
4.  **Lập kế hoạch:** Claude dựa vào kết quả phân tích để quyết định: *"Đoạn này có hội thoại quan trọng, mình sẽ lấy 2 FPS. Đoạn kia là cảnh tĩnh, chỉ lấy 0.1 FPS"*.
5.  **Trích xuất (`video_watch`):** FFmpeg cắt các khung hình (JPEGs) theo kế hoạch.
6.  **Lưu trữ (Indexing):** Lưu khung hình vào `~/.claude-video-vision/sessions/` để dùng lại nếu người dùng hỏi tiếp.
7.  **Tổng hợp:** Claude nhận được danh sách hình ảnh + transcript và đưa ra câu trả lời cuối cùng cho người dùng.

### Đánh giá:
Đây là một dự án có độ hoàn thiện rất cao. Nó không chỉ đơn thuần là gọi API, mà còn giải quyết được các bài toán khó về **chi phí Token**, **hiệu suất máy tính cục bộ** và **trải nghiệm người dùng** thông qua việc kết hợp thông minh giữa các công cụ truyền thống (FFmpeg) và AI hiện đại.