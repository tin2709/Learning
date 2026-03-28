Dưới đây là bản phân tích chi tiết về dự án **Gryph**, một lớp bảo mật và quan sát (observability) dành cho các AI Coding Agent (như Claude Code, Cursor, Windsurf...).

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án được xây dựng bằng ngôn ngữ **Go**, tập trung vào hiệu suất cao, khả năng đóng gói thành file binary duy nhất và tính bảo mật:

*   **Ngôn ngữ:** **Go (Golang) 1.25+**. Tận dụng tính hiện đại của Go để xử lý đồng thời các luồng log từ nhiều Agent.
*   **Lưu trữ dữ liệu:** **SQLite** thông qua driver thuần Go (`modernc.org/sqlite`). Đây là lựa chọn tối ưu cho mô hình "Local-only", không cần máy chủ trung tâm.
*   **ORM:** **Ent (entgo.io)**. Một thư viện Entity Framework mạnh mẽ cho Go, giúp định nghĩa schema dưới dạng code và tự động tạo mã nguồn (code generation) cho các truy vấn phức tạp.
*   **CLI & Config:** **Cobra** (xây dựng giao diện dòng lệnh) và **Viper** (quản lý cấu hình). Đây là bộ đôi tiêu chuẩn trong hệ sinh thái Go.
*   **Giao diện người dùng (TUI):** **Bubble Tea** và **Lip Gloss** (từ Charmbracelet). Giúp xây dựng các bảng dashboard và giao diện tương tác ngay trên Terminal rất hiện đại và mượt mà.
*   **Xử lý Diff:** **go-difflib**, dùng để tạo ra các bản so sánh trước và sau khi AI chỉnh sửa file.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Gryph tuân thủ nguyên tắc **"Hook & Adapter"** để có thể mở rộng tới mọi AI Agent trên thị trường:

*   **Adapter Pattern (Mẫu thiết kế thích ứng):** Thư mục `agent/` chứa các adapter riêng cho từng Agent (Claude, Cursor, Gemini...). Mỗi adapter chịu trách nhiệm parse định dạng JSON đặc thù của Agent đó thành một mô hình sự kiện chung (`core/events`).
*   **Decoupled Architecture (Kiến trúc tách biệt):**
    *   `core/`: Chứa các model nghiệp vụ (Event, Session, Security). Đây là tầng ổn định nhất.
    *   `agent/`: Tầng giao tiếp với thế giới bên ngoài (các AI Agent).
    *   `storage/`: Tầng trừu tượng hóa việc lưu trữ.
    *   `cli/`: Tầng giao tiếp với người dùng.
*   **Local-First & Zero-Cloud:** Kiến trúc loại bỏ hoàn toàn việc gửi dữ liệu ra ngoài. Toàn bộ quá trình phân tích sự nhạy cảm (privacy check) và lưu trữ đều nằm trên máy của lập trình viên.
*   **Schema-driven Data:** Sử dụng Ent giúp Gryph có một cấu trúc dữ liệu chặt chẽ, dễ dàng thực hiện các truy vấn phức tạp (như tìm tất cả lệnh shell chạy trong một tuần qua).

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Hook Injection (Tiêm mã Hook):** Gryph thực hiện cài đặt bằng cách sửa đổi các file cấu hình của Agent (ví dụ: `~/.claude/settings.json`). Nó đăng ký chính mình như một "PreToolUse" hoặc "PostToolUse" hook để chặn (intercept) dữ liệu trước và sau khi AI thực hiện hành động.
*   **Deterministic Session IDs:** Sử dụng UUID v5 (SHA1-based) để tạo ID phiên làm việc cố định dựa trên dữ liệu từ Agent. Điều này giúp gộp các sự kiện rải rác vào đúng một Session duy nhất dù Agent có thể không cung cấp ID liên tục.
*   **Privacy & Redaction (Lọc quyền riêng tư):** Sử dụng các biểu thức chính quy (Regex) và mẫu đường dẫn để tự động phát hiện các file nhạy cảm (`.env`, `*.key`). Gryph sẽ log hành động (AI đã đọc file này) nhưng từ chối lưu nội dung file để đảm bảo an toàn.
*   **Lazy Loading cho Adapter:** Các adapter chỉ được nạp vào bộ nhớ khi cần thiết, giúp CLI khởi động cực nhanh dù hỗ trợ rất nhiều loại Agent khác nhau.
*   **Code Generation:** Tận dụng tối đa khả năng tạo mã tự động của Go cho ORM (Ent) và JSON Schema, đảm bảo tính nhất quán giữa code và tài liệu định dạng dữ liệu.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng dữ liệu di chuyển từ lúc AI hoạt động đến lúc người dùng xem log:

1.  **Giai đoạn Đăng ký (Install):** `gryph install` quét máy tính, tìm các Agent đã cài, sau đó ghi các lệnh hook của Gryph vào file cấu hình của Agent đó.
2.  **Giai đoạn Ghi nhận (Capturing):**
    *   AI Agent chuẩn bị thực hiện một hành động (ví dụ: `write_file`).
    *   Cơ chế Hook của Agent gọi lệnh: `gryph _hook <agent-name> <hook-type>`.
    *   Gryph nhận dữ liệu thô qua stdin dưới dạng JSON.
    *   Adapter tương ứng parse dữ liệu, thực hiện kiểm tra Privacy, Redact thông tin nhạy cảm.
    *   Dữ liệu được lưu vào SQLite.
3.  **Giai đoạn Phân tích (Querying):**
    *   Người dùng chạy `gryph logs` hoặc `gryph query`.
    *   CLI đọc từ SQLite, sử dụng ORM Ent để lọc dữ liệu.
    *   Giao diện TUI (Bubble Tea) render kết quả dưới dạng bảng hoặc biểu đồ thống kê chi phí/token.

---

### Tổng kết
**Gryph** là một công cụ bảo mật "phòng thủ chiều sâu" cho kỷ nguyên AI Agent. Nó không cố gắng ngăn cản AI (vốn rất khó), mà tập trung vào việc **Tạo ra vết tích kiểm toán (Audit Trail)** minh bạch, giúp lập trình viên kiểm soát được những gì "hộp đen" AI đang thực hiện trên máy tính của mình.