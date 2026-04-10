Dựa trên mã nguồn và tài liệu của dự án **OmniRoute (v3.6.1)**, dưới đây là bản phân tích chuyên sâu về hệ thống này theo 4 khía cạnh bạn yêu cầu:

---

### 1. Phân tích Công nghệ cốt lõi (Core Technology)

OmniRoute được xây dựng trên một tập hợp công nghệ hiện đại, tập trung vào hiệu suất và khả năng mở rộng cục bộ:

*   **Runtime & Framework:**
    *   **Node.js (>=18 <24):** Yêu cầu phiên bản cụ thể do các thư viện native (như `better-sqlite3`) chưa tương thích hoàn toàn với Node 24.
    *   **Next.js 16 (App Router):** Sử dụng kiến trúc hiện đại nhất của Next.js để xử lý cả Frontend (Dashboard) và Backend (API Routes).
    *   **Electron:** Cho phép đóng gói toàn bộ hệ thống thành ứng dụng desktop chạy đa nền tảng (Windows, macOS, Linux).
*   **Dữ liệu & Trạng thái:**
    *   **Better-sqlite3:** Một thư viện SQLite cực nhanh cho Node.js, hoạt động ở chế độ đồng bộ để tối ưu hóa IO trên môi trường local. Hệ thống có cơ chế tự động migration (16-20 file SQL).
    *   **Zustand:** Quản lý state ở phía Client nhẹ nhàng và linh hoạt.
*   **Networking & Proxy:**
    *   **Undici:** HTTP client hiệu suất cao cho Node.js, được dùng để thay thế `fetch` mặc định nhằm xử lý các dispatcher proxy phức tạp (SOCKS5/HTTP).
    *   **SSE (Server-Sent Events):** Công nghệ cốt lõi để truyền phát (streaming) câu trả lời từ AI về client theo thời gian thực.
    *   **wreq-js:** Sử dụng để giả lập TLS Fingerprint (Chrome 124), giúp vượt qua các cơ chế chặn bot của các nhà cung cấp như Google/Cloudflare.
*   **Giao thức AI:**
    *   **MCP (Model Context Protocol):** Hỗ trợ 25 công cụ tích hợp cho phép các AI Agent (như Claude Desktop) tương tác trực tiếp với hệ thống.
    *   **A2A Protocol (v0.3):** Giao thức Agent-to-Agent dựa trên JSON-RPC 2.0 để các AI tự điều phối công việc với nhau.

---

### 2. Tư duy Kiến trúc (Architectural Mindset)

Kiến trúc của OmniRoute thể hiện tư duy **"Abstraction Layer"** (Lớp trừu tượng hóa) mạnh mẽ:

*   **Kiến trúc Hub-and-Spoke (Trục và Nan hoa):**
    *   Hệ thống coi định dạng của OpenAI là "ngôn ngữ chung" (Hub). Mọi yêu cầu từ các client (Claude, Gemini, v.v.) đều được đưa về chuẩn OpenAI trước khi chuyển đổi sang định dạng của nhà cung cấp đích (Spoke). Điều này giúp giảm độ phức tạp từ $N \times N$ xuống $N + N$.
*   **Kiến trúc 4 tầng Fallback (Tiered Resilience):**
    *   Tư duy thiết kế ưu tiên chi phí và độ tin cậy: `Subscription (Trả phí tháng) -> API Key (Trả phí theo dùng) -> Cheap (Giá rẻ) -> Free (Miễn phí)`.
*   **Tách biệt Core Proxy (`open-sse`) và Management (`src`):**
    *   `open-sse` là một workspace độc lập xử lý logic "nặng" về streaming và dịch thuật (translation), trong khi `src` xử lý giao diện và quản trị dữ liệu.
*   **Tư duy Offline-First & Local-First:**
    *   Toàn bộ dữ liệu nằm tại máy người dùng (`~/.omniroute`). Cloud Sync chỉ là một tùy chọn phụ trợ để đồng bộ hóa cấu hình qua Cloudflare Workers.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

Mã nguồn OmniRoute sử dụng nhiều kỹ thuật lập trình nâng cao để xử lý luồng dữ liệu AI:

*   **Strategy Pattern (Mẫu chiến thuật):**
    *   Được áp dụng trong các `Executors`. Mỗi nhà cung cấp (Anthropic, Gemini, Cursor, AWS Kiro) có một class Executor riêng kế thừa từ `BaseExecutor` để xử lý logic header, URL và refresh token đặc thù.
*   **Self-Registering Plugins:**
    *   Các bộ dịch (Translators) tự đăng ký vào registry khi module được load (`register("claude", "openai", ...)`). Kỹ thuật này giúp hệ thống cực kỳ dễ mở rộng mà không cần sửa file core.
*   **Streaming Transform Streams:**
    *   Sử dụng `TransformStream` của Web API để xử lý dữ liệu SSE theo từng chunk. Hệ thống có thể can thiệp vào dữ liệu đang stream (ví dụ: bóc tách thẻ `<think>`, tính toán token) mà không làm gián đoạn luồng truyền của client.
*   **Deduplication & Concurrency Control:**
    *   Kỹ thuật `refreshPromiseCache` trong dịch vụ làm mới token giúp ngăn chặn "Thundering Herd Problem" (nhiều request cùng làm mới token một lúc gây lỗi).
*   **Zod-driven Validation:**
    *   Sử dụng Zod không chỉ để validate API input mà còn để định nghĩa schema cho cấu hình nhà cung cấp, đảm bảo lỗi được phát hiện ngay khi ứng dụng khởi chạy.

---

### 4. Luồng hoạt động hệ thống (System Operation Flows)

#### A. Luồng xử lý Chat Request (`/v1/chat/completions`):
1.  **Entry:** Nhận request -> Validate API Key cục bộ.
2.  **Model Resolution:** Kiểm tra xem model là model đơn hay là một `Combo` (chuỗi fallback).
3.  **Account Selection:** Chọn tài khoản (Connection) tốt nhất dựa trên trọng số, độ ưu tiên và trạng thái "Circuit Breaker".
4.  **Translation (Request):** Chuyển đổi payload từ OpenAI format -> Provider format.
5.  **Execution:** Gửi yêu cầu lên Upstream (có kèm TLS spoofing nếu cần).
6.  **Translation (Response):** Nhận stream từ Upstream -> Chuyển đổi ngược về OpenAI format -> Xử lý các tag đặc biệt như `<think>`.
7.  **Usage Tracking:** Trích xuất thông tin sử dụng (token) -> Ghi log vào SQLite -> Trả dữ liệu về Client.

#### B. Luồng Tự phục hồi (Self-healing):
1.  Nếu Upstream trả về lỗi (429 - Rate limit, 401 - Expired):
2.  **Retry logic:** Tự động kích hoạt làm mới token (OAuth refresh).
3.  **Fallback:** Nếu vẫn lỗi, hệ thống tự động nhảy sang tài khoản tiếp theo hoặc model tiếp theo trong danh sách `Combo`.
4.  **Circuit Breaker:** Nếu một nhà cung cấp lỗi quá nhiều lần, hệ thống tạm thời ngắt kết nối đó trong một khoảng thời gian (cooldown).

#### C. Luồng Context Relay (Duy trì ngữ cảnh):
*   Khi một tài khoản sắp hết quota, hệ thống tự động gọi một AI model để tóm tắt (summarize) hội thoại hiện tại.
*   Khi request tiếp theo chuyển sang tài khoản mới, bản tóm tắt này được tiêm (inject) vào dưới dạng `system message` để AI mới hiểu được ngữ cảnh cũ.

Dự án này là một ví dụ điển hình về việc kết hợp giữa kiến trúc phần mềm truyền thống (Proxy/Gateway) với các yêu cầu đặc thù của kỷ nguyên Generative AI.