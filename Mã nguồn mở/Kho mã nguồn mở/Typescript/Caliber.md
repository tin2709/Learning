Chào bạn, đây là phân tích chi tiết về dự án **Caliber**, một công cụ CLI mạnh mẽ giúp tối ưu hóa cấu hình cho các AI Coding Agent (như Claude Code, Cursor, Codex) dựa trên phân tích mã nguồn.

---

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ & Runtime:** Sử dụng **TypeScript (99.4%)** chạy trên nền **Node.js (>=20)**. Mã nguồn được đóng gói bằng **tsup** để hỗ trợ ESM (ECMAScript Modules).
*   **Giao diện dòng lệnh (CLI):**
    *   `commander`: Xử lý hệ thống lệnh và tham số.
    *   `inquirer` (@inquirer/select, checkbox, confirm): Xây dựng giao diện tương tác (interactive prompt).
    *   `chalk` & `ora`: Tạo màu sắc và hiệu ứng loading (spinners) chuyên nghiệp.
*   **Tầng LLM (AI Layer):**
    *   Tích hợp đa nền tảng: `@anthropic-ai/sdk`, `openai`, `@anthropic-ai/vertex-sdk` (Google Cloud).
    *   Khả năng tận dụng "ghế" (seat) có sẵn từ **Claude Code CLI** và **Cursor ACP** (JSON-RPC) mà không cần API Key riêng.
*   **Quản lý dữ liệu & Kiểm thử:**
    *   `vitest`: Framework kiểm thử hiệu năng cao.
    *   `posthog-node`: Thu thập dữ liệu sử dụng (telemetry) ẩn danh để cải thiện sản phẩm.
    *   `diff`: So sánh sự thay đổi giữa các phiên bản cấu hình.

---

### 2. Tư duy Kiến trúc (Architectural Mindset)

Kiến trúc của Caliber được xây dựng theo mô hình **Provider-Agnostic** và **Deterministic-First**:

*   **Tầng LLM trừu tượng (src/llm):** Một giao diện chung duy nhất cho tất cả các nhà cung cấp AI. Hệ thống tự động điều phối giữa các model "nặng" (để generate) và model "nhẹ" (để phân loại/score) nhằm tối ưu chi phí và tốc độ.
*   **Fingerprinting (Dấu vân tay mã nguồn):** Thay vì đọc toàn bộ code (tốn token), Caliber phân tích cấu trúc cây thư mục, phân bổ đuôi mở rộng file (`.ts`, `.py`, ...) và `package.json` để hiểu stack công nghệ của dự án.
*   **Kiến trúc Plugin-based cho Agent:** Mỗi AI Agent (Claude, Cursor, Codex) có một "Writer" riêng (`src/writers/`) để đảm bảo định dạng đầu ra (Markdown, MDC, JSON) tuân thủ đúng đặc tả của Agent đó.
*   **Hệ thống Chấm điểm Định tính (Deterministic Scoring):** Caliber không dựa vào LLM để đánh giá cấu hình. Nó sử dụng logic cứng (hệ thống file thực tế) để kiểm tra xem các đường dẫn trong file config có tồn tại không, mật độ backtick có đủ cao không, v.v.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Two-Tier Model System:** Kỹ thuật dùng 2 cấp độ model. Các tác vụ nhẹ như phân loại lệnh hay chấm điểm nhanh được gửi tới model Haiku/GPT-4o-mini để tiết kiệm.
*   **Smart Context Sampling:** Trong các repo lớn, Caliber sử dụng thuật toán `sampleFileTree` để ưu tiên các file quan trọng nhất (theo mtime, cấu trúc thư mục) nhằm đưa vào ngữ cảnh (context window) mà không làm tràn bộ nhớ LLM.
*   **Session Learning (Capture & Distill):** Sử dụng các Git hooks và SessionEnd hooks để bắt các lỗi mà AI hay mắc phải trong quá trình làm việc, sau đó "chưng cất" (distill) thành các bài học mới ghi vào `CALIBER_LEARNINGS.md`.
*   **Parallel Execution Engine:** Quá trình phát hiện stack, generate config và search cộng đồng diễn ra song song để giảm thời gian chờ đợi của người dùng xuống mức tối thiểu.
*   **Auto-Refinement (Self-Healing):** Sau khi generate, hệ thống tự động chạy `caliber score`. Nếu điểm thấp, nó sẽ gửi phản hồi ngược lại cho LLM để sửa lỗi (ví dụ: đường dẫn file sai) cho đến khi đạt điểm tối ưu.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng hoạt động của lệnh chính `caliber init` diễn ra qua 4 giai đoạn:

1.  **Setup (Cài đặt):** Xác định nhà cung cấp LLM (API Key hoặc Seat). Người dùng chọn loại Agent mục tiêu (Claude Code/Cursor/Codex).
2.  **Engine (Phân tích & Tạo):**
    *   **Scan:** Lấy vân tay dự án (ngôn ngữ, framework).
    *   **Generate:** LLM tạo ra các file `CLAUDE.md`, `.cursor/rules/`, v.v.
    *   **Parallel Tasks:** Tìm kiếm các "skills" từ cộng đồng phù hợp với stack (ví dụ: skill cho Zod, Vitest).
    *   **Validation:** Hệ thống chấm điểm tự động kiểm tra lỗi logic trong config.
3.  **Review (Đánh giá):** Hiển thị Diff (sự thay đổi) cho người dùng. Người dùng có thể sử dụng tính năng **Chat-Based Refinement** để yêu cầu AI chỉnh sửa lại cấu hình bằng ngôn ngữ tự nhiên.
4.  **Finalize (Hoàn tất):**
    *   Ghi file vào đĩa (có sao lưu backup tại `.caliber/backups/`).
    *   Cài đặt Git hooks để tự động cập nhật cấu hình khi code thay đổi (lệnh `refresh`).
    *   Kích hoạt Session Learning để AI thông minh dần theo thời gian làm việc với dự án.

---
**Tổng kết:** Caliber là một công cụ "Meta-Agent" — nó là AI dùng để huấn luyện và thiết lập môi trường cho các AI khác, giúp giải quyết vấn đề cấu hình thủ công rườm rà và dễ sai sót trong kỷ nguyên AI Coding.