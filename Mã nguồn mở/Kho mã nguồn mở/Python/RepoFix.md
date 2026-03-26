Dựa trên mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chuyên sâu về hệ thống **RepoFix**:

### 1. Công nghệ cốt lõi (Core Tech Stack)

RepoFix được xây dựng bằng ngôn ngữ **Python (3.10+)**, tận dụng hệ sinh thái thư viện hiện đại để xử lý hệ thống và AI:

*   **CLI & giao diện:** Sử dụng `Typer` để xây dựng CLI nhanh chóng và `Rich` để hiển thị các bảng biểu, thanh trạng thái (spinner), và định dạng log màu sắc chuyên nghiệp.
*   **Xử lý Git & Hệ thống:** Sử dụng `GitPython` để thao tác với các kho mã nguồn và `psutil` kết hợp với `subprocess` để quản lý tiến trình con, kiểm tra trạng thái CPU/Memory.
*   **Xử lý dữ liệu & Cấu hình:** `Pydantic` (v2) được dùng để định nghĩa các schema dữ liệu và validate cấu hình (RunnerConfig). Hỗ trợ các định dạng `YAML`, `TOML`, `JSON`.
*   **Trí tuệ nhân tạo (Hybrid AI):**
    *   **Local LLM:** Tích hợp `llama-cpp-python` để chạy mô hình **Qwen2.5-Coder-3B** ngay trên máy người dùng (On-device), đảm bảo riêng tư và hoạt động offline.
    *   **Cloud LLM:** Hỗ trợ đa nền tảng qua `google-genai` (Gemini) và `httpx` (để gọi OpenAI, Anthropic, hoặc các OpenAI-compatible endpoints như Ollama, vLLM).
*   **Môi trường cô lập:** Có logic quản lý Virtualenv (Python) và node_modules (Node.js) tùy biến theo từng nhánh (branch) của dự án.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của RepoFix chuyển dịch từ "README dành cho người đọc" sang "README dành cho máy thực thi":

*   **Heuristic-first, AI-second:** Hệ thống ưu tiên các quy tắc dựa trên kinh nghiệm (Heuristics) và Regex để nhận diện lỗi phổ biến (cổng đang dùng, thiếu thư viện). Chỉ khi các quy tắc này thất bại, nó mới "leo thang" (escalate) lên AI.
*   **Pipeline Tự chữa lành (Self-healing Loop):** Thiết kế theo vòng lặp: *Chạy -> Lỗi -> Phân loại -> Sửa -> Thử lại*. Số lần thử lại (`retries`) có thể cấu hình được.
*   **Kiến trúc Stateless & Memory:** Mặc dù hoạt động trên các repo khác nhau, RepoFix có một lớp `memory` (SQLite/JSON) để lưu trữ các giải pháp đã thành công cho từng loại lỗi, giúp các lần chạy sau nhanh hơn.
*   **Trình đăng ký tiến trình (Process Registry):** Thay vì chỉ chạy lệnh rồi bỏ đó, RepoFix có cơ chế `registry` để theo dõi PID (Process ID), đường dẫn log, giúp người dùng quản lý vòng đời app (`start`, `stop`, `restart`, `logs`).

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Regex-driven Error Classification:** Trong `fixing/classifier.py`, hệ thống sử dụng hàng chục mẫu Regex phức tạp để "bắt" lỗi từ cả `stdout` và `stderr`. Ví dụ: phân biệt giữa lỗi thiếu package Node.js và lỗi sai file entry point (đều là "Cannot find module").
*   **Concurrent Log Streaming:** Trong `executor.py`, hệ thống sử dụng đa luồng (`threading`) để đọc đồng thời hai luồng `stdout` và `stderr` mà không làm nghẽn tiến trình chính, cho phép hiển thị log thời gian thực trong khi vẫn phân tích lỗi ngầm.
*   **Robust JSON Extraction:** Kỹ thuật `extract_json_object` trong `llm_json.py` cực kỳ mạnh mẽ. Nó không chỉ tìm markdown fences (```json) mà còn sử dụng thuật toán đếm ngoặc nhọn `{}` để trích xuất JSON từ các phản hồi thô của LLM vốn thường kèm theo văn bản giải thích.
*   **Docker Integration:** Tự động phát hiện `docker-compose` và đặc biệt là kỹ thuật `docker_compose_bind_fix.py` để sửa lỗi Docker tự tạo thư mục rỗng khi người dùng quên tạo file config trên máy host.

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng hoạt động của RepoFix gồm 7 giai đoạn chính:

1.  **Phân giải (Resolution):** Tiếp nhận URL GitHub hoặc đường dẫn cục bộ. Nếu là URL, nó sẽ clone vào thư mục cache `~/.repofix/repos/`.
2.  **Khám phá (Discovery):**
    *   Quét tệp tin để xác định Stack (Node, Python, Go...).
    *   Phân tích README và các file manifest (`package.json`, `Makefile`, `Dockerfile`) để tìm lệnh cài đặt và lệnh chạy.
3.  **Chuẩn bị (Preparation):** Kiểm tra môi trường, tự động tạo file `.env` từ `.env.example`, kiểm tra quyền ghi của các thư mục.
4.  **Thực thi (Execution):** Chạy lệnh cài đặt (`install`) và khởi động ứng dụng (`run`).
5.  **Giám sát & Phân tích (Monitoring):** Log được stream ra màn hình. Nếu ứng dụng crash hoặc thoát với mã lỗi không bằng 0, hệ thống thu thập 50-100 dòng log cuối cùng.
6.  **Sửa lỗi (Fixing):**
    *   **Classifier:** Xác định nguyên nhân (ví dụ: `port_conflict`).
    *   **Fixer:** Thực hiện hành động sửa (ví dụ: đổi cổng, `npm install <package>`).
    *   **Escalation:** Nếu là lỗi lạ, gửi context sang Local LLM hoặc Cloud AI để lấy lệnh sửa dạng JSON.
7.  **Đăng ký (Lifecycle):** Khi app chạy thành công, thông tin được lưu vào `processes.json` để quản lý về sau qua các lệnh `repofix ps` hoặc `repofix logs`.

**Tóm lại:** RepoFix không chỉ là một công cụ clone code, mà là một **"Hệ điều hành mini cho môi trường dev"**, biến quá trình thiết lập dự án thủ công thành một quy trình tự động hóa có khả năng học hỏi từ sai lầm.