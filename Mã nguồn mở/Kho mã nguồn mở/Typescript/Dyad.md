Dựa trên cấu trúc thư mục và nội dung các tệp tin bạn cung cấp (đặc biệt là `README.md`, `AGENTS.md`, và các tài liệu kiến trúc), dưới đây là phân tích chi tiết về dự án **Dyad**:

### 1. Công nghệ cốt lõi (Core Technology Stack)
Dyad là một ứng dụng Desktop xây dựng trên nền tảng **Electron**, cho phép nó kết hợp sức mạnh của giao diện web và khả năng can thiệp sâu vào hệ thống của Node.js.

*   **Frontend:**
    *   **Framework:** React (v19) với TanStack Router để điều hướng và TanStack Query để quản lý dữ liệu.
    *   **State Management:** Jotai (Atomic state) giúp quản lý trạng thái ứng dụng một cách tinh gọn.
    *   **UI/Styling:** Tailwind CSS (v4), Lucide React (icons), và đặc biệt là **Base UI** (@base-ui/react) cho các thành phần headless UI.
    *   **Editor:** Monaco Editor (giống VS Code) và Lexical cho các ô nhập liệu chat phức tạp.
*   **Backend (Main Process):**
    *   **Runtime:** Node.js tích hợp trong Electron.
    *   **Database:** **SQLite** được quản lý thông qua **Drizzle ORM**. Đây là lựa chọn tối ưu cho ứng dụng local để lưu trữ lịch sử chat, cài đặt và metadata của app.
    *   **AI Integration:** Sử dụng **Vercel AI SDK**, hỗ trợ đa dạng Provider (OpenAI, Anthropic, Google Gemini, DeepSeek, Ollama, LM Studio).
    *   **Hệ thống file & Shell:** Sử dụng `node-pty` để chạy terminal ảo và `dugite` cho các thao tác Git.
*   **Tooling:** Biome/Oxlint (thay thế Prettier/ESLint) để đạt tốc độ lint/format cực nhanh.

### 2. Tư duy kiến trúc (Architectural Thinking)
Kiến trúc của Dyad tập trung vào việc biến LLM (Large Language Model) thành một "kỹ sư phần mềm" thực thụ có quyền truy cập vào máy tính cục bộ.

*   **Đặc quyền ưu tiên Local (Local-First):** Khác với v0 hay Bolt chạy trên Cloud, Dyad chạy trực tiếp trên máy người dùng. Điều này đảm bảo quyền riêng tư và cho phép LLM can thiệp trực tiếp vào file system mà không cần proxy phức tạp.
*   **Mô phỏng Tool-Calling qua XML (XML-based Tooling):** Dyad hướng dẫn LLM trả về các tag đặc biệt như `<dyad-write>`, `<dyad-edit>`, `<dyad-add-dependency>`.
    *   *Lý do:* Cho phép gọi nhiều công cụ cùng lúc trong một lần phản hồi (parallelism) và giúp việc streaming giao diện (UI) mượt mà hơn khi người dùng có thể thấy sự thay đổi ngay khi LLM đang "viết" tag.
*   **Phân tách tiến trình (Security Boundary):**
    *   **Renderer Process:** Chỉ lo hiển thị, không có quyền ghi file.
    *   **Main Process:** Giữ quyền hạn cao nhất (ghi file, chạy lệnh shell). Giao tiếp qua **IPC (Inter-Process Communication)** một cách nghiêm ngặt để đảm bảo bảo mật.
*   **Smart Context (Ngữ cảnh thông minh):** Thay vì gửi toàn bộ code (gây tốn token), Dyad có cơ chế lọc các file quan trọng nhất dựa trên câu hỏi của người dùng để tối ưu chi phí và độ chính xác.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)
*   **IPC Contract-based:** Định nghĩa các "hợp đồng" giao tiếp giữa UI và Backend thông qua TypeScript, giúp bắt lỗi ngay khi lập trình (type-safety).
*   **Streaming Parser:** Một kỹ thuật quan trọng trong `DyadMarkdownParser.tsx`. Khi LLM đang stream dữ liệu về, parser phải nhận diện được các tag XML dở dang để hiển thị các UI component tương ứng (ví dụ: một cái card hiển thị file đang được tạo) thay vì chỉ hiện text thô.
*   **Atomic State với Jotai:** Chia nhỏ trạng thái (như `chatAtoms`, `appAtoms`) giúp ứng dụng không phải render lại toàn bộ khi một thay đổi nhỏ xảy ra, duy trì hiệu năng cao trên desktop.
*   **Database Migrations:** Sử dụng Drizzle-kit để quản lý sự thay đổi của cấu trúc cơ sở dữ liệu SQLite một cách nhất quán (thư mục `drizzle/`).

### 4. Luồng hoạt động của hệ thống (System Workflow)

1.  **Giai đoạn Thu thập Ngữ cảnh (Context Gathering):**
    *   Người dùng nhập yêu cầu.
    *   Hệ thống quét cấu trúc thư mục, lấy nội dung các file liên quan (hoặc toàn bộ nếu app nhỏ).
    *   Gộp vào System Prompt (chứa hướng dẫn cách dùng các tag `<dyad-*>`).
2.  **Giai đoạn Suy luận & Stream (LLM Inference):**
    *   Gửi yêu cầu tới AI Provider (ví dụ: Claude 3.5 Sonnet).
    *   AI phản hồi dưới dạng Markdown kết hợp XML tags.
    *   UI bắt được các tag này và hiển thị dưới dạng các "Action Cards" (ví dụ: nút "Approve" để cho phép ghi file).
3.  **Giai đoạn Thực thi (Execution):**
    *   Người dùng nhấn **Approve**.
    *   Renderer gửi lệnh qua IPC tới Main Process.
    *   Main Process thực hiện: ghi file, chạy `npm install`, tạo commit Git hoặc chạy SQL.
4.  **Giai đoạn Xem trước (Preview):**
    *   Ứng dụng sử dụng một iframe hoặc một server local (Vite) để render app vừa tạo.
    *   Cơ chế HMR (Hot Module Replacement) giúp thay đổi hiện ra ngay lập tức mà không cần load lại trang.

### 5. Điểm đặc biệt khác
*   **MCP (Model Context Protocol):** Dyad đã tích hợp MCP, cho phép LLM kết nối với các công cụ bên ngoài (Google Drive, Slack, cơ sở dữ liệu) một cách chuẩn hóa.
*   **Agent v2:** Hệ thống đang chuyển dịch sang cơ chế Agentic tự trị hơn, có khả năng tự chạy lệnh shell, đọc log lỗi và tự sửa lỗi (Auto-fix) thông qua các vòng lặp tool-calling.

**Tổng kết:** Dyad là một dự án có kiến trúc Electron hiện đại, tối ưu cho việc tương tác giữa AI và hệ thống tệp cục bộ, với tư duy thiết kế tập trung vào hiệu suất cao và trải nghiệm người dùng "thấy là có ngay".