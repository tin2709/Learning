Dựa trên mã nguồn của dự án **Corbell**, dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống này:

### 1. Công nghệ cốt lõi (Core Technology)

Corbell là một công cụ "Local-first" (ưu tiên chạy tại chỗ) kết hợp giữa Static Analysis (Phân tích tĩnh) và AI để quản lý kiến trúc phần mềm.

*   **Ngôn ngữ & Runtime:** Python 3.11+, sử dụng `Typer` cho CLI và `Pydantic v2` để quản lý dữ liệu/config.
*   **Phân tích mã nguồn (AST Parsing):** 
    *   **Tree-sitter:** Sử dụng các thư viện `tree-sitter` (Python, JS, TS, Go, Java) để trích xuất chữ ký hàm (method signatures) và cấu trúc code một cách chính xác thay vì chỉ dùng Regex.
    *   **Python AST:** Sử dụng module `ast` có sẵn của Python làm phương án dự phòng.
*   **Vector Search & Embeddings:**
    *   **Sentence-Transformers:** Sử dụng model `all-MiniLM-L6-v2` chạy local để chuyển code thành vector.
    *   **SQLite (với NumPy):** Sử dụng SQLite làm database lưu trữ cả Graph (quan hệ) và Embeddings (dưới dạng BLOB), tính toán Cosine Similarity bằng NumPy.
*   **LLM Integration:** Hỗ trợ đa dạng provider (Anthropic, OpenAI, AWS Bedrock, Azure, GCP, và Ollama cho local LLM) thông qua một lớp `LLMClient` thống nhất.
*   **Giao diện & Visualization:** D3.js được sử dụng để vẽ đồ thị quan hệ (force-directed graph) trong giao diện web đơn giản chạy bằng `http.server` của Python.
*   **Model Context Protocol (MCP):** Tích hợp giao thức MCP của Anthropic, cho phép các AI Agent (như Cursor, Claude Desktop) truy cập trực tiếp vào context kiến trúc của dự án.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Corbell phản ánh tư duy **"Architecture as Code"** và **"Living Documentation"**:

*   **Graph-based Knowledge:** Hệ thống coi toàn bộ hệ sinh thái multi-repo là một đồ thị tri thức (Knowledge Graph). Các Node không chỉ là service mà còn là Method, Database, Queue, và Flow (luồng thực thi).
*   **Context-Injected Generation:** Thay vì yêu cầu AI tự đoán, Corbell trích xuất các "Skeletal context" (khung xương mã nguồn) từ đồ thị để "bơm" vào Prompt. Điều này giúp AI tạo ra thiết kế (Spec) tuân thủ đúng pattern hiện tại của đội ngũ.
*   **Heuristic-driven Discovery:** Sử dụng các tập luật (patterns) để tự động phát hiện infrastructure (AWS CDK, Terraform) và các mối quan hệ ẩn (ví dụ: một service gọi một service khác thông qua env var hoặc RPC).
*   **Decoupling (Sự tách biệt):** Toàn bộ logic lõi (`corbell.core`) được tách biệt hoàn toàn khỏi giao diện dòng lệnh (`corbell.cli`), cho phép tích hợp vào UI hoặc MCP Server dễ dàng.

### 3. Kỹ thuật lập trình chính (Primary Programming Techniques)

*   **Visitor Pattern:** Được sử dụng rộng rãi trong `method_graph.py` và `extractor.py` để duyệt qua các cây cú pháp trừu tượng (AST) của nhiều ngôn ngữ khác nhau nhằm tìm kiếm hàm, lớp và lời gọi hàm.
*   **Provider/Factory Pattern:** Lớp `LLMClient` và `EmbeddingStore` áp dụng pattern này để hỗ trợ nhiều backend khác nhau (ví dụ: chuyển đổi giữa OpenAI và Anthropic chỉ bằng cấu hình YAML).
*   **Static & Semantic Hybrid Search:** Kết hợp tìm kiếm theo từ khóa (Graph query) và tìm kiếm ngữ nghĩa (Vector search) để tự động phát hiện các service liên quan đến một yêu cầu tính năng (PRD).
*   **Graph Traversal (BFS):** Sử dụng thuật toán Breadth-First Search trong `flow_tracer.py` để tìm đường đi giữa các phương thức (Call path) và xác định các luồng thực thi từ Entry-point (ví dụ: từ một API Controller đi sâu vào logic bên dưới).
*   **Git Coupling Analysis:** Kỹ thuật phân tích lịch sử commit (`git_coupling.py`) để tìm ra các file thường xuyên thay đổi cùng nhau, từ đó cảnh báo các "mối quan hệ ngầm" mà phân tích code tĩnh không thấy được.

### 4. Luồng hoạt động hệ thống (System Operational Flow)

Luồng hoạt động của Corbell chia làm 3 giai đoạn chính:

#### Giai đoạn 1: Xây dựng tri thức (Ingestion)
1.  **`graph build`:** Quét các repo, dùng Tree-sitter trích xuất cây lời gọi hàm (Call Graph). Quét IaC (Terraform/CDK) để tìm DB, Queue.
2.  **`embeddings build`:** Chia nhỏ code thành các chunk và lưu vào Vector DB local.
3.  **`docs learn`:** Quét các tài liệu ADR/RFC cũ để học phong cách thiết kế của team.

#### Giai đoạn 2: Tạo thiết kế (Generation)
1.  Người dùng nhập một PRD (yêu cầu tính năng).
2.  **PRD Processor:** Chuyển PRD thành các câu truy vấn code.
3.  **Discovery:** Tìm kiếm trong Vector DB để xác định những Service nào sẽ bị ảnh hưởng.
4.  **Prompt Construction:** Lấy chữ ký hàm và cấu trúc đồ thị của các Service đó, kết hợp với các ràng buộc (Constraints) kiến trúc, tạo thành một Prompt khổng lồ gửi cho LLM.
5.  **LLM:** Viết ra một file Markdown thiết kế chi tiết (Spec).

#### Giai đoạn 3: Kiểm soát và Triển khai (Review & Decompose)
1.  **`spec review`:** AI kiểm tra file thiết kế đối chiếu với đồ thị thực tế để phát hiện sai sót.
2.  **`spec decompose`:** Chia nhỏ thiết kế thành các task kỹ thuật dưới dạng YAML.
3.  **`export linear`:** Đẩy các task này lên công cụ quản lý dự án (Linear) kèm theo đầy đủ context về code cho lập trình viên.

### Tóm lại
Corbell không chỉ là một trình tạo code, nó là một **"Architecture Intelligence Layer"** (Lớp trí tuệ kiến trúc). Nó giải quyết bài toán khó nhất trong microservices: **Sự mất kết nối giữa tài liệu thiết kế và mã nguồn thực tế.** Bằng cách chạy hoàn toàn local, nó đảm bảo tính bảo mật mã nguồn cho doanh nghiệp.