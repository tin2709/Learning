Dựa trên mã nguồn và cấu trúc thư mục của dự án **CoreControl**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Tech Stack)

Dự án sử dụng mô hình kết hợp giữa hiệu suất của ngôn ngữ hệ thống và sự linh hoạt của web hiện đại:

*   **Frontend & API (Web Container):**
    *   **Framework:** Next.js 15 (App Router).
    *   **Ngôn ngữ:** TypeScript.
    *   **Giao diện:** Tailwind CSS v4 + Shadcn UI (sử dụng hệ màu OKLCH cho thiết kế hiện đại/Dark mode).
    *   **Quản lý dữ liệu:** Prisma ORM kết nối với PostgreSQL.
    *   **Biểu đồ & Sơ đồ:** Chart.js (theo dõi tài nguyên) và React Flow (vẽ sơ đồ mạng hạ tầng).
    *   **Đa ngôn ngữ:** `next-intl` (hỗ trợ Tiếng Anh và Tiếng Đức).

*   **Monitoring Agent (Go Container):**
    *   **Ngôn ngữ:** Go (Golang).
    *   **Mục tiêu:** Chạy các tác vụ nền, giám sát uptime và gửi thông báo với hiệu suất cao và tốn ít RAM nhất.
    *   **Thư viện:** `pgx` (Postgres driver), `gomail` (Email), `godotenv`.

*   **Hạ tầng:**
    *   **Database:** PostgreSQL 17.
    *   **Triển khai:** Docker & Docker Compose (hỗ trợ đa kiến trúc: amd64, arm64, armv7).

---

### 2. Kiến trúc và Tư duy thiết kế

Dự án được thiết kế theo hướng **Decoupled Architecture** (Kiến trúc tách rời):

*   **Sự phân chia nhiệm vụ (Separation of Concerns):**
    *   **Web App:** Chỉ tập trung vào giao diện (UI), quản lý người dùng (Auth) và cung cấp API để cấu hình dữ liệu (CRUD Server/App).
    *   **Agent:** Là "trái tim" của hệ thống giám sát. Nó không có giao diện, chỉ kết nối trực tiếp vào Database để lấy danh sách cần kiểm tra và ghi lại kết quả.

*   **Cơ chế giám sát (Monitoring Logic):**
    *   Dự án sử dụng công cụ **Glances** làm nguồn dữ liệu cho server. Agent gọi vào API của Glances (cài trên các server đích) để lấy thông số CPU, RAM, nhiệt độ...
    *   Đối với ứng dụng, Agent thực hiện HTTP GET request để kiểm tra mã trạng thái (Status Code).

*   **Xử lý dữ liệu lớn (Data Aggregation):**
    *   Trong `app/api/servers/get/route.ts`, hệ thống có tư duy xử lý dữ liệu theo khoảng thời gian (1h, 1d, 7d, 30d). Thay vì gửi hàng triệu bản ghi lịch sử về trình duyệt, API thực hiện tính toán trung bình (average) theo từng khoảng thời gian nhỏ (intervals) trước khi trả về.

*   **Tư duy sơ đồ hóa (Network Flow):**
    *   Logic tại `api/flowchart/route.ts` tự động tính toán tọa độ (X, Y) cho các thiết bị dựa trên mối quan hệ: **Thiết bị vật lý > Máy ảo > Ứng dụng**. Đây là thuật toán sắp xếp phân cấp (Hierarchical Layout) tự chế.

---

### 3. Tóm tắt luồng hoạt động (Workflow)

#### A. Luồng xác thực & Khởi tạo:
1. Khi chạy lần đầu, nếu DB trống, hệ thống cho phép dùng `admin@example.com / admin`.
2. Sau khi đăng nhập, một JWT (JSON Web Token) được tạo và lưu vào Cookie để duy trì phiên làm việc.

#### B. Luồng thu thập dữ liệu (Monitoring Workflow):
1. **Agent** khởi chạy các "Ticker" (luồng lặp định kỳ):
    *   Mỗi 5 giây: Quét các Server bật tính năng `monitoring`. Gọi API Glances -> Lấy CPU/RAM/Temp -> Lưu vào bảng `server_history` và cập nhật bảng `server`.
    *   Mỗi 10 giây: Quét các Application có `publicURL`. Gửi HTTP Request -> Lưu trạng thái vào `uptime_history`.
2. **Xử lý sự cố:** Nếu trạng thái Online/Offline thay đổi so với lần kiểm tra trước, Agent sẽ kích hoạt luồng thông báo.

#### C. Luồng thông báo (Notification Workflow):
1. Agent đọc cấu hình từ bảng `notification` (Telegram, Discord, Gotify, Ntfy, SMTP...).
2. Agent thay thế các biến như `!name`, `!status`, `!url` vào template do người dùng soạn sẵn.
3. Gửi thông báo đến các kênh tương ứng qua Webhook hoặc giao thức SMTP.

#### D. Luồng hiển thị (Dashboard Workflow):
1. Người dùng mở trang chủ: Gọi `api/dashboard/get` để lấy tổng số lượng thiết bị.
2. Người dùng xem chi tiết Server: Gọi `api/servers/get` kèm `timeRange`.
3. Backend xử lý hàng nghìn bản ghi lịch sử trong DB thành khoảng 60-180 điểm dữ liệu đại diện.
4. Frontend dùng Chart.js để vẽ biểu đồ tài nguyên mượt mà.

### Điểm nổi bật:
*   **Tối ưu hóa:** Agent viết bằng Go giúp hệ thống cực kỳ nhẹ, có thể giám sát hàng trăm ứng dụng mà không làm treo máy chủ Dashboard.
*   **Tính tự động:** Sơ đồ mạng tự động cập nhật khi bạn thêm/xóa server hoặc máy ảo, không cần kéo thả thủ công.
*   **Dễ triển khai:** Gói gọn trong Docker, phù hợp cho cả người dùng gia đình (HomeLab) và doanh nghiệp nhỏ.