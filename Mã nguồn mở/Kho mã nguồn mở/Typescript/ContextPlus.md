Dựa trên toàn bộ mã nguồn và tài liệu bạn cung cấp, đây là bản phân tích chuyên sâu về hệ thống **Context+** theo 4 trụ cột chính:

---

### 1. Công nghệ cốt lõi (Core Tech Stack)
Context+ không chỉ là một wrapper đơn giản, nó kết hợp nhiều kỹ thuật xử lý ngôn ngữ và phân tích cấu trúc:

*   **Model Context Protocol (MCP):** Sử dụng SDK của Anthropic để xây dựng giao diện kết nối giữa AI Agent (Claude, Cursor) và hệ thống file cục bộ qua chuẩn `stdio`.
*   **Tree-sitter AST (Abstract Syntax Tree):** Sử dụng `web-tree-sitter` kết hợp với các tệp WASM của 43 ngôn ngữ. Thay vì dùng Regex (thường sai số cao), hệ thống phân tích cú pháp thực sự của mã nguồn để trích xuất Function, Class, Method với độ chính xác tuyệt đối về dòng (line numbers).
*   **Ollama (Local AI):** Tận dụng Ollama làm backend cho Embedding (`nomic-embed-text`) và Chat (`gemma2`, `llama3.2`). Điều này đảm bảo tính riêng tư (mã nguồn không gửi lên cloud) và tiết kiệm chi phí.
*   **Spectral Clustering (Phân cụm phổ):** Sử dụng thư viện `ml-matrix` để thực hiện thuật toán phân cụm dựa trên trị riêng (Eigenvalues). Đây là kỹ thuật cao cấp để nhóm các file có liên quan về mặt ngữ nghĩa (semantic) lại với nhau ngay cả khi chúng nằm ở các thư mục khác nhau.
*   **Vector Database (Cơ chế đệm):** Không sử dụng DB bên ngoài, Context+ tự xây dựng hệ thống cache vector dựa trên tệp JSON trong thư mục `.mcp_data`, tối ưu hóa việc truy vấn tương tự cosine mà không cần overhead của một DB full-scale.

### 2. Tư duy Kiến trúc (Architectural Design)
Kiến trúc của Context+ được thiết kế để giải quyết vấn đề "Tràn ngữ cảnh" (Context Bloat) của LLM:

*   **Cơ chế Pruning (Tỉa cành) đa cấp:** 
    *   *Level 2:* Hiển thị đầy đủ symbol.
    *   *Level 1:* Chỉ hiển thị file header.
    *   *Level 0:* Chỉ hiển thị tên file.
    Hệ thống tự động tính toán số lượng token và hạ cấp độ chi tiết để đảm bảo luôn vừa vặn với context window của AI.
*   **Kiến trúc 3 lớp (Laminar Architecture):**
    1.  **Core Layer:** Xử lý việc đọc file, trích xuất AST, tính toán vector.
    2.  **Tools Layer:** 17 công cụ MCP cụ thể thực thi các tác vụ (Search, Analysis, Commit).
    3.  **Git/Shadow Layer:** Một hệ thống quản lý phiên bản "vô hình" tạo các restore point trước khi AI thay đổi code, tách biệt hoàn toàn với Git history của người dùng.
*   **Hệ thống Memory Graph (RAG):** Thay vì chỉ search text đơn thuần, Context+ xây dựng một đồ thị thuộc tính (Property Graph). Các node (Concept, Symbol, File) liên kết với nhau bằng các cạnh có trọng số giảm dần theo thời gian (Decay Scoring - $e^{-\lambda t}$), giúp AI ưu tiên những thông tin mới và quan trọng nhất.

### 3. Kỹ thuật lập trình chính (Key Coding Techniques)
Mã nguồn thể hiện trình độ xử lý dữ liệu và hệ thống rất cao:

*   **Adaptive Embedding Input:** Trong `src/core/embeddings.ts`, hệ thống có cơ chế tự động thu nhỏ (shrink) hoặc chia nhỏ (chunk) input nếu Ollama báo lỗi quá tải context. Nó thử nghiệm đệ quy cho đến khi tìm được kích thước tối ưu (`embedBatchAdaptive`).
*   **Real-time Embedding Tracker:** Sử dụng `fs.watch` trong `src/core/embedding-tracker.ts` với cơ chế **Debounce** (trì hoãn xử lý cho đến khi người dùng ngừng gõ). Điều này giúp cập nhật vector ngay lập tức nhưng không làm nghẽn GPU/CPU khi người dùng lưu file liên tục.
*   **Hybrid Search Ranking:** Kết hợp giữa *Semantic Score* (Cosine Similarity từ Vector) và *Keyword Score* (độ phủ từ khóa). Trọng số mặc định là 78/22, giúp cân bằng giữa việc hiểu ý định và việc tìm đúng tên biến cụ thể.
*   **Shadow Backup System:** Kỹ thuật lưu trữ tệp tạm thời bằng cách mã hóa đường dẫn tệp thành tên tệp (ví dụ: `src/main.ts` thành `src__main.ts`) trong thư mục `.mcp_data/backups` để tránh xung đột cấu trúc thư mục khi sao lưu.

### 4. Luồng hoạt động hệ thống (System Workflow)
Hệ thống vận hành theo một vòng lặp khép kín nhằm tối ưu hóa hiệu suất cho Agent:

1.  **Khởi động (Startup):** Server MCP chạy, khởi tạo `EmbeddingTracker`. Nếu ở chế độ `eager`, nó sẽ quét toàn bộ project để xây dựng cache vector ngay lập tức.
2.  **Khám phá (Discovery):** AI Agent gọi `get_context_tree`. Hệ thống duyệt thư mục (tuân thủ `.gitignore`), trích xuất 2 dòng header đầu tiên của mỗi file để AI biết file đó làm gì mà không cần đọc code bên trong.
3.  **Điều hướng ngữ nghĩa (Navigation):** Khi người dùng hỏi về một tính năng, Agent dùng `semantic_navigate`. Hệ thống clustering sẽ nhóm các file "Auth", "Database", "UI" lại và dán nhãn (labeling) bằng AI để Agent định hướng vùng cần làm việc.
4.  **Phân tích tác động (Blast Radius):** Trước khi Agent sửa một hàm, nó gọi `get_blast_radius`. Hệ thống quét toàn bộ project để tìm các vị trí hàm đó được import hoặc sử dụng, ngăn chặn việc sửa đổi gây lỗi dây chuyền (breaking changes).
5.  **Ghi mã nguồn (Writing):** Agent bắt buộc phải qua `propose_commit`. Tool này đóng vai trò "Linter" khắt khe:
    *   Kiểm tra xem có đủ 2 dòng header không.
    *   Kiểm tra độ sâu lồng nhau (nesting).
    *   Cấm hoàn toàn inline comment (để giữ code sạch cho LLM).
    *   Tự động tạo Restore Point.
6.  **Kiểm tra cuối (Validation):** Gọi `run_static_analysis` để thực thi trình biên dịch thật (tsc, go vet, v.v.) nhằm xác nhận code không chỉ đúng về ngữ pháp mà còn đúng về logic kiểu dữ liệu.

### Tóm tắt tư duy "ForLoop" (Dựa trên SKILL.md)
Triết lý lập trình của dự án này là **"Minimalist for AI"**:
*   Code cực kỳ gọn (Inline các hàm <20 dòng).
*   Ưu tiên hiệu suất Token (Token-efficiency).
*   Cấu trúc file phẳng, dễ đọc cho máy.
*   Loại bỏ mọi sự dư thừa (unused variables/files).

Đây là một công cụ biến mã nguồn thành một "Database có cấu trúc" để AI có thể truy vấn thay vì chỉ đọc văn bản thuần túy.