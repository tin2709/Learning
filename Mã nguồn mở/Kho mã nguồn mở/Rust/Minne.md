Dựa trên mã nguồn của dự án **Minne**, đây là phân tích chi tiết về kiến trúc hệ thống, công nghệ cốt lõi và kỹ thuật lập trình của một hệ thống quản trị tri thức cá nhân (PKM) hiện đại bằng Rust.

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

Minne là một minh chứng điển hình cho xu hướng "Modern SSR" (Server-Side Rendering hiện đại) kết hợp với sức mạnh xử lý của Rust:

*   **Ngôn ngữ chính:** **Rust** (Workspace-based), tận dụng tối đa tính an toàn bộ nhớ và hiệu suất song song.
*   **Web Framework:** **Axum** (được xây dựng trên Tower/Hyper). Đây là framework có hiệu suất cực cao và khả năng mở rộng tốt trong hệ sinh thái Rust.
*   **Frontend Engine:** **HTMX** kết hợp với **MiniJinja** (template engine). Thay vì dùng React/Vue, Minne trả về các đoạn HTML động, giúp giảm tải phía Client và tăng tốc độ phản hồi.
*   **Cơ sở dữ liệu (Trái tim của hệ thống):** **SurrealDB**. Đây là lựa chọn chiến lược vì nó hỗ trợ đồng thời:
    *   *Document DB:* Lưu trữ ghi chú.
    *   *Graph DB:* Quản lý mối quan hệ giữa các thực thể tri thức (Edge/Node).
    *   *Vector DB:* Lưu trữ embeddings để tìm kiếm ngữ nghĩa (Semantic Search).
*   **AI & NLP:** 
    *   **FastEmbed-rs:** Để tạo vector embedding và reranking (xếp hạng lại) cục bộ trên CPU, giảm phụ thuộc vào API bên ngoài.
    *   **OpenAI SDK:** Tương thích với bất kỳ provider nào hỗ trợ chuẩn OpenAI (như Ollama, Anthropic, v.v.).
*   **Trình thu thập dữ liệu:** **Headless Chrome** (thông qua crate `headless_chrome`) để scrape các trang web phức tạp và chuyển đổi về dạng văn bản sạch bằng `dom_smoothie`.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Hệ thống được thiết kế theo mô hình **Modular Monolith** (Khối thống nhất phân rã thành các module):

*   **Cấu trúc Crate (Workspace):** Dự án chia làm nhiều crate chuyên biệt:
    *   `api-router`: Xử lý các request từ mobile/iOS Shortcut.
    *   `html-router`: Xử lý giao diện Web SSR.
    *   `ingestion-pipeline`: Luồng xử lý dữ liệu đầu vào (Scraping, OCR, LLM mapping).
    *   `retrieval-pipeline`: Luồng truy xuất dữ liệu (Hybrid search, Graph traversal).
    *   `common`: Chứa các schema database và logic dùng chung.
*   **Kiến trúc Phân tách Server/Worker:** Minne có thể chạy ở 3 chế độ: `main` (tất cả trong một), `server` (chỉ web), hoặc `worker` (chỉ xử lý tác vụ nền). Điều này giúp tối ưu hóa tài nguyên trên các thiết bị yếu như NAS hoặc Raspberry Pi.
*   **Hybrid RAG (Retrieval-Augmented Generation):** Thay vì chỉ tìm kiếm vector đơn thuần, kiến trúc của Minne kết hợp: **Full-text search + Vector Similarity + Graph Traversal**. Nó tìm các ghi chú liên quan, sau đó đi theo các "cạnh" (edges) trong đồ thị để lấy thêm ngữ nghĩa xung quanh.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **State Machine cho Ingestion:** Trong module `ingestion-pipeline`, dự án sử dụng mô hình máy trạng thái (`Pending` -> `Processing` -> `Succeeded`/`Failed`) để quản lý các tác vụ xử lý nội dung nặng nề.
*   **Macro-based stored objects:** Sử dụng macro `stored_object!` để tự động hóa việc triển khai các trait CRUD (Create, Read, Update, Delete) cho các cấu trúc dữ liệu, giảm thiểu code lặp lại (boilerplate).
*   **Server-Sent Events (SSE) cho AI Chat:** Module `message_response_stream.rs` triển khai stream phản hồi từ LLM trực tiếp đến giao diện thông qua SSE, kết hợp với HTMX để hiển thị chữ chạy thời gian thực (typing effect) mà không cần WebSocket phức tạp.
*   **Middleware Patterns:** Sử dụng `Tower Layers` để xử lý các vấn đề cắt ngang (cross-cutting concerns) như:
    *   Tự động tính toán Analytics (page loads/visitors).
    *   Quản lý Session dựa trên SurrealDB.
    *   Nén dữ liệu (Compression).
*   **Local-first Embeddings:** Kỹ thuật sử dụng **ONNX Runtime** để chạy các mô hình embedding ngay trên thiết bị người dùng, giúp bảo vệ quyền riêng tư và tăng tốc độ tìm kiếm dưới 50ms.

---

### 4. Luồng hoạt động hệ thống (System Workflows)

#### A. Luồng Nạp dữ liệu (Ingestion Workflow)
1.  **Ingress:** Người dùng gửi URL, File (PDF/Audio/Image) hoặc Text qua Web hoặc API.
2.  **Task Creation:** Một `ingestion_task` được tạo trong SurrealDB.
3.  **Extraction:** Worker lấy task, nếu là URL thì dùng Headless Chrome để render, nếu là Audio thì dùng Whisper để transcribe.
4.  **LLM Enrichment:** Sử dụng LLM để trích xuất các thực thể tri thức (Entities) và các mối quan hệ (Relationships) dưới dạng JSON cấu trúc.
5.  **Graph & Vector Mapping:** Các thực thể được lưu thành các Node, mối quan hệ thành Edge. Nội dung được chia nhỏ (chunking) và tạo vector embedding lưu vào bảng chuyên dụng.

#### B. Luồng Truy vấn tri thức (Retrieval/Chat Workflow)
1.  **Query Analysis:** LLM phân tích câu hỏi của người dùng.
2.  **Hybrid Search:** Hệ thống thực hiện tìm kiếm từ khóa (FTS) và tìm kiếm không gian vector đồng thời trên SurrealDB.
3.  **Graph Expansion:** Từ các kết quả tìm được, hệ thống truy vấn các node lân cận để lấy thêm bối cảnh.
4.  **Reranking:** (Nếu bật) FastEmbed sẽ sắp xếp lại các đoạn văn bản theo độ phù hợp thực tế với câu hỏi.
5.  **Generation:** Gửi context đã tối ưu vào LLM để tạo câu trả lời cuối cùng và stream về UI.

### Tóm tắt đặc điểm nổi bật
Minne không chỉ là một ứng dụng lưu trữ, mà là một **AI-Native Graph PKM**. Nó giải quyết bài toán khó nhất của việc ghi chú: **"Tự động liên kết tri thức mà không làm mất đi quyền kiểm soát thủ công của người dùng"**. Việc chọn Rust và SurrealDB cho thấy dự án hướng tới sự bền vững về hiệu suất và khả năng xử lý dữ liệu phức tạp lâu dài.