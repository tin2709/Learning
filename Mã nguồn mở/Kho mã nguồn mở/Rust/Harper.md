Dựa trên cấu trúc thư mục và nội dung mã nguồn của dự án **Harper** (thuộc sở hữu của Automattic), đây là phân tích chuyên sâu về hệ thống kiểm tra ngữ pháp hiện đại, ưu tiên quyền riêng tư này:

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

Harper là một dự án đa nền tảng với sự kết hợp chặt chẽ giữa hiệu suất của hệ thống thấp và tính linh hoạt của giao diện web:

*   **Ngôn ngữ chính:** **Rust (chiếm ~79%)**. Được chọn vì khả năng quản lý bộ nhớ an toàn và tốc độ xử lý cực cao, giúp đạt mục tiêu kiểm tra ngữ pháp dưới 10ms.
*   **WebAssembly (WASM):** Dự án sử dụng `wasm-pack` để biên dịch lõi Rust thành WASM (`harper-wasm`). Điều này cho phép chạy cùng một logic kiểm tra ngữ pháp trên trình duyệt (`harper.js`) mà không cần gửi dữ liệu về máy chủ.
*   **Xử lý ngôn ngữ tự nhiên (NLP):**
    *   **Brill Tagger:** Một thuật toán gán nhãn từ loại (Part-of-Speech) dựa trên quy tắc.
    *   **FST (Finite State Transducers):** Sử dụng crate `fst` để lưu trữ từ điển một cách nén và tìm kiếm cực nhanh.
    *   **Levenshtein Automata:** Dùng để gợi ý sửa lỗi chính tả dựa trên khoảng cách chỉnh sửa.
*   **Frontend & Integrations:** Sử dụng **Svelte/SvelteKit** cho trang web và plugin trình duyệt. **Tauri v2** được dùng để xây dựng ứng dụng Desktop.
*   **Phân tích cú pháp (Parsing):** Tích hợp **Tree-sitter** để hiểu cấu trúc mã nguồn của nhiều ngôn ngữ lập trình (C++, Java, Go...), giúp Harper chỉ kiểm tra ngữ pháp trong các đoạn comment mà bỏ qua mã code.

### 2. Tư duy Kiến trúc (Architectural Philosophy)

Kiến trúc của Harper được thiết kế theo mô hình **"Core-out"** (Lấy lõi làm trung tâm):

*   **Tính mô-đun hóa cực cao:**
    *   `harper-core`: Chứa toàn bộ logic xử lý văn bản, tokenization, và các quy tắc (linters).
    *   `harper-ls`: Triển khai **Language Server Protocol (LSP)**, cho phép Harper tích hợp vào bất kỳ trình soạn thảo nào hỗ trợ LSP (VS Code, Neovim, Helix).
    *   `harper-comments`: Một lớp chuyên biệt để trích xuất văn bản từ comment trong code.
*   **Privacy-First (Riêng tư là trên hết):** Kiến trúc không có backend xử lý văn bản. Mọi thao tác linting đều diễn ra local trên máy người dùng hoặc trong worker của trình duyệt.
*   **Hiệu suất là tính năng:** Dự án coi thời gian linting dài là một "bug". Việc sử dụng Rust và FST giúp Harper tiêu tốn ít hơn 1/50 bộ nhớ so với LanguageTool.

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Techniques)

*   **Ngôn ngữ quy tắc Weir (`.weir`):** Đây là một điểm cực kỳ sáng tạo. Harper phát triển một DSL (Domain Specific Language) riêng để định nghĩa các quy tắc ngữ pháp. Thay vì viết code Rust phức tạp, người đóng góp có thể viết các biểu thức ngắn gọn để mô tả các mẫu lỗi.
*   **Kỹ thuật Masking (Mặt nạ):** Khi xử lý các định dạng phức tạp như Markdown, LaTeX hay mã nguồn, Harper sử dụng các bộ `Masker`. Kỹ thuật này giúp bộ máy kiểm tra chỉ "nhìn thấy" phần văn bản thuần túy cần kiểm tra, trong khi vẫn giữ nguyên vị trí (span) gốc để highlight chính xác lỗi trên file nguồn.
*   **Hệ thống Chunker Neural (Burn):** Sử dụng framework Deep Learning **Burn** (viết bằng Rust) để thực hiện `BurnChunkerCpu`, giúp trích xuất các cụm danh từ (nominal phrases) một cách thông minh bằng AI nhưng vẫn chạy tốt trên CPU.
*   **Snapshot Testing:** Sử dụng các file `.snap.yml` để đảm bảo các bản cập nhật quy tắc không gây ra lỗi hồi quy (regression) trên các tập dữ liệu mẫu như Hiến pháp Hoa Kỳ hay các bài luận văn.

### 4. Luồng Hoạt động Hệ thống (System Workflows)

#### A. Luồng Linting cơ bản:
1.  **Input:** Nhận văn bản thô (Markdown, Code, Plain text).
2.  **Transformation:** Parser (ví dụ: `MarkdownParser`) phân rã văn bản thành các `Token`.
3.  **POS Tagging:** `BrillTagger` gán nhãn từ loại cho từng từ (Danh từ, Động từ, Tính từ...).
4.  **Rule Execution:** `LintGroup` chạy song song các quy tắc (Linter). Mỗi quy tắc quét qua các Token để tìm mẫu lỗi.
5.  **Reporting:** Trả về các đối tượng `Lint` chứa `Span` (vị trí lỗi), `Message` và `Suggestions` (gợi ý sửa).

#### B. Luồng hoạt động của Harper Desktop (Native Overlay):
1.  Sử dụng các API trợ năng (Accessibility) của hệ điều hành (đặc biệt trên macOS qua `MacBroker`) để đọc văn bản từ các ứng dụng khác đang mở.
2.  Tạo một lớp phủ đồ họa (Overlay) bằng **egui/winit** vẽ trực tiếp lên màn hình.
3.  Khi người dùng click vào lỗi trên lớp phủ, một popup Svelte (trong Tauri) sẽ xuất hiện để cho phép áp dụng gợi ý sửa lỗi ngay lập tức.

#### C. Luồng build WASM:
*   `Justfile` điều phối: Gọi `wasm-pack` -> Tối ưu hóa file `.wasm` -> Đóng gói vào `harper.js` -> Phát hành lên NPM. Điều này tạo ra một thư viện kiểm tra ngữ pháp mạnh mẽ cho cộng đồng JavaScript.

### Tổng kết
Harper không chỉ là một công cụ kiểm tra ngữ pháp; nó là một **nền tảng phân tích ngôn ngữ (NLP Platform)** viết bằng Rust. Sự kết hợp giữa thuật toán truyền thống (Brill Tagger) và hiện đại (Neural Chunker), cùng với hệ thống quy tắc Weir linh hoạt, biến nó thành đối thủ đáng gờm cho các dịch vụ đám mây như Grammarly.