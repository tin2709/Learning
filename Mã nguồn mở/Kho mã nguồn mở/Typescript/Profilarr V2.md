Dựa trên tài liệu và mã nguồn bạn cung cấp cho dự án **Profilarr V2**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và kỹ thuật lập trình của hệ thống này:

---

### 1. Công nghệ cốt lõi (Core Technologies)

Profilarr V2 là một ví dụ điển hình về việc sử dụng các công nghệ hiện đại, chú trọng vào hiệu suất và tính an toàn:

*   **Runtime:** **Deno 2.x**. Lựa chọn này mang lại sự bảo mật mặc định (permissions), hỗ trợ TypeScript nguyên bản và hiệu suất cao hơn Node.js truyền thống.
*   **Frontend Framework:** **SvelteKit** với **Svelte 5** (phiên bản không dùng runes). Svelte nổi tiếng với việc biên dịch mã nguồn trước (compiled), giúp bundle nhẹ và chạy cực nhanh.
*   **Ngôn ngữ:** **TypeScript** là ngôn ngữ chính cho cả backend và frontend, đảm bảo an toàn về kiểu dữ liệu (type-safety).
*   **Cơ sở dữ liệu (App DB):** **SQLite** kết hợp với **Kysely** (một type-safe query builder). Đây là lựa chọn thông minh cho một ứng dụng tự chạy (self-hosted) vì không cần setup server DB phức tạp nhưng vẫn có sức mạnh của SQL.
*   **Parser Service:** Một microservice viết bằng **C# (.NET 8)**. Điều này rất đặc biệt: mục tiêu là để tái hiện chính xác logic phân tích tên phim/series (parsing) mà Radarr và Sonarr (vốn viết bằng .NET) đang sử dụng.
*   **Styling:** **Tailwind CSS**, giúp xây dựng giao diện nhanh và nhất quán.
*   **Infrastructure:** **Docker & Docker Compose**, hỗ trợ triển khai đa nền tảng (amd64, arm64).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Profilarr rất độc đáo, tập trung vào khái niệm **PCD (Profilarr Config Database)**:

*   **Kiến trúc dựa trên Operations (Ops-based):** Thay vì lưu trữ trạng thái cuối cùng của cấu hình, Profilarr lưu trữ một chuỗi các "thao tác" (Ops) nối tiếp nhau (append-only). Các thao tác này được replay để tạo ra trạng thái hiện tại.
*   **Cấu trúc phân lớp (Layered Configuration):** Cấu hình được xây dựng từ 4 lớp:
    1.  **Schema:** Định nghĩa cấu trúc dữ liệu.
    2.  **Base:** Các cấu hình chuẩn (từ Dictionarry, TRaSH Guides...).
    3.  **Tweaks:** Các chỉnh sửa nhỏ tại repo.
    4.  **User:** Tùy chỉnh riêng của người dùng (không bao giờ bị ghi đè khi cập nhật lớp Base).
*   **In-memory Compilation:** Để truy vấn nhanh, hệ thống "biên dịch" (compile) tất cả các Ops vào một SQLite database chạy hoàn toàn trên RAM. Mọi thao tác đọc/validate đều diễn ra trên cache này.
*   **Stable Keys thay vì Auto-ID:** Các thực thể (như Quality Profile) được định danh bằng tên (name) hoặc khóa phức hợp thay vì ID tự tăng. Điều này giúp đồng bộ hóa giữa Profilarr và các instance Radarr/Sonarr (vốn có hệ thống ID riêng) trở nên khả thi.
*   **Kiến trúc "Stupidity Mitigation":** Tư duy bảo mật "phòng thủ chiều sâu". Hệ thống giả định người dùng có thể vô tình lộ API key, nên nó thực hiện hashing API key và "Secret stripping" (xóa sạch bí mật) trong các bản backup và log.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Value Guards (Bảo vệ giá trị):** Khi cập nhật dữ liệu, hệ thống sử dụng kỹ thuật kiểm tra giá trị cũ (guards). Nếu dữ liệu ở "thượng nguồn" (upstream) đã thay đổi, thao tác Update sẽ thất bại (rowcount = 0), từ đó phát hiện xung đột (conflict).
*   **Atomic Job Claiming:** Hệ thống Job sử dụng một cơ chế "claim" nguyên tử trong SQLite. Dispatcher sẽ tìm Job đến hạn và đổi trạng thái từ `queued` sang `running` trong một câu lệnh Update duy nhất để tránh việc hai tiến trình chạy cùng một Job.
*   **Type-Safe API với OpenAPI:** Sử dụng quy trình "Contract-first". Định nghĩa OpenAPI spec trước, sau đó generate TypeScript types cho cả Client và Server, giúp giảm thiểu sai sót khi giao tiếp giữa UI và Backend.
*   **Sanitization & XSS Prevention:** Hệ thống có bộ lọc Sanitize HTML tùy chỉnh cho Markdown. Nó không chỉ xóa thẻ `<script>` mà còn giải mã thực thể HTML để ngăn chặn các kỹ thuật bypass XSS phức tạp.
*   **Modular UI Library:** Thư viện UI (`src/lib/client/ui`) được chia nhỏ thành các atomic components (Button, Badge, Modal, FormInput...) giúp dễ dàng bảo trì và tái sử dụng.

---

### 4. Luồng hoạt động hệ thống (System Operation Flows)

*   **Luồng Khởi tạo (Startup Flow):**
    1.  Đọc biến môi trường & tạo thư mục cấu hình.
    2.  Khởi tạo App DB & chạy Migrations để cập nhật Schema.
    3.  Khôi phục các Job bị kẹt (nếu server crash lần trước).
    4.  Bắt đầu Dispatcher của hệ thống Job.
*   **Luồng Đồng bộ (Sync Flow):**
    1.  Đọc các Ops từ DB.
    2.  Compile vào In-memory Cache.
    3.  Chuyển đổi (Transform) dữ liệu từ định dạng PCD sang Payload API của Radarr/Sonarr.
    4.  Đẩy (Push) qua HTTP client với cơ chế retry/backoff.
    5.  Ghi lại lịch sử và gửi thông báo (Discord, Telegram...).
*   **Luồng Bảo mật (Auth Flow):**
    1.  Middleware kiểm tra IP (Local Bypass?).
    2.  Kiểm tra API Key (`X-Api-Key`).
    3.  Kiểm tra Session (Sliding Expiration - tự động gia hạn nếu user vẫn đang hoạt động).
    4.  Xử lý OIDC (nếu cấu hình) qua các bước trao đổi Token an toàn.

### Tổng kết
Profilarr V2 không chỉ là một công cụ quản lý cấu hình thông thường mà là một **nền tảng quản lý trạng thái (State Management Platform)**. Hệ thống kết hợp nhuần nhuyễn giữa tính linh hoạt của Git (qua PCD) và tính nghiêm ngặt của SQL để giải quyết bài toán đồng bộ hóa cấu hình phức tạp trong hệ sinh thái "Arr".