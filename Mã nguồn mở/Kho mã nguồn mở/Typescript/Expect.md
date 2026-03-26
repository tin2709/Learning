Dựa trên tài liệu và mã nguồn bạn cung cấp, đây là phân tích chi tiết về dự án **Expect** — một hệ thống kiểm thử tự động dành cho AI Agent.

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

Hệ thống này sử dụng một tập hợp công nghệ rất hiện đại, tập trung vào mô hình lập trình hàm (functional programming) và tự động hóa trình duyệt:

*   **Effect-TS (Trụ cột chính):** Đây là nền tảng của toàn bộ backend. Hệ thống sử dụng Effect v4 (beta) để quản lý side-effects, dependency injection (qua `Layer`), và xử lý lỗi một cách chặt chẽ. Việc sử dụng `Effect.gen`, `Schema`, và `ServiceMap` cho thấy một tư duy lập trình hướng đến sự an toàn tuyệt đối về kiểu dữ liệu (type-safety).
*   **Playwright & MCP (Model Context Protocol):** Sử dụng Playwright để điều khiển trình duyệt. MCP được dùng làm giao thức giao tiếp, cho phép các AI Agent (như Claude) gọi các "tools" (click, screenshot, type) để tương tác với web như một con người.
*   **Ink & React:** Thay vì CLI truyền thống, `expect` sử dụng **Ink** để xây dựng Terminal UI (TUI) bằng React. Điều này cho phép tạo ra các giao diện phức tạp, tương tác cao (như chọn PR, xem tiến trình test) ngay trong terminal.
*   **React Compiler:** Dự án áp dụng công nghệ mới nhất từ React, tự động tối ưu hóa (memoization) mà không cần `useMemo` hay `useCallback` thủ công.
*   **Observability (rrweb & Remotion):** Sử dụng `rrweb` để ghi lại (record) mọi sự kiện trong trình duyệt và `Remotion` (trong package `@expect/video`) để chuyển đổi các phiên test thành video, giúp lập trình viên xem lại quá trình AI thực hiện test.
*   **Hono:** Sử dụng Hono làm web server siêu nhẹ để chạy proxy server phục vụ live-view (xem trực tiếp quá trình test qua trình duyệt).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của **Expect** được thiết kế theo mô hình **Supervisor-Agent**, tối ưu cho việc mở rộng và bảo trì trong monorepo:

*   **Tách biệt logic nghiệp vụ (Supervisor) và giao diện (CLI):** Package `@expect/supervisor` đóng vai trò là "bộ não" quản lý toàn bộ trạng thái (state), vòng đời của agent và các thao tác Git. CLI chỉ là một lớp hiển thị (stateless renderer) dữ liệu từ supervisor.
*   **Hệ thống Layer (Dependency Injection):** Sử dụng `Layer` của Effect để quản lý các dịch vụ. Ví dụ, `layerCli` kết hợp `Executor`, `Reporter`, `Git`, và `Agent` thành một khối thống nhất nhưng vẫn đảm bảo tính module hóa, dễ dàng thay thế (như đổi từ Claude sang OpenAI).
*   **Kiến trúc Monorepo chặt chẽ:** Phân chia rõ ràng giữa các domain: `@expect/browser` (tự động hóa), `@expect/cookies` (trích xuất dữ liệu đăng nhập), `@expect/shared` (models chung).
*   **Tư duy "No Barrel Files":** Tránh sử dụng tệp `index.ts` để re-export. Điều này giúp giảm thiểu việc phụ thuộc vòng (circular dependencies) và tăng tốc độ biên dịch/phân tích mã nguồn.
*   **An toàn hóa dữ liệu nhạy cảm:** Việc trích xuất cookie được thực hiện cục bộ (`@expect/cookies`), giúp AI Agent có thể test các trang yêu cầu đăng nhập mà không cần gửi mật khẩu lên cloud.

---

### 3. Kỹ thuật lập trình chính (Key Coding Techniques)

*   **Functional Programming với Effect:**
    *   Sử dụng `Schema.ErrorClass` để định nghĩa lỗi có cấu trúc.
    *   Thay vì `try/catch`, sử dụng `catchTag` để bắt các lỗi cụ thể, biến các lỗi hạ tầng không thể khôi phục thành `Defects` (thông qua `Effect.die`).
    *   Sử dụng `Branded Types` cho ID (ví dụ: `TaskId`) để ngăn chặn việc truyền nhầm ID giữa các thực thể khác nhau.
*   **Reactive State Management:** Sử dụng **Effect Atom** (trong `apps/cli/src/data/runtime.ts`) để đồng bộ trạng thái giữa logic Effect backend và các thành phần React UI trong CLI.
*   **Xử lý bất đồng bộ nâng cao:** Sử dụng `FiberMap` để quản lý các tác vụ chạy song song với khả năng tự động hủy (auto-cancellation), tránh rò rỉ bộ nhớ hoặc tài nguyên khi người dùng nhấn ESC để dừng test.
*   **Image Rendering trong Terminal:** Sử dụng các chuỗi escape (Kitty/iTerm2 protocol) để hiển thị ảnh chụp màn hình trực tiếp trong Terminal, một kỹ thuật nâng cao dành cho các CLI hiện đại.
*   **Tận dụng Schema-driven Development:** Toàn bộ dữ liệu trao đổi (JSON, DB, API) đều được kiểm tra qua `Effect.Schema`, đảm bảo dữ liệu luôn đúng cấu trúc trước khi xử lý.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng đi của dữ liệu từ khi khởi chạy đến khi kết thúc:

1.  **Quét thay đổi (Scanning):** Người dùng chạy lệnh `expect`. Supervisor sử dụng dịch vụ `Git` để lấy diff giữa branch hiện tại và branch chính (main).
2.  **Lập kế hoạch (Planning):** Nội dung diff được gửi đến AI Agent thông qua `@expect/agent`. Agent phân tích mã nguồn thay đổi và tạo ra một **Test Plan** gồm các bước (ví dụ: "Mở trang chủ", "Click nút login", "Kiểm tra lỗi").
3.  **Duyệt kế hoạch (Reviewing):** Test Plan hiển thị trên giao diện TUI (Ink). Người dùng có thể xem và xác nhận.
4.  **Thực thi (Executing):** 
    *   Dịch vụ `Executor` khởi động. 
    *   Trình duyệt Playwright được mở. 
    *   Cookie từ trình duyệt thật (Chrome/Safari/Firefox) của người dùng được nạp vào để giữ trạng thái đăng nhập.
    *   Agent gọi các công cụ MCP để thực hiện từng bước trong trình duyệt.
5.  **Ghi chép & Báo cáo (Reporting):** 
    *   `rrweb` ghi lại các hành động. 
    *   Nếu bước nào lỗi, Agent sẽ chụp ảnh màn hình và phân tích lý do.
6.  **Kết quả (Finalizing):** TUI hiển thị trạng thái Pass/Fail, thời gian thực hiện và cung cấp link xem lại (Replay) phiên test qua proxy server địa phương hoặc video.

### Tổng kết
Dự án này là một ví dụ điển hình về việc ứng dụng **Functional Programming (Effect-TS)** để giải quyết các vấn đề phức tạp trong việc điều phối AI. Nó không chỉ đơn thuần là gọi API của OpenAI/Anthropic, mà còn xây dựng một hạ tầng observability mạnh mẽ để kiểm soát hành vi của AI Agent trong môi trường thực tế.