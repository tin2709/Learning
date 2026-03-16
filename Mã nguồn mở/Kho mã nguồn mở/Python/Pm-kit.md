Dựa trên mã nguồn và cấu trúc thư mục của **PM-Kit**, đây là phân tích chi tiết về hệ thống biến AI Coding Agent thành một trợ lý Quản lý dự án (PM) chuyên nghiệp:

### 1. Công nghệ cốt lõi (Core Tech Stack)

PM-Kit không phải là một ứng dụng phần mềm truyền thống có giao diện đồ họa riêng, mà là một **"Framework tri thức dựa trên Markdown"** vận hành trên các công cụ sẵn có:

*   **Runtime Environment:** **Claude Code CLI** là "bộ não" thực thi chính. Ngoài ra còn hỗ trợ Cursor, Windsurf, Copilot thông qua hệ thống liên kết cấu hình (Symlinks).
*   **Knowledge Base:** **Markdown & Obsidian**. Dữ liệu được lưu trữ dưới dạng tệp văn bản thuần túy, có cấu trúc liên kết Wiki-links (`[[ ]]`).
*   **Search Engine:** **QMD (Vector Search)**. Sử dụng Bun và mô hình nhúng (Embeddings) để thực hiện tìm kiếm ngữ nghĩa (Semantic search) thay vì chỉ tìm kiếm từ khóa đơn thuần.
*   **Automation & Scripting:**
    *   **Bash Scripts:** Điều khiển luồng thiết lập (setup), cập nhật (update) và các hooks hệ thống.
    *   **Python:** Xử lý logic phức tạp như Recalc (tính toán lại Excel), xử lý PDF.
    *   **Node.js:** Tạo và chỉnh sửa các định dạng tài liệu Office (Word, PowerPoint).
*   **Data Structure:** **YAML** (cho cấu hình `config.yaml` và Frontmatter trong mỗi ghi chú).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của PM-Kit được xây dựng trên triết lý **"Văn bản là API" (Naming-as-API)**:

*   **Tách biệt Framework và Dữ liệu:** Các file hệ thống (kỹ năng, agent, script) nằm trong `.claude/`, `_core/`, trong khi dữ liệu người dùng nằm ở các thư mục như `daily/`, `docs/`, `decisions/`. Điều này cho phép cập nhật framework mà không làm mất dữ liệu.
*   **Mô hình 2 lớp (Two-Layer Design):**
    *   **Skills (Giao diện lệnh):** Các lệnh `/command` định nghĩa I-P-O (Input-Process-Output) và quyền hạn công cụ.
    *   **Agents (Logic thực thi):** Các agent chuyên biệt (Scribe - Ghi chép, Analyst - Phân tích, Maintainer - Bảo trì, Processor - Điều phối) đảm nhận các tác vụ nặng nề hơn thông qua Long Chain-of-Thought.
*   **Knowledge Graph (Đồ thị tri thức):** Thay vì phân cấp thư mục cứng nhắc, hệ thống dùng **MOC (Map of Content)** tại `01-index/`. Mỗi ghi chú mới phải backlink về Index của dự án để đảm bảo tính điều hướng trong Obsidian.
*   **Stateless logic:** Hệ thống không dùng cơ sở dữ liệu. Mọi trạng thái được đọc trực tiếp từ các file Markdown bằng lệnh `glob` và `grep` thời gian thực.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Automated Hooks:**
    *   `SessionStart`: Tự động nạp ngữ cảnh dự án, ngày tháng ngay khi mở Claude.
    *   `PostToolUse`: Tự động Git commit sau mỗi lần AI ghi hoặc sửa file, đảm bảo lịch sử thay đổi luôn được bảo toàn (Auto-save).
*   **Keyword Detection Pipeline:** Sử dụng AI để quét văn bản thô, nhận diện các tín hiệu như "blocked", "decided", "shipped" để tự động gợi ý tạo các ghi chú Blocker (vấn đề) hoặc Decision (quyết định) tương ứng.
*   **Templating System:** Sử dụng cú pháp giống Handlebars trong các file `_templates/` để đảm bảo mọi ghi chú (PRD, Meeting, retro) luôn đồng nhất về định dạng và Metadata.
*   **Export Automation:** Kỹ thuật chuyển đổi Markdown thành tệp tin Office (.docx, .xlsx, .pptx) và PDF chuyên nghiệp để báo cáo cho các bên liên quan (Stakeholders) mà không cần rời khỏi Terminal.
*   **Semantic Recovery:** Tích hợp QMD giúp AI tìm thấy các quyết định cũ từ vài tháng trước ngay cả khi người dùng không nhớ chính xác từ khóa.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Một chu kỳ làm việc điển hình của PM-Kit:

1.  **Khởi động (Booting):** Người dùng gõ `claude`. Hook `session-init.sh` chạy, xác định hôm nay là ngày nào, những dự án nào đang hoạt động.
2.  **Tiếp nhận (Input):** Người dùng nhập: `/daily project-a: shipped module login, blocked on API key`.
3.  **Xử lý (Processing):** 
    *   Agent **Processor** nhận diện "module login" là thành quả, "API key" là vật cản.
    *   Agent **Scribe** mở template `blocker.md`, tạo file mới trong `blockers/project-a/` và điền thông tin người chịu trách nhiệm, độ ưu tiên.
4.  **Lưu trữ (Storage):** Ghi thông tin vào file `daily/2026-03-17.md`. Agent **Maintainer** cập nhật file Index dự án để thêm link đến Blocker vừa tạo.
5.  **Đóng gói (Closing):** Hook `auto-commit.sh` thực hiện `git add .` và `git commit -m "Update daily note"`. Người dùng gõ `/push` để đồng đẩy lên Cloud.
6.  **Truy vấn (Query):** Cuối tuần, người dùng gõ `/weekly`. Agent **Analyst** sẽ đọc tất cả file trong tuần, tổng hợp các mục đã hoàn thành và các vấn đề còn tồn đọng thành một báo cáo Retro.

### Kết luận
PM-Kit là một minh chứng cho việc sử dụng **Markdown làm hệ điều hành**. Nó tối giản hóa công cụ (không server, không database phức tạp) nhưng tối đa hóa khả năng của AI bằng cách cung cấp một cấu trúc dữ liệu cực kỳ chặt chẽ và dễ đoán.