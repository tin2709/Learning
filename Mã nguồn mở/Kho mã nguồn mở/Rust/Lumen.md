Dựa trên mã nguồn của dự án **Lumen**, đây là phân tích chi tiết về kiến trúc và công nghệ của công cụ review mã nguồn bằng AI này:

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ lập trình:** **Rust** (chiếm 97.7%). Lumen tận dụng tính an toàn, hiệu năng cao và hệ sinh thái thư viện TUI (Terminal User Interface) mạnh mẽ của Rust.
*   **Tích hợp AI (LLM):** Sử dụng crate `genai`. Đây là một thư viện trừu tượng hóa việc giao tiếp với nhiều nhà cung cấp AI khác nhau như OpenAI, Claude (Anthropic), Gemini (Google), Groq, DeepSeek và cả các mô hình chạy cục bộ qua Ollama.
*   **Giao diện dòng lệnh (CLI) & TUI:**
    *   `clap`: Xử lý đối số dòng lệnh mạnh mẽ.
    *   `ratatui` & `crossterm`: Bộ đôi tiêu chuẩn để xây dựng giao diện terminal tương tác, hỗ trợ vẽ diff side-by-side, thanh điều hướng và cửa sổ modal.
*   **Thuật toán Diff:** Sử dụng crate `similar` để tính toán sự khác biệt giữa các phiên bản tệp tin.
*   **Syntax Highlighting:** Sử dụng **Tree-sitter** (qua `tree-sitter-highlight`). Khác với các trình làm đẹp code dùng Regex, Tree-sitter hiểu cấu trúc cây cú pháp (AST), giúp highlight chính xác cho hơn 15 ngôn ngữ lập trình.
*   **Hệ thống quản lý phiên bản (VCS):** Sử dụng `git2` (bindings của libgit2) và hỗ trợ cả **Jujutsu (jj)** thông qua crate `jj-lib`.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Lumen được thiết kế theo hướng **Modularity (Mô-đun hóa)** và **Abstraction (Trừu tượng hóa)** cao:

*   **VCS Agnostic (Độc lập hệ thống quản lý phiên bản):** Lumen định nghĩa một trait `VcsBackend` (`vcs/backend.rs`). Mọi chức năng (lấy diff, lấy thông tin commit) đều gọi qua trait này. Điều này cho phép Lumen hỗ trợ mượt mà cả Git và các VCS mới nổi như Jujutsu mà không cần sửa đổi logic AI.
*   **Command Pattern:** Mỗi tính năng lớn (`diff`, `draft`, `explain`, `operate`) được tách thành các cấu trúc riêng trong thư mục `command/`. Mỗi lệnh tự quản lý logic thực thi và cách hiển thị kết quả riêng.
*   **Layered Prompting:** Logic xây dựng prompt được tách rời (`ai_prompt.rs`). Hệ thống thu thập dữ liệu thô từ VCS (diff/message), sau đó "đúc" vào các template prompt có cấu trúc (system prompt và user prompt) trước khi gửi đi.
*   **State-Driven TUI:** Giao diện xem diff (`diff/state.rs`) được quản lý bằng một máy trạng thái (`AppState`). Mọi hành động của người dùng (cuộn, đổi tệp, thêm annotation) đều cập nhật vào trạng thái trung tâm trước khi Ratatui render lại.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Phân tích ngữ cảnh bằng AST:** Trong tính năng `sticky_lines.rs` và `context.rs`, Lumen sử dụng Tree-sitter để xác định các khối code (như hàm hoặc lớp). Khi người dùng cuộn code quá dài, hệ thống vẫn hiển thị tên hàm đang chứa đoạn code đó ở trên cùng (sticky header) — một tính năng cao cấp thường chỉ thấy ở IDE.
*   **Tối ưu hóa hiệu năng tệp lớn:**
    *   Sử dụng `Lazy` (từ `once_cell`) để nạp cấu hình highlight và cấu trúc Tree-sitter chỉ khi cần.
    *   Cơ chế **Caching side-by-side diff**: Kết quả diff và highlight được lưu lại trong `AppState` để tránh tính toán lại mỗi khung hình (frame) khi render TUI.
*   **Coordinate Mapping (Ánh xạ tọa độ):** Mã nguồn trong `coordinates.rs` xử lý việc chuyển đổi tọa độ chuột từ lưới ký tự terminal sang vị trí chính xác trong dòng/cột của tệp nguồn, hỗ trợ chọn văn bản bằng chuột (click-drag) ngay trong terminal.
*   **Xử lý phản hồi AI có cấu trúc:** Lệnh `operate` yêu cầu AI trả về kết quả theo định dạng thẻ XML. Lumen sử dụng `xml-rs` để phân tách an toàn lệnh cần chạy (`<command>`), giải thích (`<explanation>`) và cảnh báo (`<warning>`).

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Khởi tạo:** 
    *   `main.rs` đọc cấu hình từ tệp JSON (`~/.config/lumen/lumen.config.json`) và biến môi trường.
    *   Tự động phát hiện thư mục hiện tại là Git hay JJ để khởi tạo Backend tương ứng.
2.  **Thu thập dữ liệu:** 
    *   Lấy danh sách các thay đổi (`staged` hoặc commit cụ thể).
    *   Trích xuất nội dung code cũ và mới.
3.  **Xử lý thông minh (AI/Logic):**
    *   **Nếu là `diff`:** Chạy thuật toán so sánh dòng -> Chạy highlight cú pháp -> Khởi chạy vòng lặp sự kiện (event loop) của TUI.
    *   **Nếu là `explain`/`draft`:** Tổng hợp code diff vào prompt -> Gửi LLM -> Nhận phản hồi -> Render Markdown ra terminal bằng `mdcat`.
4.  **Tương tác:**
    *   Trong TUI, người dùng có thể thêm `Annotation` (bình luận). Những bình luận này được lưu vào bộ nhớ và có thể xuất ra định dạng Review Comment của GitHub.

### Tổng kết
Lumen không chỉ là một wrapper đơn giản cho LLM. Điểm mạnh của nó nằm ở việc **kết hợp sâu sắc giữa hiểu biết về cú pháp code (Tree-sitter) và AI**. Kiến trúc đa Backend của nó giúp công cụ này có tính tương lai cao, sẵn sàng thích nghi với các quy trình làm việc hiện đại nhất của kỹ sư phần mềm.