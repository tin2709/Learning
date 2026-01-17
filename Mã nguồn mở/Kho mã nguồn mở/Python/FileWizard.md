Dưới đây là bản phân tích chi tiết dự án **File Wizard** dựa trên mã nguồn bạn đã cung cấp, được trình bày bằng tiếng Việt:

---

### 1. Công nghệ Cốt lõi (Core Technology)

Dự án là một ứng dụng Web tập trung vào việc xử lý tệp tin đa năng, kết hợp giữa Web framework hiện đại và các công cụ dòng lệnh (CLI) mạnh mẽ:

*   **Backend Framework:** **FastAPI** (Python) – Lựa chọn tối ưu cho hiệu suất cao, xử lý bất đồng bộ (async) và tự động tạo tài liệu API.
*   **Task Queue (Hàng đợi công việc):** **Huey** – Một thư viện Python nhẹ để xử lý các tác vụ chạy ngầm (background jobs) như chuyển đổi file nặng hoặc OCR mà không làm treo giao diện web.
*   **Database:** **SQLAlchemy** – ORM để quản lý lịch sử công việc (job history) và trạng thái xử lý.
*   **Frontend:** Sử dụng **Vanilla HTML/JS/CSS** kết hợp với thư viện **Choices.js** (để tạo dropdown đẹp). Không sử dụng các framework nặng như React hay Vue, giúp trang web load rất nhanh.
*   **Công cụ xử lý "Xương sống":**
    *   **Tài liệu:** LibreOffice (soffice), Pandoc, MarkItDown, Docling.
    *   **Hình ảnh:** ImageMagick, GraphicsMagick, libvips, Inkscape, resvg.
    *   **Âm thanh/Video:** FFmpeg, SoX.
    *   **AI/OCR:** Tesseract OCR, ocrmypdf, Faster-Whisper (Chuyển giọng nói thành văn bản), Kokoro/Piper (Chuyển văn bản thành giọng nói - TTS).
*   **Containerization:** **Docker & Docker Compose** với khả năng hỗ trợ cả CPU và GPU (CUDA).

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Technical & Architectural Thinking)

*   **Kiến trúc Wrapper (Lớp bọc):** File Wizard không tự viết lại logic chuyển đổi file mà đóng vai trò là một "nhạc trưởng". Nó bao bọc các công cụ CLI kinh điển, cung cấp giao diện đồ họa thân thiện cho những dòng lệnh phức tạp.
*   **Cấu hình linh hoạt (YAML-driven):** Tư duy kiến trúc ở đây rất mở. Toàn bộ các câu lệnh CLI được định nghĩa trong file `settings.default.yml`. Người dùng có thể tự thêm công cụ mới hoặc sửa đổi tham số dòng lệnh thông qua giao diện `/settings` mà không cần can thiệp vào code Python.
*   **Xử lý bất đồng bộ và Đa nhiệm:** Sử dụng **Supervisor** để quản lý đồng thời cả Web server (Gunicorn/Uvicorn) và Worker xử lý hàng đợi (Huey) bên trong container. Điều này đảm bảo tính ổn định: nếu một tiến trình chết, nó sẽ tự động khởi động lại.
*   **Phân tách môi trường (Multi-stage Docker Builds):** Dự án chia làm 3 phiên bản build:
    *   `small`: Tối ưu dung lượng, loại bỏ các thư viện nặng như TeX.
    *   `full`: Đầy đủ tính năng.
    *   `cuda`: Hỗ trợ tăng tốc phần cứng qua GPU cho các mô hình AI (Whisper, OCR).

---

### 3. Các kỹ thuật chính nổi bật (Key Highlights)

*   **Command Templating (Mẫu câu lệnh):** Hệ thống sử dụng các placeholder như `{input}`, `{output}`, `{filter}` trong file cấu hình. Khi thực thi, ứng dụng sẽ tự động thay thế chúng bằng đường dẫn file thực tế, giúp việc tích hợp công cụ CLI mới cực kỳ dễ dàng.
*   **Real-time Status Polling:** Frontend sử dụng kỹ thuật polling (truy vấn định kỳ) để cập nhật trạng thái công việc (Pending, Processing, Completed, Failed) và tiến độ xử lý mà không cần reload trang.
*   **Bảo mật OIDC:** Hỗ trợ xác thực qua OpenID Connect (như Google, Keycloak, Authelia), cho phép triển khai ứng dụng an toàn trên môi trường internet thay vì chỉ dùng cục bộ.
*   **Tối ưu hóa tài nguyên AI:** Có cơ chế "Model Inactivity Timeout" để giải phóng bộ nhớ RAM/VRAM khi các mô hình AI (như Whisper) không được sử dụng trong một khoảng thời gian.

---

### 4. Tóm tắt luồng hoạt động (Activity Flow Summary)

Luồng hoạt động của dự án diễn ra qua các bước sau:

1.  **Tiếp nhận (Input):** Người dùng kéo thả file vào giao diện Web. File được tải lên và lưu vào thư mục `uploads/`.
2.  **Đăng ký tác vụ (Job Creation):** 
    *   FastAPI nhận yêu cầu, tạo một bản ghi "Job" trong database với trạng thái `pending`.
    *   Yêu cầu được đẩy vào hàng đợi **Huey**.
3.  **Xử lý ngầm (Background Processing):**
    *   Worker của Huey lấy công việc ra, tìm cấu hình công cụ tương ứng trong `settings.yml`.
    *   Ứng dụng gọi lệnh hệ thống (Subprocess) để thực thi công cụ CLI (ví dụ: chạy lệnh `ffmpeg` để convert MP3).
    *   Trạng thái chuyển sang `processing`.
4.  **Hoàn tất và Phản hồi (Output):**
    *   File kết quả được lưu vào thư mục `processed/`.
    *   Database cập nhật trạng thái `completed`.
    *   Giao diện người dùng (đang polling) nhận thấy trạng thái đã xong và hiển thị nút **Download**.
5.  **Quản lý (Management):** Người dùng có thể vào mục History để xem lại, tải lại hoặc vào Settings để cấu hình lại các thông số kỹ thuật.

---
**Kết luận:** Dự án này là một giải pháp **Self-hosted** tuyệt vời cho cá nhân hoặc doanh nghiệp nhỏ muốn sở hữu một "thụy sĩ quân dụng kiếm" về xử lý file, đảm bảo quyền riêng tư vì dữ liệu không bao giờ rời khỏi máy chủ của chính họ.