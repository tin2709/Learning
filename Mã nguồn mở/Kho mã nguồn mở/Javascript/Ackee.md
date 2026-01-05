Dựa trên mã nguồn và tài liệu của kho lưu trữ **Ackee**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của dự án này:

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Ackee được xây dựng trên một stack hiện đại, tập trung vào hiệu suất và tính gọn nhẹ:

*   **Backend Runtime:** Node.js (Yêu cầu v22+ trong bản cập nhật mới nhất).
*   **Cơ sở dữ liệu:** MongoDB (Sử dụng Mongoose ODM). Đây là lựa chọn tối ưu cho việc lưu trữ dữ liệu log và thực hiện các truy vấn gộp (aggregation) phức tạp.
*   **API Layer:** GraphQL (Apollo Server). Toàn bộ giao tiếp giữa UI và Server đều qua GraphQL, giúp tối ưu hóa dữ liệu truyền tải.
*   **Frontend:** React kết hợp với Apollo Client để quản lý trạng thái và dữ liệu từ API.
*   **Styling:** SCSS (Sass) được biên dịch tùy chỉnh.
*   **Deployment & Infrastructure:** 
    *   Hỗ trợ mạnh mẽ **Docker** và **Docker Compose**.
    *   Khả năng chạy **Serverless** trên Vercel hoặc Netlify (thông qua các adapter trong thư mục `api/` và `functions/`).
*   **Build Tool:** Sử dụng các công cụ tùy chỉnh như `build.js` kết hợp với `rosid-handler-js-next` và `rosid-handler-sass` để đóng gói tài nguyên.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Architecture & Design Thinking)

Ackee được thiết kế theo phong cách **Modular Monolith** nhưng sẵn sàng cho **Serverless**:

*   **GraphQL-Centric:** Ackee không sử dụng REST API truyền thống cho dashboard. Tư duy ở đây là "API-First" – mọi chức năng trên giao diện người dùng đều có một mutation hoặc query tương ứng. Điều này cho phép cộng đồng dễ dàng xây dựng các công cụ bên thứ ba (như plugin WordPress, module Nuxt).
*   **Separation of Concerns (Tách biệt trách nhiệm):**
    *   `src/models/`: Định nghĩa cấu trúc dữ liệu (Schema).
    *   `src/database/`: Lớp trung gian thực hiện các thao tác CRUD và truy vấn thực tế vào MongoDB.
    *   `src/resolvers/`: Xử lý logic nghiệp vụ cho GraphQL.
    *   `src/aggregations/`: Chứa các "pipeline" MongoDB phức tạp để tính toán số liệu thống kê (views, durations, active visitors).
*   **Tư duy Privacy-First (Quyền riêng tư là trên hết):** Kiến trúc hệ thống không lưu trữ bất kỳ thông tin định danh cá nhân (PII) nào. Nó sử dụng cơ chế hash (IP + User Agent + Salt hàng ngày) để tạo `clientId` tạm thời.

---

### 3. Các kỹ thuật chính nổi bật (Key Techniques)

*   **Cơ chế Anonymization (Vô danh hóa):**
    *   Hệ thống sử dụng một "Salt" thay đổi hàng ngày để băm dữ liệu. Salt này không được lưu trữ vĩnh viễn, khiến việc truy vết lịch sử duyệt web dài hạn của một người dùng là không thể, ngay cả khi database bị rò rỉ.
*   **MongoDB Aggregation Pipelines:** 
    *   Thay vì lưu trữ các bảng tổng hợp sẵn, Ackee thực hiện tính toán thống kê "on-the-fly" bằng các pipeline cực kỳ tối ưu (nằm trong `src/aggregations/`). Điều này đảm bảo dữ liệu luôn là thời gian thực (Real-time).
*   **Adapter Serverless:** 
    *   Dự án có các file như `vercel.json`, `netlify.toml` và các handler đặc biệt (`src/serverless.js`) để ánh xạ yêu cầu từ môi trường Lambda/Cloud Functions vào Apollo Server.
*   **Cơ chế Tracking không Cookie:** 
    *   Kỹ thuật xác định người dùng duy nhất dựa trên fingerprinting tạm thời và tiêu đề HTTP, loại bỏ nhu cầu hiển thị banner chấp nhận cookie phiền phức.
*   **Tùy biến Tracker:**
    *   Cho phép người dùng thay đổi tên file `tracker.js` (thành một tên ngẫu nhiên qua biến môi trường `ACKEE_TRACKER`) để tránh bị các trình chặn quảng cáo (Ad-blockers) phát hiện dựa trên tên file tĩnh.

---

### 4. Tóm tắt luồng hoạt động của Project (Project Workflow)

Luồng hoạt động của Ackee chia làm hai giai đoạn chính: **Ghi nhận dữ liệu** và **Hiển thị báo cáo**.

#### Bước 1: Thu thập (Tracking)
1.  Trang web của người dùng nhúng file `tracker.js`.
2.  Khi có khách truy cập, `tracker.js` thu thập các thông số cơ bản (URL, Referrer, Language, Screen size...).
3.  Gửi một yêu cầu GraphQL Mutation `createRecord` đến Ackee server.

#### Bước 2: Xử lý & Lưu trữ (Ingestion)
1.  Server nhận yêu cầu, trích xuất IP và User-Agent.
2.  **Định danh:** Tính toán `clientId` bằng cách băm (Hash) các thông tin trên với Salt của ngày hiện tại.
3.  **Lưu trữ:** Tạo một bản ghi mới trong bộ sưu tập `Record` của MongoDB.
4.  **Làm sạch:** Ngay sau khi ghi nhận, hệ thống thực hiện `anonymize` các bản ghi cũ của cùng `clientId` đó (xóa các thông tin chi tiết như thiết bị, trình duyệt) để chỉ giữ lại số liệu thống kê tổng quát, đảm bảo tính riêng tư.

#### Bước 3: Phân tích & Hiển thị (Reporting)
1.  Người quản trị đăng nhập vào Dashboard (React App).
2.  Dashboard gửi các Query GraphQL (như `getDomainsFacts`, `getStatistics`).
3.  Server gọi các hàm trong `src/database/` -> thực thi các MongoDB Aggregation (`src/aggregations/`).
4.  Dữ liệu được gộp theo ngày/tháng/năm và trả về dưới dạng JSON cho Dashboard hiển thị biểu đồ.

#### Bước 4: Theo dõi thời gian thực (Heartbeat)
1.  `tracker.js` gửi các yêu cầu `updateRecord` định kỳ để tính toán thời gian người dùng ở lại trang web (Duration).
2.  Tính năng "Active Visitors" sử dụng một truy vấn đếm các bản ghi có cập nhật (updated) trong vòng vài giây gần nhất để hiển thị số người đang trực tuyến.

---

### Kết luận
Ackee là một minh chứng cho việc kết hợp giữa **GraphQL** và **MongoDB Aggregation** để tạo ra một hệ thống phân tích dữ liệu mạnh mẽ nhưng vẫn đảm bảo sự đơn giản và tôn trọng quyền riêng tư người dùng. Nó được tối ưu hóa cho cả việc tự cài đặt (self-hosted) lẫn triển khai trên các nền tảng đám mây hiện đại.