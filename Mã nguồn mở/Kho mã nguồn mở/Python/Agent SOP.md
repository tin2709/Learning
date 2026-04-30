Dưới đây là phân tích chi tiết về dự án **Agent SOP** (Standard Operating Procedures) của Strands Agents. Đây là một khung làm việc (framework) nhằm chuẩn hóa cách hướng dẫn AI thực hiện các quy trình phức tạp thông qua ngôn ngữ tự nhiên.

### 1. Công nghệ cốt lõi (Core Technology)

*   **Ngôn ngữ & Phân phối:** Được viết bằng **Python (>=3.10)**. Phân phối đa nền tảng dưới dạng gói PyPI (`strands-agents-sops`), MCP Server, Anthropic Skills và Cursor Commands.
*   **Model Context Protocol (MCP):** Sử dụng MCP SDK để biến các file Markdown thành các "Prompts" mà các trình soạn thảo AI (như Claude Code, Kiro CLI) có thể khám phá và sử dụng theo yêu cầu.
*   **Ngôn ngữ hướng dẫn (Instruction Language):** Sử dụng **Markdown** kết hợp với từ khóa **RFC 2119** (MUST, SHOULD, MAY, MUST NOT). Đây là kỹ thuật mượn từ các tiêu chuẩn kỹ thuật Internet để tạo ra các ràng buộc hành vi cực kỳ chính xác cho LLM.
*   **Hệ thống Build:** Sử dụng **Hatch** và `hatch-vcs` để quản lý phiên bản dựa trên Git và đóng gói dữ liệu (copy các file SOP vào trong package Python thông qua build hook).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án thể hiện một tư duy kiến trúc hiện đại về việc "Lập trình cho AI" bằng ngôn ngữ tự nhiên:

*   **Progressive Disclosure (Tiết lộ dần dần):** Thay vì ném toàn bộ hướng dẫn vào System Prompt (gây tốn token và làm loãng ngữ cảnh), kiến trúc này cho phép Claude/AI chỉ tải (load) quy trình cụ thể (SOP) khi cần thiết.
*   **Phân tách giữa Kế hoạch và Thực thi:**
    *   **PDD (Prompt-Driven Development):** Chuyển ý tưởng thô thành thiết kế.
    *   **Code Task Generator:** Chia thiết kế thành các task nhỏ (`.code-task.md`).
    *   **Code Assist:** Thực thi code dựa trên task (theo chuẩn TDD).
*   **Cấu trúc dữ liệu có thứ tự (.agents/ directory):** Tư duy quản lý file rất khoa học. Chia tách rõ ràng giữa những gì nên commit (tóm tắt codebase, kế hoạch thiết kế) và những gì nên bỏ qua (file nháp thực thi - scratchpad).
*   **Tính đa diện (Multi-modal Distribution):** Một file nguồn `.sop.md` duy nhất có thể được "biên dịch" sang nhiều định dạng: MCP Tool, Cursor Command, hoặc Claude Skill. Điều này đảm bảo tính nhất quán (Single Source of Truth).

### 3. Các kỹ thuật chính (Key Techniques)

*   **RFC 2119 Constraints:** Đây là kỹ thuật then chốt. Việc sử dụng các từ khóa như "MUST NOT" kèm theo lý do (Context) giúp AI hiểu không chỉ *cái gì* không được làm mà còn là *tại sao*, từ đó giảm thiểu sự ảo tưởng (hallucination).
*   **Parameterization (Tham số hóa):** SOP không phải là prompt tĩnh. Nó định nghĩa các tham số đầu vào (required/optional) và có cơ chế "Parameter Acquisition" để AI tự biết hỏi người dùng các thông tin còn thiếu trước khi bắt đầu.
*   **Wrapper Pattern:** Trong mã nguồn Python, dự án bao bọc nội dung SOP trong các thẻ XML (`<agent-sop>`, `<content>`, `<user-input>`). Đây là cấu trúc mà Claude cực kỳ ưa thích để phân biệt giữa hướng dẫn hệ thống và dữ liệu người dùng.
*   **Deduplication & Precedence:** Hệ thống nạp (loading) ưu tiên "First-wins". Các SOP tùy chỉnh của người dùng (external SOPs) sẽ ghi đè các SOP mặc định của hệ thống nếu trùng tên, cho phép tùy biến linh hoạt cho từng dự án.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

Luồng hoạt động của một Agent sử dụng SOP diễn ra như sau:

1.  **Kích hoạt:** Người dùng gọi một SOP (ví dụ: gõ `@codebase-summary` trong Kiro hoặc `/code-assist` trong Claude).
2.  **Khởi tạo (Setup):** AI đọc phần `Parameters`. Nếu thiếu thông tin (như `task_description`), AI sẽ dừng lại và yêu cầu người dùng cung cấp một lần duy nhất.
3.  **Giai đoạn Khám phá (Explore):** AI thực hiện các lệnh tìm kiếm (find, grep) để hiểu ngữ cảnh dự án và ghi lại vào file `context.md` trong thư mục `.agents/`.
4.  **Lập kế hoạch (Plan):** AI soạn thảo chiến lược thực hiện và danh sách kiểm tra (checklist) trong file `plan.md`.
5.  **Thực thi (Code/Execute):** AI thực hiện từng bước trong mục `Steps`. Với mỗi bước, nó đối chiếu với các `Constraints` để đảm bảo không vi phạm quy tắc (ví dụ: phải viết test trước khi viết code).
6.  **Ghi nhận (Progress):** Trạng thái công việc được cập nhật liên tục vào `progress.md`. Điều này cho phép quy trình có thể "hồi sinh" (resumability) nếu phiên làm việc bị ngắt quãng.
7.  **Hoàn tất (Commit):** Sau khi Build và Test thành công, AI soạn thảo Commit Message theo chuẩn Conventional Commits và thực hiện commit.

### Tổng kết
**Agent SOP** không chỉ là một tập hợp các prompt, mà là một **hệ điều hành quy trình** cho AI. Nó giải quyết vấn đề lớn nhất của AI hiện nay: **tính không nhất quán (inconsistency)** bằng cách ép AI vào một khung làm việc có kỷ luật, có kiểm tra và có khả năng tái sử dụng.