Dựa trên mã nguồn và tài liệu của dự án **Vibeyard**, dưới đây là phân tích chi tiết về các khía cạnh kỹ thuật và kiến trúc của hệ thống:

### 1. Công nghệ cốt lõi (Core Tech Stack)

*   **Runtime:** **Electron** - Sử dụng mô hình 3 quy trình (Main, Preload, Renderer) để xây dựng ứng dụng desktop.
*   **Terminal Rendering:** **xterm.js** - Thư viện chuẩn để hiển thị terminal trên web/electron, hỗ trợ WebGL để tối ưu hiệu năng.
*   **PTY (Pseudo-Terminal):** **node-pty** - Dùng để spawn và quản lý các tiến trình shell thực thụ ở phía backend (Main process).
*   **Ngôn ngữ:** **TypeScript (91%)** - Áp dụng strict mode trên toàn bộ project.
*   **Giao thức AI:** **Model Context Protocol (MCP)** - Tích hợp SDK của Anthropic để kết nối các công cụ/dữ liệu bên ngoài vào AI agent.
*   **Build Tool:** **esbuild** (cho renderer) và **tsc** (cho main/preload).
*   **Testing:** **Vitest** kết hợp với v8 coverage.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Vibeyard được thiết kế để giải quyết vấn đề "mù ngữ cảnh" khi chạy AI agent trong terminal thuần túy:

*   **Provider Pattern (Hệ thống nhà cung cấp):** Hệ thống không gắn chặt vào "Claude Code". Nó sử dụng interface `CliProvider` để trừu tượng hóa các hành vi của CLI (biên dịch binary, cấu hình biến môi trường, cài đặt hook). Điều này cho phép mở rộng sang các AI CLI khác như Copilot CLI hay Gemini CLI một cách dễ dàng.
*   **File-based Hook System (Hệ thống Hook dựa trên tệp):** Đây là một tư duy sáng tạo. Vì các CLI tool thường không có API để báo cáo trạng thái, Vibeyard cài đặt các lệnh shell (hooks) vào CLI. Khi CLI chạy đến một trạng thái nhất định (ví dụ: bắt đầu trả lời, gọi tool), nó sẽ ghi một tệp nhỏ vào `/tmp/vibeyard/`. Main process dùng `fs.watch` để theo dõi và cập nhật UI ngay lập tức.
*   **Reactive State Singleton:** Renderer process sử dụng một AppState singleton với mô hình Event Emitter. Các component UI đăng ký (subscribe) vào state và tự cập nhật khi có thay đổi, giúp giữ cho UI đồng bộ mà không cần các framework nặng nề như React.
*   **Performance First (Vanilla UI):** Không sử dụng framework UI (React/Vue). Tác giả chọn thao tác DOM trực tiếp (Vanilla TS) để đảm bảo độ trễ thấp nhất khi render luồng dữ liệu terminal liên tục.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Context Isolation & Security:** Sử dụng `contextBridge` để chỉ expose các API cần thiết từ quy trình Main sang Renderer, ngăn chặn các cuộc tấn công script vào hệ thống tệp của người dùng.
*   **Atomic Persistence:** Khi lưu trạng thái ứng dụng (`state.json`), hệ thống ghi vào một file `.tmp` trước rồi mới rename đè lên file chính. Kỹ thuật này ngăn chặn việc hỏng dữ liệu (corruption) nếu ứng dụng bị crash giữa chừng.
*   **Debouncing & Throttling:** Áp dụng cho các tác vụ nặng như: ghi file trạng thái xuống ổ đĩa (300ms delay), refresh bảng Git status, và cập nhật cấu hình khi file thay đổi để tránh quá tải CPU.
*   **Crawl & Parse Metadata:** Sử dụng Regex để quét output của terminal nhằm trích xuất thông tin về chi phí (USD), token đã dùng và tên model đang chạy mà không làm gián đoạn luồng hiển thị.
*   **Intelligent Path Resolution:** Tự động resolve `PATH` bằng cách chạy một shell login ẩn (login shell) khi khởi động, giúp ứng dụng tìm được các binary như `node`, `nvm` hay `claude` ngay cả khi app được mở từ GUI (vấn đề thường gặp trên macOS).

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng khởi tạo Session:
1.  **Renderer:** Gửi yêu cầu `pty:create` qua IPC.
2.  **Main:** Provider (Claude) tính toán các biến môi trường (như `CLAUDE_IDE_SESSION_ID`) và tham số dòng lệnh.
3.  **Main:** `node-pty` spawn tiến trình CLI.
4.  **Flow:** Dữ liệu từ PTY -> Main -> Preload -> Renderer -> xterm.js hiển thị.

#### B. Luồng theo dõi trạng thái (Status Tracking):
1.  **CLI Tool:** Chạy một hook (ví dụ: `PostToolUse`).
2.  **Filesystem:** Hook ghi trạng thái `working` vào `/tmp/vibeyard/{sid}.status`.
3.  **Main:** `fs.watch` phát hiện thay đổi -> Gửi sự kiện `session:hookStatus` tới Renderer.
4.  **Renderer:** Cập nhật icon trạng thái, phát âm thanh thông báo nếu session đang chạy ngầm hoàn thành.

#### C. Luồng đánh giá "AI Readiness":
1.  **Analyzer:** Quét project để tìm các tệp hướng dẫn (`CLAUDE.md`, `.cursorrules`).
2.  **Context Check:** Đếm số dòng của các tệp lớn, kiểm tra các tệp nhạy cảm (secrets) không có trong `.claudeignore`.
3.  **Scoring:** Tính toán điểm số dựa trên trọng số các hạng mục (Instructions: 50%, Optimization: 20%, Extensions: 30%) và hiển thị cảnh báo một-click-để-fix.

### Tổng kết
Vibeyard là một ví dụ điển hình về việc xây dựng **"Wrapper IDE"** hiện đại. Nó không cố gắng thay thế trình soạn thảo code, mà tập trung vào việc **quản trị phiên làm việc (session management)** và **tối ưu ngữ cảnh (context optimization)** cho AI, giúp lập trình viên điều khiển nhiều "agent" cùng lúc mà không bị mất dấu chi phí hay trạng thái thực thi.