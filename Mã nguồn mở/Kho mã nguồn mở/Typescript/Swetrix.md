Dựa trên tài liệu bạn cung cấp, Swetrix là một hệ thống phân tích web mã nguồn mở, tập trung vào quyền riêng tư và không sử dụng cookie. Dưới đây là phân tích chi tiết về dự án này bằng tiếng Việt:

### 1. Công Nghệ Cốt Lõi (Core Technology Stack)

Hệ thống được xây dựng trên một ngăn xếp công nghệ hiện đại, tối ưu cho việc xử lý dữ liệu lớn (Big Data) và thời gian thực:

*   **Backend:** Sử dụng **Nest.js** (Node.js Framework) với ngôn ngữ TypeScript. Đây là lựa chọn giúp hệ thống có cấu trúc module rõ ràng, dễ bảo trì và mở rộng.
*   **Cơ sở dữ liệu (Hybrid Database Strategy):**
    *   **MySQL (TypeORM):** Lưu trữ các dữ liệu quan hệ, cấu hình thực thể như: Người dùng (Users), Tổ chức (Organisations), Dự án (Projects), Cài đặt báo động (Alerts).
    *   **ClickHouse:** Đây là "trái tim" của hệ thống phân tích. ClickHouse là một Column-oriented DBMS cực nhanh, chuyên dùng để lưu trữ và truy vấn hàng tỷ dòng dữ liệu sự kiện (events), lượt xem trang (pageviews) và dữ liệu hiệu năng (performance).
    *   **Redis:** Sử dụng để lưu trữ cache và quản lý trạng thái thời gian thực (như khách truy cập đang trực tuyến).
*   **Frontend:** Sử dụng **React Router** (với Vite), **Tailwind CSS** để thiết kế giao diện và **Billboard.js** để vẽ biểu đồ dữ liệu.
*   **AI Integration:** Tích hợp **OpenAI/OpenRouter** (Claude 3.5 Sonnet/Haiku) để cung cấp tính năng "Ask AI", cho phép người dùng chat với dữ liệu của họ.
*   **Infrastructure:** Chạy trên **Docker** và Docker Compose, hỗ trợ triển khai dễ dàng qua môi trường self-hosted.

### 2. Tư Duy Kiến Trúc (Architectural Thinking)

Kiến trúc của Swetrix được thiết kế theo hướng **Separation of Concerns (Phân tách các mối quan tâm)** và **Scalability (Khả năng mở rộng)**:

*   **Mô hình Monorepo:** Dự án quản lý cả `admin` (CLI), `backend`, `web` (frontend) và `docs` trong một kho lưu trữ duy nhất, giúp đồng bộ hóa logic nghiệp vụ.
*   **Phân tách OLTP và OLAP:**
    *   **OLTP (MySQL):** Xử lý các giao dịch thông thường (đăng ký, đăng nhập, tạo dự án).
    *   **OLAP (ClickHouse):** Xử lý phân tích dữ liệu lớn. Việc tách biệt này giúp các truy vấn báo cáo nặng không làm chậm hệ thống quản lý tài khoản.
*   **Kiến trúc Đa tầng (Multi-tenant):** Hỗ trợ "Organisations", cho phép nhiều người dùng cùng quản lý các dự án chung với các cấp độ quyền hạn (Owner, Admin, Viewer).
*   **Thiết kế hướng Module:** Backend chia thành các module độc lập như `analytics`, `auth`, `project`, `ai`, `billing`... giúp việc tắt/mở tính năng (ví dụ: bản Community vs bản Cloud) trở nên linh hoạt.

### 3. Các Kỹ Thuật Chính (Key Techniques)

*   **Privacy-first (Cookieless Tracking):** Hệ thống không sử dụng Cookie để theo dõi người dùng. Thay vào đó, nó sử dụng kỹ thuật **Salting & Hashing** (thông qua `SaltService`) để tạo ra các ID ẩn danh dựa trên IP và User Agent, đảm bảo tuân thủ GDPR mà không xâm phạm quyền riêng tư.
*   **Real-time Heartbeat:** Sử dụng **WebSocket (Socket.io)** trong `heartbeat.gateway.ts` để theo dõi thời gian thực người dùng ở lại trên trang web.
*   **Bot Detection:** Tích hợp các decorator và guard (`bot-detection.guard.ts`) để lọc bỏ lưu lượng truy cập từ robot, đảm bảo số liệu thống kê chính xác.
*   **Advanced Analytics:**
    *   **Funnels:** Phân tích phễu chuyển đổi.
    *   **User Flow:** Theo dõi luồng hành vi người dùng.
    *   **Error Tracking:** Ghi lại các lỗi JavaScript từ phía client.
*   **Hệ thống Plugin/Integration:** Hỗ trợ thông báo qua Slack, Discord, Telegram khi có các cảnh báo (Alerts) về lưu lượng hoặc lỗi.

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

Luồng dữ liệu đi từ website của khách hàng đến bảng điều khiển (Dashboard) diễn ra như sau:

1.  **Thu thập (Ingestion):** Một đoạn mã JavaScript nhỏ trên website người dùng gửi dữ liệu sự kiện (pageview, custom event) về API endpoint `/log` của Swetrix Backend.
2.  **Xử lý & Làm giàu (Processing & Enrichment):**
    *   Backend kiểm tra API Key (Project ID).
    *   Xác định vị trí địa lý (GeoIP) từ IP của khách (thông qua DB-IP).
    *   Phân tích User Agent để biết thiết bị, trình duyệt.
    *   Kiểm tra xem có phải là Bot hay không.
3.  **Lưu trữ (Storage):**
    *   Dữ liệu thô được ghi vào **ClickHouse** để phục vụ phân tích lâu dài.
    *   Trạng thái trực tuyến được cập nhật vào **Redis**.
4.  **Truy vấn (Querying):** Khi người dùng mở Dashboard, Frontend gửi yêu cầu đến các API thuộc module `analytics`. Backend sẽ thực hiện các câu lệnh SQL phức tạp (Aggregations) trên ClickHouse để tính toán các chỉ số (Unique visitors, Bounce rate, Top pages...).
5.  **Hiển thị (Visualization):** Dữ liệu trả về dưới dạng JSON, sau đó Frontend sử dụng Billboard.js để render thành các biểu đồ trực quan.
6.  **AI Insight (Tùy chọn):** Nếu người dùng sử dụng "Ask AI", hệ thống sẽ lấy dữ liệu từ ClickHouse đưa vào ngữ cảnh (Context) của LLM để trả lời các câu hỏi về xu hướng dữ liệu.

Tóm lại, Swetrix là một giải pháp hoàn chỉnh từ thu thập đến phân tích dữ liệu, được tối ưu hóa sâu về mặt hiệu năng xử lý dữ liệu lớn nhờ vào việc kết hợp giữa Nest.js và ClickHouse.