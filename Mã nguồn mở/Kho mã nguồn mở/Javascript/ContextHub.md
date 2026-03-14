Dựa trên mã nguồn và cấu trúc của dự án **Context Hub (chub)**, đây là bản phân tích chi tiết về công nghệ, kiến trúc và cách thức vận hành:

### 1. Công nghệ cốt lõi (Core Technologies)
Dự án được xây dựng chủ yếu bằng **JavaScript (Node.js)** với tư duy tối giản và tập trung vào hiệu suất cho AI Agent:

*   **Runtime:** Node.js (Yêu cầu >= 18).
*   **CLI Framework:** `commander` (Dùng để xây dựng giao diện dòng lệnh mạnh mẽ).
*   **Giao thức AI:** `@modelcontextprotocol/sdk` (Triển khai **Model Context Protocol - MCP**), giúp các AI như Claude Code hoặc Cursor có thể kết nối trực tiếp với công cụ này như một "tool" bản địa.
*   **Xử lý dữ liệu:** 
    *   `yaml`: Để phân tích (parse) frontmatter trong các file Markdown.
    *   `zod`: Kiểm tra tính hợp lệ (validation) của dữ liệu (đặc biệt trong MCP server).
    *   `bm25`: Thuật toán tìm kiếm văn bản (Probabilistic ranking function) được cài đặt trực tiếp bằng JS để tìm kiếm doc hiệu quả mà không cần database nặng nề.
*   **Dữ liệu (Content):** Markdown thuần túy kèm YAML frontmatter, giúp cả người và máy đều dễ đọc/sửa.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của Context Hub tuân theo triết lý **"Agent-First"** (Ưu tiên Agent):

*   **Decoupled Content (Tách biệt nội dung):** Mã nguồn CLI (`cli/`) và Nội dung tài liệu (`content/`) hoàn toàn độc lập. Điều này cho phép cộng đồng đóng góp tài liệu mà không cần biết lập trình Node.js.
*   **Local-First with Remote Sync:** Tài liệu được tải về và lưu vào cache cục bộ (`~/.chub`). Điều này giúp Agent truy cập tài liệu tức thì, không phụ thuộc vào độ trễ của mạng trong quá trình sinh code.
*   **Incremental Disclosure (Tiết lộ dần dần):** Kiến trúc hỗ trợ fetch file chính (`DOC.md`) trước, và chỉ fetch thêm các reference sâu hơn khi Agent yêu cầu. Kỹ thuật này giúp tiết kiệm "Context Window" (vốn đắt đỏ) của LLM.
*   **Multi-source Registry:** Hỗ trợ nhiều nguồn tài liệu (Official, Internal, Community). Hệ thống có cơ chế gộp (merge) các registry từ nhiều URL hoặc đường dẫn local khác nhau.

### 3. Các kỹ thuật chính (Key Techniques)
*   **BM25 Search Indexing:** Khi chạy lệnh `build`, hệ thống tạo ra một file `search-index.json`. File này chứa các token được tính toán trọng số theo thuật toán BM25, cho phép tìm kiếm mờ (fuzzy search) cực nhanh trên terminal.
*   **Local Annotations (Ghi chú cục bộ):** Sử dụng hệ thống lưu trữ file tại `~/.chub/annotations/`. Kỹ thuật này giúp Agent "ghi nhớ" kinh nghiệm từ phiên làm việc trước (ví dụ: "API này có bug ở phiên bản X") và tự động chèn vào nội dung tài liệu ở lần fetch sau.
*   **JSON Everywhere:** Mọi lệnh CLI đều có flag `--json`. Đây là kỹ thuật quan trọng để Agent có thể "pipe" kết quả vào các công cụ xử lý khác (như `jq`) một cách lập trình được.
*   **Telemetry & Identity:** Hệ thống tạo một Client ID ẩn danh dựa trên mã băm (hash) của UUID máy tính (ioreg trên Mac, machine-id trên Linux) để theo dõi phản hồi tài liệu mà không xâm phạm quyền riêng tư.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng hoạt động điển hình của một AI Agent khi sử dụng Context Hub:

1.  **Cập nhật (Update):** Người dùng hoặc Agent chạy `chub update` để tải danh mục tài liệu (Registry) từ CDN về máy.
2.  **Khám phá (Search):** Agent thực hiện lệnh `chub search "stripe payments"`. CLI sử dụng thuật toán BM25 trên index cục bộ để trả về ID chính xác (ví dụ: `stripe/api`).
3.  **Truy xuất (Get):** Agent gọi `chub get stripe/api --lang js`. 
    *   Hệ thống kiểm tra cache.
    *   Nếu không có, nó tải từ CDN.
    *   Đồng thời, nó tìm trong thư mục `annotations` xem trước đó Agent đã ghi chú gì về ID này chưa.
4.  **Hợp nhất (Merge):** Nội dung tài liệu + Ghi chú cục bộ được gộp lại và trả về cho Agent làm ngữ cảnh để viết code.
5.  **Phản hồi (Feedback/Annotate):** 
    *   Nếu code chạy sai, Agent gọi `chub annotate` để lưu lại lỗi đó cho lần sau.
    *   Nếu tài liệu cũ, Agent gọi `chub feedback` để báo cho tác giả (Author) cập nhật.

**Kết luận:** Context Hub không chỉ là một kho chứa tài liệu, mà là một **"Hệ thống ghi nhớ và cải thiện ngữ cảnh"** dành riêng cho kỷ nguyên lập trình bằng AI, giúp giải quyết triệt để vấn đề Agent bị ảo giác (hallucination) do dữ liệu training cũ.