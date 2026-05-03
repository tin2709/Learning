Dựa trên mã nguồn và cấu trúc thư mục của dự án **Pebble**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và kỹ thuật lập trình của hệ thống này:

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

Dự án sử dụng mô hình Hybrid Desktop App hiện đại, kết hợp hiệu suất của hệ thống (System-level) và tính linh hoạt của Web:

*   **Ngôn ngữ lập trình:** **Rust** (Backend/Core) và **TypeScript/React 19** (Frontend).
*   **Framework Desktop:** **Tauri v2**. Khác với Electron, Tauri sử dụng Webview có sẵn của OS và Backend bằng Rust, giúp giảm dung lượng file thực thi và bộ nhớ RAM.
*   **Cơ sở dữ liệu:** **SQLite** (thông qua `rusqlite`). Đây là lựa chọn chuẩn cho các ứng dụng "Local-first".
*   **Công cụ tìm kiếm:** **Tantivy**. Đây là một thư viện Full-text search engine viết bằng Rust (tương tự Lucene), cho phép tìm kiếm email cực nhanh ngay trên máy cục bộ.
*   **Quản lý trạng thái:** **Zustand** (Frontend) kết hợp với **TanStack Query** để quản lý trạng thái đồng bộ hóa dữ liệu từ Backend.
*   **Giao diện:** **Tailwind CSS** và **Lucide React** (icons).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Pebble được thiết kế theo hướng **Modular Monolith** và **Local-first**:

*   **Local-first (Ưu tiên cục bộ):** Mọi dữ liệu (email, folder, settings, search index) đều được lưu trữ trực tiếp tại máy người dùng. Network chỉ đóng vai trò là "ống dẫn" để lấy dữ liệu về hoặc gửi đi.
*   **Cấu trúc Workspace (Rust Crates):** Dự án chia nhỏ thành các "crate" riêng biệt để dễ quản lý và kiểm thử:
    *   `pebble-core`: Chứa các kiểu dữ liệu chung (Types) và Error handling.
    *   `pebble-mail`: Logic đồng bộ hóa (IMAP, Gmail API, Outlook API).
    *   `pebble-search`: Quản lý chỉ mục tìm kiếm Tantivy.
    *   `pebble-crypto`: Mã hóa thông tin nhạy cảm (Tokens, mật khẩu).
    *   `pebble-privacy`: Xử lý bảo mật HTML (Sanitizer).
*   **Tính đa hình (Trait-based Abstraction):** Sử dụng các `Trait` trong Rust (như `MailProvider`, `MailTransport`) để định nghĩa interface chung. Điều này cho phép hệ thống xử lý Gmail, Outlook hay IMAP theo cùng một cách ở tầng logic cao hơn mà không cần quan tâm đến giao thức cụ thể bên dưới.

---

### 3. Kỹ thuật lập trình chính (Key Techniques)

#### A. Bảo mật và Quyền riêng tư (Privacy & Security)
*   **Credential Encryption:** Sử dụng `pebble-crypto` kết hợp với **OS Keystore** (Keychain trên macOS, Credential Locker trên Windows) thông qua thư viện `keyring` để lưu Master Key, sau đó dùng AES-256-GCM để mã hóa các OAuth Token.
*   **HTML Sanitization:** Sử dụng thư viện `ammonia` và `lol_html` (một parser cực nhanh của Cloudflare) để làm sạch nội dung Email. Kỹ thuật này giúp chặn các tracker (tracking pixel), mã độc JavaScript và các CSS exfiltration.

#### B. Chiến lược đồng bộ hóa (Sync Strategy)
*   **Exponential Backoff:** Trong `backoff.rs`, hệ thống triển khai cơ chế chờ đợi tăng dần nếu gặp lỗi mạng, tránh việc spam request lên mail server khi mất kết nối.
*   **Real-time & Polling:** Kết hợp cơ chế **IDLE** (của IMAP) và **History API** (của Gmail) để nhận thông báo email mới ngay lập tức, thay vì chỉ quét (scan) định kỳ.
*   **Thread Calculation:** Logic tính toán luồng hội thoại (`thread.rs`) dựa trên header `In-Reply-To` và `References` để nhóm các email lại với nhau một cách chính xác ngay tại local.

#### C. Full-text Search
*   **Ngram Tokenizer:** Trong `pebble-search`, hệ thống sử dụng Ngram để hỗ trợ tìm kiếm các từ khóa không hoàn chỉnh (phù hợp với các ngôn ngữ CJK hoặc tìm kiếm địa chỉ email).

---

### 4. Luồng hoạt động hệ thống (System Workflow)

#### Luồng nhận Email (Sync Flow):
1.  **Trigger:** Một sự kiện (Startup, Timer, Focus window, hoặc Push notification) kích hoạt `SyncWorker`.
2.  **Auth:** `pebble-oauth` kiểm tra Access Token. Nếu hết hạn, nó dùng Refresh Token để lấy key mới từ OS Keystore.
3.  **Fetch:** `pebble-mail` kết nối tới Server (Gmail REST API hoặc IMAP over TLS/Proxy).
4.  **Parse:** Dữ liệu thô (MIME) được `mail-parser` phân tích thành cấu trúc: Body text, HTML, Attachments.
5.  **Persist:** 
    *   Thông tin email lưu vào **SQLite**.
    *   File đính kèm lưu vào thư mục **App Data**.
    *   Nội dung text được gửi sang `pebble-search` để lập chỉ mục (indexing).
6.  **Notify:** Backend gửi sự kiện qua Tauri IPC tới Frontend để cập nhật giao diện người dùng (Zustand store cập nhật).

#### Luồng đọc Email (Render Flow):
1.  Người dùng nhấn vào một Email.
2.  Frontend yêu cầu nội dung từ Backend.
3.  Backend lấy HTML thô, chạy qua `PrivacyGuard` (Sanitizer) để:
    *   Chặn ảnh từ các domain tracker.
    *   Chuyển đổi các link ảnh thành placeholder (nếu ở Strict mode).
    *   Làm sạch các tag nguy hiểm.
4.  Frontend nhận HTML "sạch" và hiển thị trong một **Shadow DOM** để cô lập hoàn toàn CSS của email với CSS của ứng dụng.

### Tổng kết
Pebble là một dự án có tư duy kỹ thuật rất tốt, áp dụng đúng các pattern hiện đại của Rust (Safety, Async, Traits) và chú trọng đặc biệt vào hiệu suất cục bộ (Local search, SQLite). Đây là một kiến trúc mẫu mực cho các ứng dụng desktop cần xử lý dữ liệu lớn và bảo mật cao.