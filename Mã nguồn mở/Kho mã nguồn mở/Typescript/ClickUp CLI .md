Dưới đây là phân tích chi tiết về dự án **ClickUp CLI (`cup`)**, một công cụ được thiết kế tối ưu cho cả con người và các Agent AI (như Claude Code).

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án sử dụng các công nghệ hiện đại nhất trong hệ sinh thái Node.js để đảm bảo tốc độ và tính tương thích cao:

*   **TypeScript (Strict Mode):** Toàn bộ mã nguồn sử dụng TypeScript với cấu hình nghiêm ngặt (`verbatimModuleSyntax`, `noUncheckedIndexedAccess`), đảm bảo an toàn kiểu dữ liệu, đặc biệt quan trọng khi xử lý các API phản hồi không đồng nhất.
*   **Node.js 22+ & ESM-only:** Tận dụng các tính năng mới nhất của Node.js, chỉ sử dụng định dạng ECMAScript Modules (ESM) để tối ưu hóa việc đóng gói và hiệu suất.
*   **Commander.js:** Framework mạnh mẽ để xử lý các lệnh (commands), đối số (arguments) và cờ (flags) của CLI.
*   **@inquirer/prompts:** Thư viện xử lý tương tác terminal (như checkbox, select, confirm), tạo ra trải nghiệm UI mượt mà trong chế độ TTY.
*   **tsup:** Công cụ build cực nhanh dựa trên esbuild, giúp đóng gói mã nguồn thành một tệp ESM duy nhất (`dist/index.js`).
*   **Vitest:** Hệ thống kiểm thử hiện đại, hỗ trợ cả Unit test (mock API) và E2E test (gọi API thật).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của `cup` được thiết kế theo triết lý **"Agent-First"** và **"Simplicity over Protocol"**:

*   **CLI + Skill thay vì MCP:** Thay vì xây dựng một server Model Context Protocol (MCP) phức tạp, dự án chọn hướng CLI kèm tệp `SKILL.md`. Điều này giúp AI dễ dàng thực thi lệnh thông qua shell mà không cần duy trì kết nối server-client hay giao thức bổ sung.
*   **Đa dạng chế độ đầu ra (Output Modes):** 
    *   *TTY Mode:* Hiển thị bảng màu, danh sách chọn tương tác cho người dùng.
    *   *Agent Mode (Piped):* Tự động phát hiện khi kết quả được dẫn ống (pipe) để trả về Markdown thuần, giúp tiết kiệm token và tối ưu hóa khả năng đọc hiểu của LLM.
    *   *Data Mode:* Hỗ trợ `--json` cho các luồng xử lý tự động cần dữ liệu cấu trúc.
*   **Cấu trúc Modular Command:** Mỗi lệnh ClickUp (tasks, sprints, comments...) được tách riêng thành một tệp trong `src/commands/`, giúp việc bảo trì và mở rộng tính năng trở nên độc lập.
*   **Quản lý cấu hình đa hồ sơ (Multi-profile):** Hỗ trợ nhiều tài khoản/workspace khác nhau thông qua hệ thống profile trong `~/.config/cup/config.json`.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Xử lý ID linh hoạt (Custom ID Detection):** Sử dụng Regex để tự động phát hiện định dạng Task ID (ví dụ: `PROJ-123` so với ID gốc của ClickUp). Hệ thống tự động thêm cờ `custom_task_ids=true` khi gọi API nếu phát hiện ID tùy chỉnh.
*   **Khớp trạng thái mờ (Fuzzy Status Matching):** Trong `status.ts`, hệ thống cho phép người dùng nhập viết tắt (ví dụ: `prog` thay vì `In Progress`) và tự động tìm kiếm trạng thái khớp nhất.
*   **Nhận diện Sprint thông minh:** Thuật toán trong `sprint.ts` quét các thư mục có từ khóa đặc trưng (sprint, iteration, cycle...) và phân tích chuỗi ngày tháng theo nhiều định dạng (US, ISO, Châu Âu) để tìm ra Sprint đang hoạt động.
*   **Phân trang API tự động:** Lớp `ClickUpClient` xử lý việc lặp (loop) qua các trang dữ liệu cho đến khi lấy đủ Task hoặc đạt giới hạn an toàn (`MAX_PAGES`).
*   **Kiểm soát luồng phát hành (Release Automation):** Sử dụng GitHub Actions với **OIDC Trusted Publishers**, cho phép phát hành lên npm mà không cần lưu trữ token bí mật (secret), tăng độ bảo mật tuyệt đối cho chuỗi cung ứng mã nguồn.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Khởi tạo:** Người dùng chạy `cup <lệnh>`. Hệ thống kiểm tra biến môi trường hoặc đọc tệp cấu hình để lấy API Token và Team ID.
2.  **Định tuyến:** `Commander` phân tích cú pháp và chuyển hướng yêu cầu đến handler tương ứng trong thư mục `commands/`.
3.  **Xử lý API:**
    *   Handler gọi phương thức trong `api.ts`.
    *   `ClickUpClient` thực hiện yêu cầu HTTPS đến ClickUp API (v2 hoặc v3 tùy tính năng).
    *   Dữ liệu trả về được chuẩn hóa kiểu dữ liệu (coercion) để tránh lỗi không đồng nhất giữa string/number.
4.  **Định dạng đầu ra:**
    *   Hệ thống kiểm tra `process.stdout.isTTY`.
    *   Nếu là con người (Terminal): Render bảng dùng `chalk`.
    *   Nếu là AI (Piped): Render Markdown dùng `markdown.ts`.
    *   Nếu có cờ `--json`: Xuất JSON trực tiếp.
5.  **Tương tác (nếu có):** Nếu kết quả trả về nhiều Task trong chế độ terminal, một picker sẽ hiện ra để người dùng chọn Task cần xử lý tiếp (xem chi tiết hoặc mở trình duyệt).

### Tổng kết
`cup` không chỉ là một công cụ quản lý tác vụ; nó là một **giao diện lập trình ngôn ngữ tự nhiên (Natural Language Interface)** cho ClickUp. Bằng cách cung cấp đầu ra Markdown tối ưu và logic xử lý ID thông minh, nó biến AI từ một người quan sát thành một thành viên thực thụ có khả năng thao tác trực tiếp trên dự án.