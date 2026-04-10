Dựa trên mã nguồn và tài liệu kỹ thuật của dự án **MemPalace**, dưới đây là phân tích chi tiết về hệ thống bộ nhớ AI này:

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ lập trình:** **Python 3.9+** là ngôn ngữ chủ đạo (chiếm 99%).
*   **Cơ sở dữ liệu Vector:** **ChromaDB** được sử dụng để lưu trữ các "drawers" (ngăn kéo) chứa văn bản thô và thực hiện tìm kiếm ngữ nghĩa (semantic search).
*   **Cơ sở dữ liệu quan hệ:** **SQLite** đóng vai trò là "xương sống" cho **Knowledge Graph** (Đồ thị tri thức), lưu trữ các thực thể và mối quan hệ có yếu tố thời gian.
*   **Giao thức kết nối:** **MCP (Model Context Protocol)** của Anthropic, cho phép MemPalace tích hợp trực tiếp như một server cung cấp công cụ (tools) cho các AI như Claude Code, ChatGPT, Cursor.
*   **Ngôn ngữ nén AAAK (experimental):** Một phương ngữ viết tắt (lossy abbreviation dialect) do dự án tự phát triển, giúp nén thông tin lặp lại thành ít token hơn nhưng vẫn đảm bảo các mô hình ngôn ngữ lớn (LLM) đọc hiểu được mà không cần bộ giải mã.
*   **Xử lý văn bản:** Sử dụng regex và các thuật toán heuristic để phân loại văn bản (không dùng LLM để tiết kiệm chi phí và tăng tốc độ).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của MemPalace được xây dựng dựa trên phương pháp "Memory Palace" (Cung điện ký ức) của người Hy Lạp cổ đại, tổ chức dữ liệu theo sơ đồ không gian:

*   **Cấu trúc phân cấp (The Palace Hierarchy):**
    *   **Wings (Cánh):** Đại diện cho một dự án hoặc một người (ví dụ: `wing_myapp`).
    *   **Halls (Sảnh):** Các loại bộ nhớ dùng chung (sự thật, sự kiện, khám phá, sở thích).
    *   **Rooms (Phòng):** Các chủ đề cụ thể trong một Wing (ví dụ: `auth-migration`).
    *   **Closets (Tủ):** Các bản tóm tắt hoặc từ khóa chỉ dẫn.
    *   **Drawers (Ngăn kéo):** Nơi lưu trữ văn bản gốc (verbatim), đảm bảo không mất dữ liệu do tóm tắt sai.
*   **Nguyên tắc "Verbatim First":** Khác với các hệ thống khác tóm tắt rồi mới lưu, MemPalace lưu mọi từ ngữ gốc và dùng tìm kiếm ngữ nghĩa để truy xuất, giúp giữ trọn vẹn ngữ cảnh.
*   **Chồng bộ nhớ 4 lớp (Memory Stack Layers):**
    *   **L0 (Identity):** AI là ai? (Luôn tải).
    *   **L1 (Critical Facts):** Các sự thật quan trọng về thế giới của người dùng (~170 tokens, luôn tải).
    *   **L2 (Room Recall):** Truy xuất thông tin khi chủ đề liên quan xuất hiện.
    *   **L3 (Deep Search):** Tìm kiếm sâu trên toàn bộ kho dữ liệu khi có yêu cầu cụ thể.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Heuristic Classification (`general_extractor.py`):** Sử dụng các bộ quy tắc regex (markers) cực kỳ chi tiết để phân loại nội dung thành 5 nhóm: Quyết định (Decisions), Sở thích (Preferences), Cột mốc (Milestones), Vấn đề (Problems), và Cảm xúc (Emotional) mà không cần gọi API của LLM.
*   **Hybrid Retrieval:** Kết hợp giữa tìm kiếm Vector của ChromaDB với việc tăng điểm (boost) dựa trên từ khóa (keyword overlap) và khoảng cách thời gian (temporal boost).
*   **Temporal Knowledge Graph:** Sử dụng bộ ba `Subject - Predicate - Object` kèm theo thuộc tính `valid_from` và `valid_to` để quản lý các sự thật thay đổi theo thời gian (ví dụ: "Người dùng đang làm dự án A" sẽ bị vô hiệu hóa khi họ chuyển sang dự án B).
*   **Format Normalization:** Module `normalize.py` có khả năng tự động nhận diện và chuyển đổi 5 định dạng chat phổ biến (Claude, ChatGPT, Slack, JSONL, Plain text) về một chuẩn chung.
*   **Auto-Save Hooks:** Sử dụng Bash script kết hợp với cơ chế Hook của IDE/CLI để tự động kích hoạt việc lưu bộ nhớ sau mỗi 15 tin nhắn hoặc trước khi ngữ cảnh bị tràn (compact).

### 4. Luồng hoạt động hệ thống (System Operation Flows)

#### A. Luồng Nhập liệu (Ingest/Mining Flow):
1.  **Khởi tạo (`init`):** Quét cấu trúc thư mục của người dùng để tự động nhận diện các "Rooms".
2.  **Khai thác (`mine`):**
    *   Đọc file -> Chuẩn hóa định dạng -> Chia nhỏ văn bản (Chunking).
    *   Dùng `Entity Detector` để phát hiện người và dự án.
    *   Ghi vào ChromaDB (Vector) và SQLite (Knowledge Graph).
3.  **Deduplication:** Kiểm tra độ tương đồng (threshold ~0.9) để tránh lưu trùng lặp dữ liệu.

#### B. Luồng Truy xuất (Retrieval Flow - "Wake-up"):
1.  Khi AI khởi động, lệnh `wake-up` sẽ được gọi.
2.  Hệ thống lấy L0 (Identity) và L1 (Critical Facts) đã được nén bằng AAAK để đưa vào Prompt hệ thống của AI.
3.  AI có cái nhìn tổng quan về "thế giới" của người dùng ngay lập tức.

#### C. Luồng Tìm kiếm & Tương tác (Search & Interaction):
1.  AI gọi tool `mempalace_search` thông qua giao thức MCP.
2.  Hệ thống thực hiện tìm kiếm Vector trong Wing hoặc Room cụ thể.
3.  Kết quả trả về văn bản gốc kèm theo điểm tin cậy để AI trả lời người dùng.

#### D. Luồng Cập nhật Sự thật (Fact Update):
1.  Nếu AI phát hiện một thông tin cũ không còn đúng (ví dụ: thay đổi công nghệ từ MySQL sang Postgres).
2.  Gọi `kg_invalidate` để đóng mốc thời gian của sự thật cũ.
3.  Gọi `kg_add` để thêm sự thật mới vào đồ thị tri thức.

### Tóm lại
MemPalace là một hệ thống **Local-first RAG (Retrieval-Augmented Generation)** tối ưu hóa cho bộ nhớ dài hạn của AI. Nó kết hợp sự mạnh mẽ của cơ sở dữ liệu Vector với tính cấu trúc chặt chẽ của đồ thị tri thức truyền thống, giúp AI không chỉ "tìm thấy" mà còn "hiểu" được bối cảnh công việc và cuộc sống của người dùng một cách nhất quán.