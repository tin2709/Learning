Dựa trên mã nguồn và cấu trúc thư mục của **Umami**, đây là phân tích chi tiết về một trong những nền tảng phân tích web mã nguồn mở thành công nhất hiện nay:

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Umami là một ứng dụng Full-stack hiện đại, được xây dựng trên những công nghệ có hiệu suất cao và khả năng mở rộng tốt:

*   **Framework chính:** **Next.js (React)**. Sử dụng **App Router** (thư mục `src/app`), tận dụng tối đa Server Components và API Routes.
*   **Ngôn ngữ:** **TypeScript (97%)** - đảm bảo tính an toàn về kiểu dữ liệu trên toàn bộ hệ thống.
*   **Quản lý Cơ sở dữ liệu:**
    *   **Prisma ORM:** Dùng để quản lý các dữ liệu quan hệ (Users, Websites, Teams, Settings).
    *   **PostgreSQL:** Database mặc định cho mọi loại dữ liệu.
    *   **ClickHouse (Tùy chọn):** Một cơ sở dữ liệu dạng cột (column-oriented) cực nhanh, được Umami sử dụng để lưu trữ và truy vấn hàng tỷ sự kiện (events) phân tích mà Postgres khó gánh vác nổi.
*   **Xử lý dữ liệu & Streaming:**
    *   **Kafka (Tùy chọn):** Dùng để làm hàng đợi (buffer) cho dữ liệu gửi về trước khi đưa vào ClickHouse, giúp hệ thống không bị nghẽn khi có hàng triệu request đồng thời.
    *   **Redis:** Dùng để caching.
*   **Theo dõi & Ghi hình:**
    *   **rrweb:** Công nghệ cốt lõi cho tính năng **Session Replay** (ghi lại thao tác của người dùng trên web).
    *   **Chart.js:** Thư viện chính để render các biểu đồ trực quan.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Umami tập trung vào việc **"Vượt qua bộ chặn quảng cáo"** và **"Khả năng mở rộng linh hoạt"**:

*   **Kiến trúc Hybrid Storage:** Umami rất thông minh khi tách biệt dữ liệu "trạng thái" (State) và dữ liệu "phân tích" (Analytics). Nếu bạn chạy nhỏ, Postgres là đủ. Nếu bạn chạy lớn, hệ thống có thể cấu hình để đẩy Analytic Events sang ClickHouse trong khi vẫn giữ cấu hình ở Postgres.
*   **Kiến trúc "Bypass Ad-blocker":** Umami cho phép người dùng đổi tên file tracker (`script.js` thành `anything.js`) và đổi endpoint thu thập dữ liệu thông qua cấu hình biến môi trường và Next.js Rewrites. Điều này khiến các bộ chặn quảng cáo dựa trên tên file/url khó nhận diện hơn.
*   **Privacy-First (Không Cookie):** Umami không sử dụng cookie để theo dõi người dùng. Thay vào đó, nó tạo ra một `session_id` dựa trên Hash của (IP + User Agent + Website ID + Salt hàng tháng). Điều này giúp tuân thủ GDPR mà không cần hiện bảng thông báo cookie.
*   **Multi-tenant & Teams:** Hỗ trợ phân quyền mạnh mẽ giữa cá nhân và đội nhóm (Teams), cho phép chia sẻ Dashboard thông qua các đường dẫn Share URL (sử dụng UUID).

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Tracker Script Optimization:** File `src/tracker/index.js` được viết cực kỳ tối giản và sau đó được đóng gói bằng **Rollup** với Terser để nén dung lượng xuống mức nhỏ nhất (chỉ vài KB), đảm bảo không ảnh hưởng đến tốc độ tải trang của khách hàng.
*   **SQL Raw & Aggregation:** Đối với các truy vấn phân tích phức tạp, Umami không dùng Prisma mà viết **SQL thuần** (thư mục `src/queries/sql`). Các kỹ thuật như `argMin`, `argMax` (trong ClickHouse) và `groupArray` được tận dụng để tính toán Entry/Exit pages và phễu chuyển đổi (Funnels).
*   **Middleware Proxying:** File `docker/proxy.ts` đóng vai trò như một lớp bảo vệ và điều hướng request, xử lý CORS và headers cho các yêu cầu thu thập dữ liệu từ các domain khác nhau gửi về.
*   **Dynamic Component Registry:** Thư mục `boards/` cho thấy cách Umami quản lý các Widget trên Dashboard. Các thành phần này được đăng ký vào một Registry để người dùng có thể tùy biến bố cục Dashboard theo ý muốn.
*   **I18n Workflow:** Sử dụng `next-intl` kết hợp với các file JSON trong `public/intl/` giúp hỗ trợ đa ngôn ngữ một cách mượt mà trên cả client và server.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng Thu thập (Collect Flow):
1.  Trình duyệt khách hàng tải đoạn mã tracker (ví dụ: `script.js`).
2.  Tracker thu thập các thông số: Độ phân giải màn hình, ngôn ngữ, URL, Referrer...
3.  Gửi POST request đến `/api/send`.
4.  **API Route xử lý:**
    *   Kiểm tra `isbot` để loại bỏ các lượt truy cập ảo.
    *   Phân giải IP để lấy thông tin quốc gia/thành phố (sử dụng MaxMind).
    *   Tạo/Cập nhật Session trong Postgres/ClickHouse.
    *   Lưu Event (Pageview hoặc Custom Event).

#### B. Luồng Hiển thị (Dashboard Flow):
1.  User đăng nhập vào Dashboard (Next.js SSR).
2.  Frontend gọi các API chuyên biệt như `/api/websites/[id]/stats` hoặc `/api/reports/funnel`.
3.  Backend kiểm tra quyền (Permissions) của User đối với Website đó.
4.  Thực hiện truy vấn Aggregation (Tổng hợp) từ database:
    *   Nếu dùng Clickhouse: Truy vấn từ các bảng `website_event` với engine `MergeTree`.
    *   Nếu dùng Postgres: Truy vấn từ các bảng `website_event` đã được đánh Index tối ưu.
5.  Dữ liệu trả về JSON -> Dashboard dùng Chart.js vẽ biểu đồ.

#### C. Luồng Ghi hình (Session Replay Flow):
1.  Nếu tính năng Replay được bật, tracker sẽ dùng `rrweb` để ghi lại các thay đổi của DOM (Mutation).
2.  Dữ liệu được nén (ZSTD/Gzip) và gửi về backend theo từng phân đoạn (chunks).
3.  Khi xem lại, `rrweb-player` sẽ tái tạo lại các mutation này trên một sandbox iframe để admin có thể xem video mô phỏng hành vi người dùng.

### Tổng kết
Umami là một ví dụ mẫu mực về việc sử dụng **Next.js** để xây dựng ứng dụng SaaS. Nó cân bằng hoàn hảo giữa tính đơn giản cho người dùng cuối và sự phức tạp về hạ tầng xử lý dữ liệu lớn (ClickHouse, Kafka) ở phía sau. Đây là giải pháp lý tưởng cho những ai muốn tự chủ dữ liệu mà vẫn đòi hỏi hiệu năng ngang tầm Enterprise.