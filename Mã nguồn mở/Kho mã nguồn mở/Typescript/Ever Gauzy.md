Dựa trên cấu trúc thư mục và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về dự án **Ever Gauzy Platform**:

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án là một hệ sinh thái mã nguồn mở đồ sộ sử dụng mô hình **Monorepo** để quản lý nhiều ứng dụng và thư viện trong một kho lưu trữ duy nhất.

*   **Ngôn ngữ chính:** **TypeScript** (chiếm gần 90% mã nguồn), mang lại tính an toàn về kiểu dữ liệu cho cả frontend và backend.
*   **Backend:** Xây dựng trên **NestJS** (Node.js framework). Sử dụng đồng thời nhiều ORM như **TypeORM**, **MikroORM** và **Knex** để đảm bảo khả năng tương thích với nhiều loại cơ sở dữ liệu (PostgreSQL, MySQL, SQLite).
*   **Frontend:** Chủ yếu dựa trên **Angular** kết hợp với bộ UI kit **Nebular** và template **ngx-admin**. Quản lý trạng thái bằng **Akita**.
*   **Desktop & App chuyên dụng:** Sử dụng **Electron** để xây dựng ứng dụng máy tính (Gauzy Desktop và Desktop Timer) tích hợp các tính năng cấp hệ thống như chụp màn hình, theo dõi hoạt động bàn phím/chuột.
*   **Cơ sở hạ tầng & DevOps:**
    *   **Nx & Lerna:** Công cụ quản lý Monorepo, tối ưu hóa quá trình build và test.
    *   **Docker & Docker Compose:** Đóng gói ứng dụng thành các container.
    *   **Cơ sở dữ liệu bổ trợ:** **Redis** (caching/queue), **OpenSearch** (tìm kiếm), **Cube.js** (phân tích dữ liệu/BI), **MinIO** (lưu trữ file tương đương S3).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án được thiết kế theo hướng **Modular Monolith (Nguyên khối theo mô-đun)** và sẵn sàng cho **Microservices**:

*   **Nx Workspace:** Chia dự án thành các `apps` (api, webapp, desktop, agent) và `packages` (auth, common, core, contracts). Điều này cho phép tái sử dụng code giữa server, web và ứng dụng desktop.
*   **Headless-First:** Backend cung cấp bộ API mạnh mẽ (Headless APIs), cho phép nhiều loại client khác nhau (Web, Mobile, Desktop, MCP Server) kết nối vào cùng một hệ thống logic.
*   **Plugin-Based Architecture:** Các tính năng mở rộng (như tích hợp Github, Hubstaff, AI) được tách thành các plugin riêng biệt trong thư mục `packages/plugins`, giúp việc bảo trì và mở rộng linh hoạt mà không ảnh hưởng đến lõi (Core).
*   **Đa nền tảng (Platform Agnostic):** Kiến trúc cho phép chạy trên Cloud (SaaS), tự cài đặt (Self-hosted), hoặc chạy dưới dạng ứng dụng Desktop "all-in-one" (tích hợp cả API và DB bên trong Electron).

### 3. Các kỹ thuật chính (Key Techniques)

*   **Theo dõi hoạt động thời gian thực (Activity Tracking):** Đây là kỹ thuật đặc thù của Gauzy. Ứng dụng Desktop sử dụng Electron để chụp ảnh màn hình định kỳ, tính toán cường độ gõ phím/chuột và đồng bộ hóa dữ liệu này về server thông qua hàng đợi (Queue).
*   **Abstraction Layer cho Database:** Sử dụng mẫu `Repository` và `Factory` để có thể hoán đổi giữa các ORM (TypeORM vs MikroORM) và các loại DB khác nhau chỉ bằng cách thay đổi biến môi trường.
*   **Multi-tenancy (Đa người thuê):** Kiến trúc hỗ trợ tách biệt dữ liệu giữa các `Tenants` (Khách hàng) và `Organizations` (Tổ chức), đảm bảo bảo mật và quyền truy cập trong mô hình SaaS.
*   **MCP (Model Context Protocol):** Một kỹ thuật mới được đưa vào để tích hợp với các AI Agent (như Claude, ChatGPT), cho phép AI "hiểu" và tương tác trực tiếp với dữ liệu ERP/CRM của doanh nghiệp.
*   **Hệ thống Seeding tự động:** Kỹ thuật sinh dữ liệu mẫu (faker) cực lớn để phục vụ demo và kiểm thử hiệu năng.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Khởi tạo:** Khi chạy lệnh `yarn bootstrap`, Nx sẽ thiết lập các phụ thuộc giữa các gói. Server API (`apps/api`) khi khởi động sẽ kiểm tra DB, nếu trống sẽ thực hiện "seed" dữ liệu cơ bản.
2.  **Giao diện người dùng:** Người dùng truy cập qua Web (Angular) hoặc Desktop (Electron). Yêu cầu được gửi đến NestJS API.
3.  **Xử lý Logic:** NestJS sử dụng `Guards` để kiểm tra quyền, `Interceptors` để biến đổi dữ liệu và gửi yêu cầu đến các `Services`. Nếu là tác vụ nặng (như báo cáo, đồng bộ hóa), nó sẽ được đẩy vào BullMQ/Redis.
4.  **Luồng Tracking (Dành cho nhân viên):**
    *   Gauzy Agent trên máy tính bắt đầu theo dõi.
    *   Dữ liệu thô (ảnh, logs) được lưu tạm cục bộ nếu mất mạng (Offline mode).
    *   Khi có mạng, Sync Manager sẽ đẩy dữ liệu về API.
    *   Server xử lý, tính toán năng suất và lưu vào DB trung tâm.
5.  **Phân tích & Báo cáo:** Cube.js truy vấn DB chính để cung cấp dữ liệu cho các biểu đồ (Charts) trên Dashboard theo thời gian thực.

**Đánh giá:** Ever Gauzy là một dự án có độ phức tạp kỹ thuật rất cao, thể hiện sự kết hợp nhuần nhuyễn giữa hệ sinh thái JavaScript/TypeScript hiện đại cho nhu cầu quản trị doanh nghiệp toàn diện.