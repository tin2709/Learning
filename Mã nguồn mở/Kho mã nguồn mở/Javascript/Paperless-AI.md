Dựa trên cấu trúc thư mục và nội dung mã nguồn của dự án **Paperless-AI**, dưới đây là bản phân tích chi tiết về công nghệ, kiến trúc và cách thức hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Technology Stack)
Dự án sử dụng mô hình kết hợp (Hybrid) giữa Node.js và Python để tận dụng thế mạnh của cả hai:

*   **Backend chính (Node.js/Express):** Đảm nhiệm vai trò điều phối chính, quản lý giao diện người dùng, xác thực (JWT), kết nối với API của Paperless-ngx và quản lý hàng đợi xử lý tài liệu.
*   **Dịch vụ RAG & Xử lý ngôn ngữ (Python/FastAPI):** Tệp `main.py` là trung tâm của hệ thống RAG (Retrieval-Augmented Generation). Nó sử dụng các thư viện AI mạnh mẽ như:
    *   `sentence-transformers`: Để tạo vector nhúng (embeddings) cho tài liệu.
    *   `ChromaDB`: Cơ sở dữ liệu vector để lưu trữ và tìm kiếm ngữ nghĩa.
    *   `rank-bm25`: Thuật toán tìm kiếm văn bản truyền thống để kết hợp với tìm kiếm ngữ nghĩa (Hybrid Search).
*   **Cơ sở dữ liệu:** `Better-sqlite3` được dùng để lưu trữ trạng thái xử lý tài liệu, lịch sử và thông tin người dùng cục bộ.
*   **Giao diện (Frontend):** Sử dụng `EJS` (Embedded JavaScript templates) để dựng giao diện phía server, kết hợp với `Tailwind CSS` và `Chart.js` để hiển thị biểu đồ thống kê.
*   **AI Providers:** Hỗ trợ đa dạng nguồn cấp AI từ OpenAI, Azure OpenAI, Ollama (chạy cục bộ) đến các dịch vụ tương thích OpenAI API như DeepSeek.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của Paperless-AI được xây dựng theo hướng **"AI-First Extension"**:

*   **Mô hình RAG (Retrieval-Augmented Generation):** Thay vì chỉ gửi toàn bộ văn bản cho AI (gây tốn kém token và giới hạn ngữ cảnh), hệ thống chia nhỏ tài liệu, đánh chỉ mục vào ChromaDB. Khi người dùng hỏi, nó chỉ tìm những đoạn văn bản liên quan nhất để gửi cho AI làm dữ liệu nền (context).
*   **Kiến trúc Factory (Service Factory):** Tệp `aiServiceFactory.js` cho phép hệ thống chuyển đổi linh hoạt giữa các nhà cung cấp AI (OpenAI, Ollama, Azure) mà không làm thay đổi logic xử lý tài liệu chính.
*   **Xử lý bất đồng bộ:** Sử dụng `node-cron` để quét tài liệu mới theo chu kỳ và hàng đợi (`documentQueue`) để xử lý lần lượt, tránh làm quá tải API hoặc tài nguyên hệ thống.
*   **Thiết kế Hardened Docker:** Sử dụng `PUID/PGID` để quản lý quyền hạn tệp tin và các biện pháp bảo mật như `cap_drop: ALL` trong tệp docker-compose để tăng tính an toàn cho appliance.

### 3. Các kỹ thuật chính (Key Techniques)
*   **Hybrid Search:** Kết hợp giữa tìm kiếm từ khóa truyền thống (BM25) và tìm kiếm vector (Semantic Search) trong `main.py` giúp tăng độ chính xác khi truy xuất tài liệu.
*   **Prompt Engineering & Placeholder:** Sử dụng tệp `restrictionPromptService.js` để tự động thay thế các nhãn (tags) và người gửi (correspondents) hiện có vào prompt, giúp AI "học" được cấu trúc dữ liệu cũ của người dùng.
*   **Token Management:** Tệp `serviceUtils.js` chứa các hàm tính toán và cắt gọt văn bản (truncate) dựa trên `tiktoken` để đảm bảo dữ liệu gửi đi không vượt quá giới hạn của mô hình AI.
*   **Thumbnail Caching:** Hệ thống tự động lấy và lưu trữ ảnh thu nhỏ từ Paperless-ngx vào thư mục cục bộ để hiển thị nhanh chóng trên dashboard mà không cần truy vấn API liên tục.
*   **Custom Fields Extraction:** Kỹ thuật trích xuất thông tin đặc thù (như số hóa đơn, ngày đến hạn) và tự động tạo/cập nhật các trường dữ liệu tùy chỉnh trong Paperless-ngx.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

Hệ thống vận hành theo quy trình khép kín sau:

1.  **Quét tài liệu (Scan):** Theo lịch Cron hoặc Webhook, hệ thống kiểm tra các tài liệu mới trong Paperless-ngx chưa được AI xử lý.
2.  **Trích xuất & Chỉ mục:**
    *   Node.js tải nội dung văn bản.
    *   Dịch vụ Python (RAG) thực hiện nhúng (embedding) văn bản và lưu vào cơ sở dữ liệu vector (ChromaDB).
3.  **Phân tích AI:**
    *   Hệ thống gửi văn bản tài liệu + danh sách Tags/Correspondents hiện có + System Prompt cho AI.
    *   AI trả về kết quả định dạng JSON bao gồm: tiêu đề đề xuất, người gửi, ngày tháng và các nhãn phù hợp.
4.  **Cập nhật:** Node.js nhận kết quả, gọi API của Paperless-ngx để cập nhật thông tin tài liệu. Lưu dữ liệu gốc vào SQLite để có thể "Reset" nếu cần.
5.  **Tương tác (Chat):** Khi người dùng đặt câu hỏi (vừa tìm kiếm vừa hỏi đáp), hệ thống thực hiện:
    *   Tìm kiếm các đoạn văn bản liên quan nhất từ chỉ mục RAG.
    *   Gửi câu hỏi + các đoạn văn bản liên quan cho AI.
    *   AI trả lời dựa trên nội dung tài liệu thực tế của người dùng.

### Tổng kết
Paperless-AI là một giải pháp mở rộng thông minh, biến một kho lưu trữ tài liệu tĩnh (Paperless-ngx) thành một hệ thống **quản trị tri thức động**. Sự kết hợp giữa tìm kiếm ngữ nghĩa và khả năng phân tích của LLM giúp người dùng quản lý tài liệu cá nhân/doanh nghiệp hiệu quả hơn gấp nhiều lần so với cách gán nhãn thủ công.