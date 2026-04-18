Dựa trên cấu trúc thư mục và nội dung mã nguồn của dự án **Ziit**, dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống theo dõi thời gian lập trình này:

### 1. Công nghệ cốt lõi (Core Technology)

Ziit là một ứng dụng full-stack hiện đại được xây dựng trên các công nghệ tối ưu cho hiệu năng và xử lý dữ liệu thời gian (time-series):

*   **Framework chính:** **Nuxt 3** (Vue 3 + Nitro). Nuxt được sử dụng cho cả giao diện người dùng (SSR/CSR) và các API backend tích hợp.
*   **Runtime:** **Bun**. Dự án sử dụng Bun làm package manager và runtime chính để đảm bảo tốc độ cài đặt và thực thi nhanh chóng.
*   **Cơ sở dữ liệu:** **PostgreSQL** kết hợp với extension **TimescaleDB**. Đây là lựa chọn cực kỳ quan trọng vì dữ liệu lập trình (heartbeats) là dữ liệu dạng chuỗi thời gian (time-series). TimescaleDB giúp tối ưu hóa việc lưu trữ và truy vấn hàng triệu bản ghi theo thời gian.
*   **ORM:** **Prisma**. Giúp quản lý schema và thao tác dữ liệu một cách an toàn (Type-safe).
*   **Xử lý thời gian thực & Background:** **Nuxt-cron** cho các tác vụ định kỳ và **Prisma Migrations** để quản lý cấu trúc DB.
*   **Bảo mật:** Sử dụng **PASETO** (Platform-Agnostic Security Tokens) thay vì JWT truyền thống để bảo mật token, cùng với **bcrypt** để băm mật khẩu.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Ziit được thiết kế để giải quyết bài toán: "Làm sao để lưu trữ lượng lớn dữ liệu heartbeat mà vẫn đảm bảo dashboard tải nhanh?"

*   **Kiến trúc Hypertable:** Thông qua các file migration (như `20250926115638_convert_to_hypertables`), Ziit chuyển đổi các bảng `Heartbeats` và `Summaries` thành Hypertables của TimescaleDB. Dữ liệu được chia nhỏ thành các "chunks" theo thời gian, giúp việc truy vấn dữ liệu cũ không làm chậm hệ thống.
*   **Cơ chế Tổng hợp (Summarization):** Hệ thống không chỉ lưu heartbeat thô. Nó có một quy trình "summarize" (trong `server/utils/summarize.ts`) để gộp các heartbeat lẻ tẻ thành các bản tóm tắt theo ngày (`Summaries`). Điều này giúp Dashboard hiển thị biểu đồ theo tuần/tháng/năm chỉ trong tích tắc thay vì phải scan hàng triệu dòng heartbeat.
*   **Hybrid Data Fetching:** Khi người dùng xem thống kê, hệ thống kết hợp dữ liệu từ `Summaries` (cho các ngày cũ) và tính toán "live" từ `Heartbeats` (cho dữ liệu hôm nay/hôm qua) thông qua hàm SQL `get_user_stats`.
*   **Kiến trúc Plugin/Extension:** Ziit tách biệt phần thu thập dữ liệu (IDE Extensions cho VS Code, JetBrains) và phần xử lý/hiển thị (Web App), giao tiếp qua **External API** với API Key.

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **Database-Driven Logic (PL/pgSQL):** Thay vì xử lý logic tính toán nặng nề bằng JavaScript, Ziit đẩy các phép tính thống kê phức tạp xuống tầng Database thông qua các function PL/pgSQL (như trong migration `20251029013845_custom_timescale_functions`). Điều này cực kỳ hiệu quả khi xử lý tập dữ liệu lớn.
*   **Composables & State Management:** Sử dụng Nuxt Composables (`useStats.ts`, `useToast.ts`) để quản lý trạng thái ứng dụng một cách modular và dễ tái sử dụng giữa các trang.
*   **Xử lý File lớn (Chunked Upload):** Trong phần import dữ liệu từ WakaTime (file `server/api/import/index.post.ts`), Ziit hỗ trợ upload theo từng đoạn (chunks) để xử lý các file JSON dữ liệu lịch sử lên đến hàng trăm MB mà không làm sập bộ nhớ server.
*   **Materialized Views:** Sử dụng Materialized View (`admin_dashboard_stats_mv`) cho dashboard của admin để tăng tốc độ hiển thị danh sách người dùng và tổng số giờ mà không cần đếm (count) lại toàn bộ database mỗi lần tải trang.

### 4. Luồng hoạt động của hệ thống (System Workflow)

Luồng đi của dữ liệu trong Ziit diễn ra như sau:

1.  **Giai đoạn Thu thập (Ingestion):**
    *   IDE Extension gửi một "Heartbeat" (chứa thông tin: project, language, file, branch, OS...) đến `POST /api/external/heartbeat`.
    *   Hệ thống xác thực bằng `apiKey` và lưu vào bảng `Heartbeats`.
2.  **Giai đoạn Xử lý nền (Background Processing):**
    *   Một Cron job chạy định kỳ (`server/cron/summarize.ts`) lấy các heartbeat mới chưa được xử lý.
    *   Nó tính toán khoảng cách giữa các heartbeat. Nếu khoảng cách nhỏ hơn `keystrokeTimeout`, thời gian đó được tính là đang lập trình.
    *   Kết quả được lưu vào bảng `Summaries` (phân loại theo dự án, ngôn ngữ, editor...).
3.  **Giai đoạn Hiển thị (Visualization):**
    *   Người dùng mở Dashboard (`app/pages/index.vue`).
    *   Trình duyệt gọi `GET /api/stats`.
    *   Backend thực thi hàm SQL `get_user_stats`. Hàm này lấy dữ liệu đã gộp sẵn từ `Summaries` và cộng thêm dữ liệu chưa kịp gộp từ `Heartbeats` của ngày hôm nay.
    *   Dữ liệu trả về được Chart.js vẽ lên biểu đồ.
4.  **Giai đoạn Cộng đồng (Public Stats):**
    *   Nếu người dùng bật `leaderboardEnabled`, dữ liệu của họ sẽ được gộp vào bảng xếp hạng công khai thông qua hàm `get_leaderboard_stats`.

### Tóm lại
Ziit là một ví dụ điển hình về việc sử dụng đúng công cụ cho đúng mục đích: **Nuxt** cho giao diện linh hoạt và **TimescaleDB** cho hiệu năng xử lý dữ liệu theo thời gian. Dự án cho thấy sự am hiểu sâu về kỹ thuật tối ưu hóa database để xây dựng một ứng dụng theo dõi năng suất có khả năng mở rộng tốt.