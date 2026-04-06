Dựa trên mã nguồn của dự án **lazyagent**, dưới đây là phân tích chi tiết về kiến trúc, công nghệ và kỹ thuật lập trình của hệ thống này:

### 1. Tư duy Kiến trúc (Architecture Thinking)

Dự án đi theo triết lý **"Observation over Integration"** (Quan sát thay vì tích hợp sâu). Thay vì yêu cầu các Agent (Claude, Cursor...) cung cấp API, `lazyagent` tự tìm kiếm và "đào bới" dữ liệu từ các file log (JSONL) và cơ sở dữ liệu nội bộ (SQLite) của chúng.

*   **Kiến trúc Đa giao diện (Triple-Interface Architecture):** Một nhân (`internal/core`) phục vụ cho 3 loại giao diện:
    *   **TUI:** Cho lập trình viên thích dòng lệnh (Terminal).
    *   **GUI:** macOS Menu Bar cho người dùng muốn giám sát nhanh.
    *   **API:** Cho các tích hợp bên thứ ba hoặc mở rộng tính năng.
*   **Tính trừu tượng (Abstraction):** Sử dụng interface `SessionProvider` để đồng nhất hóa dữ liệu từ nhiều nguồn khác nhau. Dù dữ liệu gốc là SQLite (Cursor) hay JSONL (Claude), chúng đều được chuyển đổi về một cấu trúc `model.Session` chung.
*   **Kiến trúc hướng sự kiện (Event-Driven):** Sử dụng cơ chế File Watcher để kích hoạt việc cập nhật UI thay vì poll liên tục (giảm tải CPU).

### 2. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ chính:** **Go 1.25+** (Backend) và **TypeScript/Svelte 5** (Frontend).
*   **Giao diện TUI:** Bộ thư viện của **Charmbracelet** (`bubbletea` cho quản lý luồng, `lipgloss` cho styling). Đây là tiêu chuẩn hiện đại nhất cho các ứng dụng terminal hiện nay.
*   **Giao diện GUI:** **Wails v3** (alpha). Wails cho phép viết ứng dụng Desktop bằng Go và Web công nghệ cao (Svelte 5 + Tailwind 4), giúp giao diện mượt mà và nhẹ hơn so với Electron.
*   **Xử lý thời gian thực:** **SSE (Server-Sent Events)** trong API server giúp đẩy dữ liệu cập nhật từ backend xuống client ngay lập tức.
*   **Quản lý cấu hình:** Lưu trữ tại `~/.config/lazyagent/config.json`, tự động khởi tạo và cập nhật.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Lazy Parsing (Xử lý file lớn):** 
    Trong `internal/claude/jsonl.go`, lập trình viên sử dụng `json.RawMessage`. Kỹ thuật này cho phép Go đọc các dòng JSONL nhưng không giải mã (unmarshal) toàn bộ các trường ngay lập tức. Hệ thống chỉ parse sâu vào các trường cần thiết (như `role` hoặc `usage`), giúp tiết kiệm bộ nhớ khi file log lên đến hàng chục MB.
*   **Mtime-based Caching (Bộ nhớ đệm thông minh):**
    Mỗi Agent Provider đều có cơ chế cache dựa trên thời gian sửa đổi file (`mtime`). Nếu file không thay đổi, hệ thống sẽ lấy dữ liệu từ cache thay vì đọc lại file, giúp tối ưu hiệu năng I/O.
*   **State Machine (Quản lý trạng thái hoạt động):**
    File `internal/core/activity.go` chứa logic quan trọng nhất để xác định Agent đang làm gì. Nó sử dụng một bộ đếm thời gian (Grace periods) để tránh trạng thái "nhấp nháy". Ví dụ: nếu Claude vừa xong việc nhưng chưa quá 10 giây, trạng thái "waiting" sẽ được giữ lại để tránh việc UI thay đổi quá nhanh.
*   **Cost Estimation (Ước tính chi phí):**
    Hệ thống tự định nghĩa các bảng giá cho các model (Claude, GPT-4, Gemini) để tính toán chi phí dựa trên token đầu vào/đầu ra, ngay cả khi Agent không cung cấp thông tin giá cả.

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng dữ liệu di chuyển theo các bước sau:

1.  **Discovery (Khám phá):** Khi khởi động, các Provider sẽ quét các đường dẫn mặc định như `~/.claude/projects` hoặc `~/Library/Application Support/Cursor/...`.
2.  **Watching (Theo dõi):** `internal/core/watcher.go` sử dụng `fsnotify` để lắng nghe mọi thay đổi trên các file log hoặc file SQLite WAL (Write-Ahead Logging).
3.  **Processing (Xử lý):**
    *   File JSONL được đọc theo dòng.
    *   Dữ liệu được lọc qua các bộ lọc: `window_minutes` (chỉ xem các session gần đây), `searchQuery` (tìm theo tên dự án).
    *   Trạng thái hoạt động (`thinking`, `writing`, `reading`) được tính toán dựa trên `ToolCall` cuối cùng.
4.  **Distribution (Phân phối):** Dữ liệu sau xử lý được đẩy lên:
    *   **TUI:** Thông qua kênh `tea.Msg` để cập nhật bảng giao diện.
    *   **Tray:** Thông qua Wails events để Svelte render lại DOM.
    *   **API:** Thông qua SSE channel cho các client đang lắng nghe `/api/events`.

### Tổng kết
Kiến trúc của `lazyagent` rất tinh gọn và thực dụng. Điểm mạnh nhất là khả năng **tận dụng tối đa bộ nhớ đệm và xử lý stream JSON** để đảm bảo ứng dụng luôn chạy mượt mà dù phải giám sát hàng chục Agent cùng lúc. Việc kết hợp giữa Go (hiệu năng backend) và Svelte 5 (hiệu năng frontend) cho thấy một tư duy chọn lựa stack công nghệ rất hiện đại.