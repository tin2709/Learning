Dựa trên tài liệu và mã nguồn của dự án **Understand Anything**, dưới đây là phân tích chuyên sâu về công nghệ, kiến trúc và kỹ thuật lập trình của hệ thống này:

### 1. Công nghệ cốt lõi (Core Tech Stack)

Dự án là sự kết hợp giữa sức mạnh lập luận của LLM và tính chính xác của phân tích tĩnh (Static Analysis):

*   **Phân tích tĩnh (Static Analysis):** Sử dụng `web-tree-sitter` (WASM). Đây là lựa chọn cực kỳ thông minh vì nó cho phép phân tích cú pháp code (AST - Abstract Syntax Tree) của nhiều ngôn ngữ (TS, JS, Python, Go...) ngay trong môi trường plugin mà không cần cài đặt compiler phức tạp.
*   **LLM Orchestration:** Sử dụng các mô hình của Anthropic (Claude 3.5 Sonnet cho tác vụ quét, Claude 3 Opus cho tác vụ phân tích sâu). Hệ thống không chỉ dùng AI để "đọc" mà dùng AI để "điều phối" (Multi-agent).
*   **Knowledge Graph & Data Schema:** Dữ liệu được mô hình hóa dưới dạng Đồ thị tri thức (Nodes & Edges). Sử dụng **Zod** để định nghĩa và validate schema nghiêm ngặt, đảm bảo tính toàn vẹn dữ liệu giữa các Agent và Dashboard.
*   **Visualization:** **React Flow** kết hợp với **Dagre** (đồ thị hướng dọc/ngang tự động). Kỹ thuật này giúp biến dữ liệu JSON khô khan thành một bản đồ kiến trúc có thể tương tác.
*   **Search Engine:** Kết hợp giữa **Fuse.js** (Fuzzy search dựa trên văn bản) và tiềm năng **Vector Embedding** (Semantic search) để tìm kiếm theo ý nghĩa (ví dụ: "chỗ nào xử lý auth?").

### 2. Tư duy Kiến trúc (Architecture Thinking)

Kiến trúc của dự án thể hiện tư duy hệ thống rất cao, tập trung vào tính **Portability (Khả năng di động)** và **Scalability (Khả năng mở rộng)**:

*   **Multi-Agent Pipeline (Pipeline đa tác nhân):** Thay vì dùng 1 Prompt khổng lồ, tác giả chia quy trình phân tích thành 5 giai đoạn (Phases) với các Agent chuyên biệt:
    1.  `project-scanner`: Lập danh mục file.
    2.  `file-analyzer`: Phân tích sâu từng batch file (chạy song song - parallel).
    3.  `architecture-analyzer`: Nhận diện các lớp kiến trúc (API, Service, Data).
    4.  `tour-builder`: Tạo lộ trình học code tự động.
    5.  `graph-reviewer`: Kiểm tra lỗi logic của đồ thị.
*   **Incremental Analysis (Phân tích gia tăng):** Hệ thống sử dụng Git Commit Hash để xác định các file đã thay đổi. Tư duy này giúp tiết kiệm Token LLM và giảm thời gian chờ đợi — chỉ phân tích lại những gì đã sửa.
*   **Monorepo Strategy:** Sử dụng `pnpm workspaces` để tách biệt `@understand-anything/core` (logic xử lý), `@understand-anything/skill` (giao diện CLI) và `@understand-anything/dashboard` (giao diện web). Điều này giúp Dashboard có thể chạy độc lập hoặc tích hợp vào các IDE khác nhau.
*   **Platform Agnostic (Không phụ thuộc nền tảng):** Dự án hỗ trợ cả Claude Code, Cursor, Codex, OpenClaw... thông qua cơ chế "AI-driven installation" (AI tự đọc file INSTALL.md để tự cài đặt).

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **AST Manipulation (Thao tác cây cú pháp):** Trong `tree-sitter-plugin.ts`, tác giả triển khai các hàm `traverse` để duyệt cây AST. Đây là kỹ thuật lập trình hệ thống cao cấp, cho phép trích xuất chính xác tên hàm, tham số, kiểu trả về và các mối quan hệ `imports/exports` mà không cần thực thi code.
*   **Parallel Processing (Xử lý song song):** Trong `SKILL.md`, lệnh `/understand` thực hiện dispatch các sub-agents chạy song song (tối đa 3 luồng). Điều này tối ưu hóa IO và thời gian phản hồi của LLM.
*   **Context Injection (Bơm ngữ cảnh):** Tác giả sử dụng kỹ thuật trích xuất 3000 ký tự đầu của README và file manifest (`package.json`, `go.mod`...) để làm "Grounding" cho AI. Điều này giúp AI hiểu dự án làm gì trước khi bắt đầu đọc từng file code lẻ tẻ.
*   **State Management (Zustand):** Dashboard sử dụng Zustand để quản lý trạng thái đồ thị phức tạp (chọn node, highlight đường đi, tour mode) một cách gọn nhẹ, tránh boilerplate của Redux.

### 4. Luồng hoạt động hệ thống (System Flow)

Quy trình hoạt động được thiết kế như một nhà máy sản xuất tri thức:

1.  **Input:** User nhập `/understand`. Plugin thu thập metadata (Git hash, README).
2.  **Analysis Phase:**
    *   `Tree-sitter` quét toàn bộ codebase để tạo cấu trúc khung (Structural Analysis).
    *   LLM nhận khung đó + nội dung file để viết tóm tắt (Semantic Analysis).
3.  **Synthesis Phase:** `GraphBuilder` tổng hợp kết quả thành file `knowledge-graph.json`. Các cạnh (Edges) được tạo ra dựa trên quan hệ `calls`, `imports`, `depends_on`.
4.  **Enrichment Phase:** AI phân tích đồ thị để tìm ra các "Layers" (tư duy kiến trúc) và tạo "Guided Tour" (tư duy giáo dục).
5.  **Output:** Dashboard React Flow render dữ liệu. Người dùng click vào Node để xem code (Code Viewer) hoặc chat với AI về Node đó (`understand-chat`).

### Tổng kết
**Understand Anything** không đơn thuần là một công cụ tóm tắt code. Nó là một **Search Engine chuyên biệt cho Kiến trúc phần mềm**, sử dụng AST làm xương sống (độ chính xác) và LLM làm bộ não (khả năng tóm tắt) để giải quyết bài toán "Cognitive Load" (quá tải nhận thức) khi tiếp cận dự án lớn.