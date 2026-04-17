Dựa trên mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chuyên sâu về dự án **abtop** — một công cụ giám sát (monitoring) dành cho các AI Agent (Claude Code, Codex CLI) được viết bằng Rust.

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án sử dụng "Modern Rust Stack" dành cho các ứng dụng terminal (TUI):

*   **Ngôn ngữ:** **Rust (Edition 2021)**, tận dụng tính an toàn bộ nhớ và hiệu suất cao để xử lý dữ liệu từ file log lớn mà không gây lag terminal.
*   **Giao diện TUI:** **Ratatui** kết hợp với **Crossterm**. Đây là bộ đôi mạnh mẽ nhất hiện nay trong hệ sinh thái Rust để xây dựng giao diện terminal có màu sắc, biểu đồ (sparklines), và bảng biểu.
*   **Xử lý dữ liệu:** **Serde & Serde_json**. Dự án xử lý dữ liệu chủ yếu từ định dạng **JSONL (JSON Lines)** — một định dạng phổ biến cho log vì cho phép append dữ liệu liên tục mà không cần parse lại toàn bộ file.
*   **Tương tác hệ thống:** Sử dụng lệnh Unix tiêu chuẩn thông qua `std::process::Command` như `ps` (quản lý tiến trình) và `lsof` (quản lý port/network).
*   **Đồ họa Terminal:** Kỹ thuật **Braille symbols** (ký tự Braille) để vẽ biểu đồ mật độ cao (token rate) trong không gian giới hạn của một dòng văn bản.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của `abtop` tuân theo mô hình **Collector-App-UI**:

*   **Thiết kế "Passive Observer" (Người quan sát thụ động):** Abtop không can thiệp vào API Key hay luồng chạy của AI Agent. Nó hoạt động như một "sidecar", chỉ đọc các file lưu tạm (session files), log (transcripts), và trạng thái tiến trình (PID) của hệ điều hành. Điều này đảm bảo tính bảo mật và không làm chậm Agent.
*   **Tính module hóa (Strategy Pattern):** Thông qua `trait AgentCollector`, dự án tách biệt logic thu thập dữ liệu của Claude và Codex. Việc thêm một Agent mới (như Gemini hay Cursor) chỉ cần triển khai trait này mà không làm thay đổi logic hiển thị.
*   **Hệ thống ưu tiên hiển thị (Responsive TUI):** Kiến trúc UI định nghĩa mức độ ưu tiên: Panel "Sessions" luôn hiển thị, trong khi các panel như "Context" hay "Quota" sẽ tự ẩn đi nếu kích thước màn hình quá nhỏ.
*   **Defense in Depth (Phòng thủ đa tầng):** Trong việc phát hiện trạng thái (Working/Waiting), hệ thống không tin vào một nguồn duy nhất. Nó kết hợp giữa: *File modification time* (mtime), *CPU Usage*, và sự tồn tại của *PID*.

---

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **Incremental File Reading (Đọc file lũy tiến):** Đây là kỹ thuật quan trọng nhất. Thay vì đọc lại file JSONL (có thể lên tới 18MB) mỗi 2 giây, `abtop` lưu lại `file_offset`. Ở mỗi chu kỳ, nó chỉ đọc từ vị trí offset cũ đến cuối file. Nếu file bị xoay (rotated) hoặc thu nhỏ, nó sẽ tự động reset về 0.
*   **Process Tree Mapping:** Abtop đi xuyên suốt cây tiến trình từ PID gốc của Agent để tìm các tiến trình con (như `npm test`, `cargo build`). Nó sử dụng `ppid` (parent process ID) để xây dựng bản đồ quan hệ và gán các port đang lắng nghe vào đúng Session tương ứng.
*   **Background Worker & MPSC Channels:** Việc tạo tóm tắt session (Summary Generation) yêu cầu gọi lệnh `claude --print`. Để tránh làm treo giao diện TUI, `abtop` đẩy việc này vào một thread riêng và dùng channel (`std::sync::mpsc`) để gửi kết quả về loop chính khi hoàn thành.
*   **Gradient & Theme Engine:** Dự án mô phỏng chính xác hệ thống màu của `btop` bằng cách nội suy (interpolation) màu RGB qua 101 bước. Điều này tạo ra các biểu đồ màu chuyển tiếp mượt mà từ xanh lá (an toàn) sang đỏ (nguy hiểm/sắp hết quota).

---

### 4. Luồng hoạt động của hệ thống (System Flow)

Luồng chạy của `abtop` là một vòng lặp sự kiện (Event Loop) liên tục:

1.  **Giai đoạn Khởi tạo (Discovery):**
    *   Quét thư mục cấu hình (`~/.claude/sessions` hoặc `~/.codex/sessions`).
    *   Xác định các PID đang sống và ánh xạ chúng với các file transcript tương ứng trên ổ đĩa.

2.  **Giai đoạn Thu thập (Collection Tick - mỗi 2s):**
    *   **Fast Poll (2s):** Đọc thêm dữ liệu từ transcript, tính toán tốc độ token (delta), kiểm tra trạng thái CPU.
    *   **Slow Poll (10s):** Chạy `lsof` để tìm port mới, chạy `git status` để xem thay đổi mã nguồn trong dự án.
    *   **Orphan Detection:** Nếu một PID con vẫn mở port nhưng PID cha (Agent) đã chết, đánh dấu đó là "Orphan Port".

3.  **Giai đoạn Xử lý trạng thái (State Update):**
    *   Cập nhật `App struct` với danh sách session mới nhất.
    *   Tính toán tỷ lệ phần trăm Context Window dựa trên các model hardcoded (vì Agent không cung cấp thông tin này trực tiếp).

4.  **Giai đoạn Hiển thị (Rendering):**
    *   Ratatui lấy dữ liệu từ `App` và vẽ các panel.
    *   Sử dụng bộ nhớ đệm (Cache) cho các tóm tắt session để tránh gọi LLM quá nhiều lần.

5.  **Tương tác người dùng:**
    *   Nếu chạy trong **tmux**, phím `Enter` sẽ gửi lệnh tới tmux server để nhảy thẳng (switch-client/select-pane) tới cửa sổ terminal đang chạy Agent đó.

### Tổng kết
`abtop` là một ví dụ điển hình về việc sử dụng Rust để tạo ra công cụ CLI hiệu năng cao. Nó kết hợp khéo léo giữa việc **parsing dữ liệu thô từ file log** và **truy vấn trạng thái hệ thống thấp cấp** để mang lại trải nghiệm quan sát toàn diện cho các AI Coding Agent.