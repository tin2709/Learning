Dựa trên các tệp tin cấu trúc và mã nguồn của dự án **Domain Locker**, dưới đây là phân tích chi tiết:

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án này sử dụng một ngăn xếp công nghệ (stack) hiện đại, kết hợp giữa sự ổn định của Angular và tốc độ của các công cụ build mới:

*   **Meta-Framework:** **Analog.js**. Đây là một framework xây dựng trên Angular (tương tự như Next.js cho React), sử dụng **Vite** để build và **Nitro** làm server engine.
*   **Frontend:**
    *   **Angular 19**: Phiên bản mới nhất của Angular.
    *   **PrimeNG**: Bộ thư viện UI component chính (Table, Charts, Organization Chart, v.v.).
    *   **Tailwind CSS**: Dùng cho thiết kế giao diện tùy chỉnh và tiện ích.
    *   **Data Visualization**: **ApexCharts** (biểu đồ lịch sử, thời hạn), **D3.js** (word cloud), **Leaflet** (bản đồ vị trí máy chủ).
*   **Backend (API Routes):** Sử dụng các **API Routes của Analog.js** (chạy trên môi trường Node.js/Nitro), viết bằng TypeScript.
*   **Database:** Hỗ trợ linh hoạt hai chế độ:
    *   **PostgreSQL**: Cho các bản tự cài đặt (Self-hosted).
    *   **Supabase**: Cho phiên bản quản lý (Managed/Cloud).
*   **Infrastructure:** Docker, Docker Compose và Helm Chart (cho Kubernetes).

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Hybrid Database Layer:** Kiến trúc sử dụng **Adapter Pattern** trong thư mục `services/db-query-services/`. Hệ thống có thể chuyển đổi mượt mà giữa `pg-database.service.ts` (Postgres thuần) và `sb-database.service.ts` (Supabase) dựa trên biến môi trường.
*   **Full-stack Monolith (tiếp cận hiện đại):** Cả frontend và backend API đều nằm trong một dự án duy nhất nhờ Analog.js, giúp đồng bộ hóa kiểu dữ liệu (TypeScript types) giữa client và server dễ dàng.
*   **Kiến trúc hướng Module:** Các tính năng được chia nhỏ thành các component độc lập (`domain-card`, `domain-list`, `domain-info`), giúp tái sử dụng mã nguồn cho các view khác nhau (Grid, List, Details).
*   **Environment-Driven (Cấu hình theo môi trường):** Phân chia rõ rệt giữa 3 chế độ: `selfHosted` (không cần auth phức tạp, dùng 1 static user ID), `demo` (chế độ chỉ xem), và `managed` (có đầy đủ Auth/MFA qua Supabase).

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Reactive Programming với RxJS:** Sử dụng rộng rãi `Observable` để xử lý dòng dữ liệu từ database và các sự kiện người dùng, đặc biệt là trong các Service xử lý logic nghiệp vụ.
*   **SSR & SSG:** Analog.js cho phép **Server-Side Rendering** (tăng tốc độ tải trang đầu) và **Static Site Generation** (cho các trang tài liệu/docs trong thư mục `src/content/docs`).
*   **Dynamic Theming:** Kỹ thuật tải các tệp CSS của PrimeNG một cách năng động (`purple-dark.css`, `orange-light.css`...) cho phép người dùng đổi chủ đề (Light/Dark/Custom) ngay lập tức mà không cần build lại app.
*   **Triggers & Cron Tasks:** Thay vì chạy một worker nền phức tạp bên trong Node.js, dự án sử dụng một container `updater` riêng biệt chạy **Crontab**. Container này gọi định kỳ vào các API endpoint (`/api/domain-updater`, `/api/domain-monitor`) để thực hiện các tác vụ nặng như quét WHOIS, DNS, SSL.
*   **Tích hợp đa nguồn dữ liệu:** Sử dụng các thư viện như `whois-json`, `pg`, `marked` (để render tài liệu từ Markdown).

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Khởi tạo (Startup Flow):**
    *   Khi chạy Docker, `start.sh` kiểm tra các biến môi trường.
    *   Nếu là Postgres, nó đợi DB sẵn sàng, sau đó áp dụng `schema.sql` để khởi tạo cấu trúc bảng.
    *   Server Analog/Nitro bắt đầu lắng nghe tại cổng 3000.

2.  **Quản lý tên miền (Domain Management Flow):**
    *   **Người dùng** thêm tên miền qua giao diện -> **API** lưu vào DB.
    *   **Hệ thống** tự động lấy Favicon (`favicon.component.ts`) và trích xuất TLD.
    *   **Background Cron** (từ container `updater`) định kỳ kích hoạt lệnh quét.
    *   **Server logic** thực hiện: `lookup DNS` -> `check SSL` -> `parse WHOIS` -> `check Uptime`.
    *   Dữ liệu mới được so sánh với dữ liệu cũ (`compare.ts`), nếu có thay đổi sẽ ghi vào bảng `domain_updates` và gửi thông báo qua Webhook/Email.

3.  **Hiển thị và Phân tích (Visualization Flow):**
    *   UI gọi các query dịch vụ để lấy dữ liệu tổng hợp.
    *   Dữ liệu được đẩy vào các biểu đồ:
        *   `registration-lifespan`: Vẽ sơ đồ Gantt về vòng đời tên miền.
        *   `year-calendar`: Hiển thị các sự kiện hết hạn theo tháng.
        *   `host-map`: Hiển thị vị trí vật lý của server trên bản đồ thế giới.

### Tổng kết
**Domain Locker** là một công cụ quản lý portfolio tên miền rất chuyên nghiệp. Nó không chỉ là một dashboard hiển thị mà còn là một hệ thống **giám sát chủ động**. Việc sử dụng Analog.js giúp dự án tận dụng được sức mạnh của hệ sinh thái Angular nhưng vẫn có sự linh hoạt và hiệu suất build của Vite. Khả năng chạy được cả trên Postgres truyền thống lẫn Supabase giúp nó cực kỳ linh hoạt cho cả người dùng cá nhân lẫn doanh nghiệp.