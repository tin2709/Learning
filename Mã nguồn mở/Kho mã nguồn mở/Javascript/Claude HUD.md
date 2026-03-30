Dựa trên toàn bộ mã nguồn và tài liệu của dự án **Claude HUD**, dưới đây là phân tích chi tiết về kiến trúc, công nghệ và kỹ thuật lập trình của hệ thống này:

### 1. Công nghệ cốt lõi (Core Tech Stack)
*   **Ngôn ngữ:** TypeScript (Target ES2022, NodeNext). Dự án tận dụng tối đa hệ thống kiểu (types) để đảm bảo tính an toàn dữ liệu từ đầu vào Stdin.
*   **Runtime:** Hỗ trợ song song **Node.js (>=18)** và **Bun**. Đặc biệt, nếu chạy bằng Bun, nó sẽ thực thi trực tiếp file `.ts` để đạt hiệu suất cao nhất.
*   **CLI Framework:** Không sử dụng thư viện bên thứ ba (như Commander hay Inquirer) cho phần hiển thị HUD để giữ kích thước cực nhẹ và tốc độ khởi động nhanh. Nó sử dụng các lệnh ANSI escape code thuần túy để xử lý màu sắc và định dạng.
*   **Testing:** Sử dụng `node:test` (test runner tích hợp sẵn của Node.js) và `c8` để đo độ bao phủ (coverage), giúp giảm thiểu phụ thuộc (dependency).

### 2. Tư duy Kiến trúc (Architectural Design)
Hệ thống hoạt động theo mô hình **"Stateless Processor"** được kích hoạt bởi Claude Code mỗi ~300ms:

1.  **Input Layer (`stdin.ts` & `transcript.ts`):** 
    *   Nhận dữ liệu JSON từ Stdin (chứa thông tin Model, Context Window, Token).
    *   Đồng thời "đào" (parse) file `transcript.jsonl` (nhật ký phiên làm việc) để trích xuất các hành động của Tool và Agent mà Stdin không cung cấp.
2.  **Logic Layer (`config-reader.ts`, `git.ts`, `memory.ts`):**
    *   Tính toán trạng thái Git (branch, dirty, ahead/behind).
    *   Đếm số lượng file cấu hình (`CLAUDE.md`, MCP servers).
    *   Tính toán tài nguyên hệ thống (RAM).
3.  **Render Layer (`src/render/`):** 
    *   Chia nhỏ thành các component: `identity-line`, `tools-line`, `project-line`, v.v.
    *   Sử dụng cơ chế **Adaptive Width** để tự động co dãn hoặc cắt bớt (truncate) dữ liệu dựa trên chiều rộng của terminal (Columns).
4.  **Configuration Layer (`config.ts`):** 
    *   Hỗ trợ cấu hình phân tầng: Default -> User Config (`config.json`) -> Manual Overrides.

### 3. Kỹ thuật lập trình chính (Key Techniques)

#### A. Xử lý UI Terminal phức tạp
*   **Grapheme Splitting:** Sử dụng `Intl.Segmenter` (trong `src/render/index.ts`) để xử lý chính xác độ dài hiển thị của các ký tự đặc biệt như Emoji hoặc chữ tượng hình (CJK). Điều này ngăn chặn lỗi vỡ giao diện khi dùng thanh tiến trình (progress bar) với Emoji.
*   **ANSI Tokenization:** Một kỹ thuật thông minh để tách các mã màu ANSI ra khỏi văn bản thuần túy trước khi tính toán độ dài hiển thị (`visualLength`), đảm bảo việc căn lề (alignment) luôn chính xác.
*   **Adaptive Bar Width:** Hàm `getAdaptiveBarWidth` tự động điều chỉnh độ dài thanh Context (10, 6 hoặc 4 ký tự) tùy theo kích thước cửa sổ terminal.

#### B. Tối ưu hóa hiệu suất (Caching)
*   **Transcript Caching:** Vì file transcript có thể rất lớn, dự án sử dụng `transcript-cache` (trong `src/transcript.ts`) dựa trên Hash và Mtime của file. Nếu file chưa thay đổi, nó sẽ load kết quả parse cũ thay vì đọc lại toàn bộ file JSONL hàng triệu dòng.
*   **Version Caching:** Lưu trữ phiên bản của Claude Code vào file JSON để tránh việc phải thực thi lệnh `claude --version` (vốn rất chậm) trong mỗi chu kỳ 300ms.

#### C. Tính toán Context Window (Buffer Strategy)
*   **Autocompact Buffer:** Claude Code có cơ chế tự động nén ngữ cảnh (autocompact) khi đạt ngưỡng nhất định. Claude HUD áp dụng một công thức `AUTOCOMPACT_BUFFER_PERCENT` (16.5%) để con số % hiển thị trên HUD khớp chính xác với lệnh `/context` của Claude Code, giải quyết vấn đề "sai số" mà các plugin khác thường gặp.

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Cài đặt (`/claude-hud:setup`):** 
    *   Đây là một file Markdown (`commands/setup.md`) đóng vai trò như một script hướng dẫn cho LLM. 
    *   Nó sẽ dò tìm runtime (Bun/Node), tìm đường dẫn tuyệt đối của plugin và ghi một lệnh shell động vào `settings.json` của Claude.
2.  **Vòng lặp thực thi:**
    *   Claude Code khởi chạy HUD như một tiến trình con (subprocess).
    *   HUD đọc Stdin -> Quét Git -> Quét Transcript -> Render ra Stdout.
    *   Claude Code nhận Stdout và vẽ lại vùng Statusline phía dưới màn hình.
3.  **Cấu hình (`/claude-hud:configure`):** 
    *   LLM sẽ đọc file `config.json`, đưa ra các lựa chọn Preset (Full/Essential/Minimal) và cập nhật lại file JSON theo ý muốn người dùng.

### 5. Điểm sáng về kỹ thuật
*   **Ghost Installation Cleanup:** Có cơ chế phát hiện và dọn dẹp các bản cài đặt lỗi (interrupted installs) - một vấn đề phổ biến của hệ thống plugin Claude Code.
*   **Cross-Device Link Handling:** Xử lý lỗi `EXDEV` trên Linux khi thư mục `/tmp` nằm trên một filesystem khác với thư mục Home bằng cách hướng dẫn người dùng set `TMPDIR`.
*   **Color Overrides:** Cho phép người dùng ghi đè màu sắc bằng tên (green, red), mã số (0-255) hoặc mã Hex (`#rrggbb`), một tính năng cao cấp cho các "Power Users" thích tùy biến giao diện terminal.

**Tổng kết:** Dự án là một ví dụ mẫu mực về việc xây dựng công cụ CLI hiện đại: **Không phụ thuộc (Zero-dep), Hiệu suất cực cao, và Xử lý UI Terminal tinh tế.**