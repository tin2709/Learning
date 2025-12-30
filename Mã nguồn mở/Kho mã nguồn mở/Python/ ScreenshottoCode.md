Dưới đây là bản phân tích chi tiết về dự án **screenshot-to-code** dựa trên mã nguồn bạn đã cung cấp, trình bày dưới dạng tài liệu kỹ thuật tiếng Việt.

---

# Phân tích Dự án Screenshot-to-Code

Dự án này là một công cụ mạnh mẽ sử dụng Trí tuệ nhân tạo (AI) để biến hình ảnh, bản thiết kế (Figma) hoặc video quay màn hình thành mã nguồn chức năng (HTML/Tailwind, React, Vue, v.v.).

## 1. Công nghệ cốt lõi (Core Technologies)

### Backend (Python/FastAPI)
*   **FastAPI:** Framework web hiệu năng cao được sử dụng để xây dựng API và quản lý kết nối WebSocket.
*   **WebSockets:** Công nghệ chủ chốt để truyền phát (stream) mã nguồn từ AI đến người dùng theo thời gian thực.
*   **LLMs (Large Language Models):** 
    *   **Anthropic (Claude 3.7/4.5 Sonnet/Opus):** Được ưu tiên cho chất lượng code và khả năng "suy nghĩ" (thinking mode).
    *   **OpenAI (GPT-4o/o1):** Dòng mô hình vision mạnh mẽ để hiểu giao diện.
    *   **Google Gemini 2.0:** Tùy chọn bổ sung cho xử lý đa phương tiện.
*   **Image Generation:** DALL-E 3 hoặc Flux Schnell (qua Replicate) để tạo hình ảnh thực tế thay thế cho các khung ảnh giữ chỗ (placeholder).
*   **Poetry:** Quản lý gói và môi trường ảo Python.
*   **Xử lý video/hình ảnh:** Pillow (xử lý ảnh) và MoviePy (trích xuất frame từ video).

### Frontend (React/TypeScript)
*   **Vite:** Công cụ build nhanh cho React.
*   **Zustand:** Quản lý trạng thái (State Management) cho toàn bộ ứng dụng và lịch sử các phiên bản (commits).
*   **Tailwind CSS & Shadcn/UI:** Thư viện giao diện để xây dựng UI ứng dụng mượt mà.
*   **CodeMirror:** Trình soạn thảo mã nguồn tích hợp để người dùng có thể chỉnh sửa trực tiếp.
*   **html2canvas:** Dùng để chụp ảnh màn hình bản xem trước (preview) giúp thực hiện tính năng chỉnh sửa đệ quy.

---

## 2. Tư duy kiến trúc (Architectural Thinking)

### Hệ thống Variant (Non-blocking)
Thay vì bắt người dùng chờ đợi tất cả các phương án AI hoàn thành, dự án sử dụng kiến trúc **Non-blocking**. Mỗi biến thể (Variant) được tạo ra bởi một mô hình AI khác nhau (ví dụ: một cái bằng Claude, một cái bằng GPT-4o) sẽ chạy song song. Biến thể nào xong trước sẽ hiển thị ngay lập tức để người dùng tương tác, giúp tối ưu hóa trải nghiệm người dùng (Perceived Performance).

### Hệ thống Commit & Versioning
Kiến trúc lưu trữ dữ liệu theo dạng **Cây lịch sử (Git-like)**:
*   Mỗi lần tạo code hoặc cập nhật được coi là một `Commit`.
*   Người dùng có thể quay lại bất kỳ phiên bản nào (Undo/Redo) hoặc rẽ nhánh (branching) từ một phiên bản cũ để yêu cầu một thay đổi khác.

### Pipeline Middleware (Backend)
Luồng xử lý tại Backend được tổ chức theo mô hình **Pipeline/Middleware**:
*   `WebSocketSetupMiddleware`: Thiết lập kết nối.
*   `ParameterExtraction`: Kiểm tra API key và cấu hình stack.
*   `ModelSelection`: Tự động chọn mô hình phù hợp với key người dùng cung cấp.
*   `CodeGeneration`: Thực thi song song việc gọi các API AI.

---

## 3. Các kỹ thuật chính (Key Techniques)

### Prompt Engineering động
Hệ thống không sử dụng một Prompt cố định. Dựa vào "Stack" người dùng chọn (React, Vue, hay Tailwind), hệ thống sẽ lắp ghép các `SYSTEM_PROMPT` khác nhau, chỉ định rõ các thư viện cần dùng (CDN link) và quy tắc viết code (ví dụ: không được ghi chú lười biếng dạng `<!-- repeat items -->`).

### Trích xuất Frame từ Video
Khi đầu vào là video, hệ thống sử dụng `MoviePy` để tính toán và cắt ra tối đa 20 khung hình quan trọng nhất (theo khoảng cách thời gian) để gửi vào mô hình Vision, giúp AI hiểu được luồng tương tác (user flow) thay vì chỉ là ảnh tĩnh.

### Chế độ "Select and Edit" (Visual Feedback Loop)
Đây là một kỹ thuật thông minh:
1.  Người dùng click vào một phần tử trong Iframe preview.
2.  Frontend xác định thẻ HTML tương ứng và gửi đoạn mã đó kèm yêu cầu sửa đổi về Backend.
3.  AI nhận được ngữ cảnh cụ thể của phần tử đó và thực hiện cập nhật chính xác vị trí cần sửa.

### Xử lý hình ảnh thông minh
Trước khi gửi ảnh lên Claude (vốn có giới hạn 5MB), Backend tự động kiểm tra kích thước, resize và nén ảnh về định dạng JPEG để đảm bảo không vi phạm giới hạn của API mà vẫn giữ đủ độ chi tiết cho AI nhận diện.

---

## 4. Tóm tắt luồng hoạt động (Operational Workflow)

1.  **Khởi tạo (Input):** Người dùng tải lên ảnh/video, nhập URL (qua ScreenshotOne API) hoặc dán code có sẵn.
2.  **Thiết lập (Setup):** Người dùng chọn Stack (ví dụ: React + Tailwind) và mô hình AI.
3.  **Thực thi (Execution):** 
    *   Frontend mở kết nối WebSocket đến Backend.
    *   Backend nhận yêu cầu, lắp ghép Prompt kèm theo ảnh dưới dạng Base64.
    *   Backend gọi đồng thời nhiều mô hình AI (Variants).
4.  **Truyền phát (Streaming):** Mã nguồn được AI tạo ra đến đâu, Backend đẩy qua WebSocket về Frontend đến đó. Người dùng thấy code được viết ra theo thời gian thực trong cửa sổ Preview.
5.  **Hoàn thiện (Post-processing):**
    *   Sau khi code xong, nếu người dùng bật tính năng tạo ảnh, Backend sẽ tìm các thẻ `<img>` với alt-text, gọi DALL-E 3 để tạo ảnh thật và thay thế vào code.
6.  **Chỉnh sửa (Iteration):** Người dùng nhập yêu cầu sửa lỗi (ví dụ: "làm nút này to hơn"). Toàn bộ lịch sử code trước đó được gửi ngược lại AI làm ngữ cảnh (context) để thực hiện các thay đổi tiếp theo.

---
*Dự án này là minh chứng tiêu biểu cho việc kết hợp giữa khả năng suy luận của Large Language Models và các kỹ thuật lập trình truyền thống để tạo ra một công cụ năng suất cao.*