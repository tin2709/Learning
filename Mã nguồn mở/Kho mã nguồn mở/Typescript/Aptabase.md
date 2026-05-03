Dựa trên mã nguồn và tài liệu kỹ thuật trong kho lưu trữ, dưới đây là phân tích chi tiết về dự án **Aptabase** - một nền tảng phân tích (analytics) mã nguồn mở, ưu tiên quyền riêng tư.

### 1. Công nghệ cốt lõi (Core Tech Stack)

Hệ thống được xây dựng theo mô hình Monolith hiện đại, tối ưu cho hiệu suất cao và khả năng mở rộng:

*   **Backend:** **.NET 8 (ASP.NET Core)**. Đây là lựa chọn mạnh mẽ cho các hệ thống xử lý dữ liệu lớn nhờ hiệu suất runtime cực tốt và thư viện hỗ trợ doanh nghiệp phong phú.
*   **Frontend:** **React 18** kết hợp với **TypeScript**.
    *   **Vite:** Công cụ build và dev server siêu nhanh.
    *   **Tailwind CSS:** Xử lý giao diện.
    *   **Jotai:** Quản lý trạng thái (state management) theo mô hình "Atomic".
    *   **TanStack Query (React Query):** Quản lý việc fetch và cache dữ liệu từ API.
*   **Cơ sở dữ liệu (Polyglot Persistence):**
    *   **PostgreSQL:** Lưu trữ dữ liệu quan hệ (người dùng, thông tin ứng dụng, cấu hình billing, feature flags).
    *   **ClickHouse / Tinybird:** Lưu trữ dữ liệu phân tích (events). ClickHouse là một OLAP database cực nhanh, chuyên dụng cho việc lưu trữ hàng tỷ bản ghi và truy vấn tổng hợp thời gian thực.
*   **Công cụ phụ trợ:**
    *   **Dapper:** Micro-ORM nhẹ cho .NET để thực thi SQL thuần với hiệu suất cao.
    *   **FluentMigrator:** Quản lý phiên bản database (migrations).
    *   **Scriban:** Engine template (sử dụng Liquid) để tạo các câu truy vấn SQL động cho ClickHouse.
    *   **MaxMind GeoIP2:** Xác định vị trí địa lý dựa trên địa chỉ IP.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Aptabase tập trung vào ba yếu tố: **Hiệu suất ghi, Bảo mật quyền riêng tư và Khả năng thay thế.**

*   **Feature-based Organization:** Cấu trúc backend tổ chức theo tính năng (`src/Features/`). Mỗi thư mục (Authentication, Ingestion, Apps, Stats...) chứa đầy đủ từ Controller, Service đến Query logic. Điều này giúp code tách biệt, dễ bảo trì hơn so với kiểu chia theo Layer (Model/View/Controller).
*   **Analytics Backend Agnostic:** Aptabase được thiết kế để có thể chạy với ClickHouse (tự host) hoặc Tinybird (managed service). Điều này được thực hiện thông qua các Interface như `IQueryClient` và `IIngestionClient`, giúp hệ thống linh hoạt trong các môi trường triển khai khác nhau.
*   **Privacy-by-Design:** Thay vì lưu trữ Unique ID cố định của người dùng (dễ bị truy vết), Aptabase sử dụng cơ chế **Daily Hashing**. Mỗi ngày, một Salt mới được tạo ra cho mỗi ứng dụng. ID người dùng được băm (hash) từ IP + UserAgent + Salt ngày hôm đó. Điều này giúp xác định số lượng người dùng trong ngày nhưng không thể nối kết dữ liệu qua nhiều ngày để định danh cá nhân (tuân thủ GDPR/CCPA).

### 3. Kỹ thuật lập trình chính (Key Techniques)

*   **Event Buffering & Batch Writing:** Đây là kỹ thuật quan trọng nhất để xử lý tải cao. Thay vì ghi trực tiếp từng event vào ClickHouse (việc này sẽ làm crash database do overhead của transaction), Aptabase đưa event vào `InMemoryEventBuffer`. Một Background Service (`EventBackgroundWritter`) sẽ thức dậy định kỳ (mỗi 10 giây) để gom toàn bộ event và ghi theo lô (Bulk Insert) vào ClickHouse/Tinybird.
*   **Dynamic SQL với Liquid:** Sử dụng template Scriban để viết SQL cho các báo cáo. Điều này cho phép hệ thống dễ dàng thêm/bớt các điều kiện lọc (filter) phức tạp từ UI mà vẫn giữ được câu lệnh SQL sạch sẽ và tối ưu.
*   **Rate Limiting đa tầng:** Áp dụng chính sách giới hạn request khác nhau cho từng loại API (SignUp: 4 req/h, Stats: 1000 req/h, EventIngestion: 20 req/s) giúp bảo vệ hệ thống khỏi các cuộc tấn công DoS hoặc spam dữ liệu.
*   **Path Aliases & Atomic State (Frontend):** Phía frontend sử dụng path alias (`@features`, `@components`) để tránh "import hell" (`../../../../`). Việc sử dụng Jotai giúp chia nhỏ trạng thái dashboard thành các nguyên tử (atoms), giúp React chỉ re-render những widget thực sự thay đổi dữ liệu.

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng nạp dữ liệu (Ingestion Flow)
1.  **Client/SDK:** Gửi event qua REST API `/api/v0/event`.
2.  **Controller:** Kiểm tra `App-Key`, validate payload.
3.  **GeoIP:** Dựa vào Header (hoặc IP) để lấy mã quốc gia và vùng miền.
4.  **Buffer:** Event được đẩy vào một Queue trong bộ nhớ.
5.  **Background Process:** `EventBackgroundWritter` lấy dữ liệu từ Queue, gọi `DailyUserHasher` để tạo UserID ẩn danh, sau đó thực hiện Bulk Insert vào ClickHouse/Tinybird.

#### B. Luồng truy vấn báo cáo (Analytics Flow)
1.  **Frontend:** Người dùng chọn khoảng thời gian và bộ lọc trên Dashboard.
2.  **State Management:** Jotai atoms cập nhật URL search params.
3.  **API Call:** TanStack Query gửi yêu cầu đến `StatsController`.
4.  **Query Client:** Backend đọc template Liquid tương ứng, điền tham số và gửi truy vấn đến ClickHouse.
5.  **Aggregation:** ClickHouse thực hiện tính toán trên hàng triệu dòng trong vài mili giây và trả về kết quả tổng hợp.
6.  **Visualization:** Frontend nhận dữ liệu JSON và vẽ biểu đồ bằng Chart.js.

#### C. Luồng quản lý quyền riêng tư (Privacy Flow)
1.  Hệ thống chạy một Cron Job hàng ngày (`PurgeDailySaltsCronJob`).
2.  Xóa các Salt cũ hơn 2 ngày trong PostgreSQL.
3.  Dữ liệu UserID cũ trong ClickHouse trở nên không thể giải mã ngược lại IP ban đầu, đảm bảo tính ẩn danh vĩnh viễn.

### Tổng kết
Aptabase là một hệ thống được thiết kế cực kỳ bài bản. Sự kết hợp giữa **.NET 8** cho logic nghiệp vụ và **ClickHouse** cho phân tích dữ liệu là một "cặp bài trùng" chuẩn mực cho các hệ thống Analytics hiện đại. Việc ưu tiên xử lý bất đồng bộ (buffering) giúp hệ thống chịu được tải lớn từ hàng nghìn ứng dụng client cùng lúc.