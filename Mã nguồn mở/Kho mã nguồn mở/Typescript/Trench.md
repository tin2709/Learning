Dựa trên mã nguồn của **Trench**, đây là phân tích chi tiết về một hệ thống hạ tầng dữ liệu (Data Infrastructure) hiện đại, được thiết kế để xử lý quy mô lớn (High Throughput).

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Trench không chọn các công nghệ database truyền thống (như MySQL/Postgres) làm trung tâm, mà tập trung vào các công nghệ xử lý dữ liệu lớn:

*   **Backend Framework:** **NestJS** (Node.js) chạy trên nền **Fastify**. Việc sử dụng Fastify thay vì Express giúp tối ưu hóa tốc độ phản hồi API, cực kỳ quan trọng cho các endpoint nhận event.
*   **Hệ thống hàng đợi (Message Broker):** **Apache Kafka**. Đây là "trái tim" của hệ thống, giúp tách rời (decouple) việc nhận dữ liệu từ API và việc ghi dữ liệu vào database, đảm bảo hệ thống không bị treo khi lượng event tăng đột biến.
*   **Cơ sở dữ liệu OLAP:** **ClickHouse**. Được mệnh danh là DB nhanh nhất thế giới cho phân tích. Trench sử dụng ClickHouse để lưu trữ và truy vấn hàng triệu event trong thời gian thực.
*   **Communication:** Giao thức REST API tuân thủ tiêu chuẩn của **Segment** (Track, Identify, Group, Page), giúp các nhà phát triển dễ dàng chuyển đổi từ các nền tảng cũ sang.
*   **Client SDK:** Viết bằng TypeScript, hỗ trợ cả Browser và Node.js, tích hợp sẵn cơ chế **Batching** (gom nhóm event) để giảm số lượng HTTP request.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Trench được xây dựng theo mô hình **Lambda/Kappa Architecture** rút gọn, tập trung vào tốc độ ingest dữ liệu:

*   **Pipeline Ingestion (Luồng nạp dữ liệu):** 
    `SDK -> API (NestJS) -> Kafka -> ClickHouse Materialized View -> ClickHouse MergeTree`.
    *   API chỉ làm nhiệm vụ đẩy data vào Kafka rồi trả về 201 ngay lập tức (Async).
    *   Dữ liệu từ Kafka được ClickHouse tự động "kéo" về thông qua **Materialized View**. Đây là tư duy kiến trúc cực kỳ thông minh, tận dụng tối đa tính năng của ClickHouse thay vì viết code consumer bằng Node.js (vốn chậm hơn).
*   **Multi-tenancy (Đa người dùng):** Trench hỗ trợ **Workspaces**. Mỗi Workspace có thể có database riêng trong ClickHouse, giúp cô lập dữ liệu tuyệt đối giữa các khách hàng hoặc dự án khác nhau.
*   **Security (Bảo mật 2 lớp):** Sử dụng cặp `Public API Key` (chỉ dùng để ghi dữ liệu - write-only) và `Private API Key` (dùng để truy vấn - read-only). Điều này đảm bảo an toàn ngay cả khi mã frontend bị lộ.
*   **Stateless API:** Các node API có thể mở rộng (scale-out) dễ dàng bằng Docker vì không lưu giữ trạng thái, mọi trạng thái đều nằm ở Kafka/ClickHouse.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Node.js Clustering:** Trong file `appCluster.service.ts`, Trench sử dụng module `cluster` của Node.js để tận dụng tối đa đa nhân CPU trên một server đơn lẻ, cho phép xử lý hàng nghìn request mỗi giây.
*   **SQL Substitution & Migrations:** Hệ thống tự viết một bộ chạy migration (`runMigrations` trong `click-house.service.ts`). Nó tự động đọc file `.sql`, thay thế các biến môi trường (như Kafka broker list) vào câu lệnh SQL trước khi thực thi.
*   **ReadOnly SQL Guard:** Để cho phép người dùng chạy query SQL trực tiếp qua API mà vẫn an toàn, Trench sử dụng Regex (`isReadOnlyQuery` trong `queries.util.ts`) để chặn đứng các câu lệnh nguy hiểm như `DROP`, `DELETE`, `INSERT`, `UPDATE`.
*   **Exponential Backoff & Retry:** Trong `webhooks.service.ts`, khi gửi webhook thất bại, hệ thống sử dụng thuật toán lùi thời gian lũy thừa (Exponential Backoff) để thử lại, tránh làm quá tải hệ thống đích.
*   **Flattening Data:** Kỹ thuật chuyển đổi cấu trúc JSON lồng nhau thành cấu trúc phẳng (`flatten` trong `utils.ts`) để phù hợp với một số hệ thống downstream không hỗ trợ JSON phức tạp.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng Thu thập (Ingestion Flow):
1.  **Client:** SDK gom các event (Batching) và gửi POST tới `/events`.
2.  **API Guard:** Kiểm tra Public API Key, xác định Workspace.
3.  **Kafka Producer:** API đẩy event vào topic Kafka tương ứng với Workspace đó.
4.  **ClickHouse Consumer:** Một bảng engine `Kafka` trong ClickHouse liên tục lắng nghe topic.
5.  **Materialized View:** Tự động parse chuỗi JSON từ Kafka và map vào các cột trong bảng `events` (sử dụng engine `MergeTree` để tối ưu nén dữ liệu).

#### B. Luồng Truy vấn (Query Flow):
1.  **User:** Gửi câu lệnh SQL qua POST tới `/queries` kèm Private Key.
2.  **Validation:** API kiểm tra tính "Read-only" của câu lệnh.
3.  **Transformation:** API tự động chuyển đổi các trường camelCase (userId) sang snake_case (user_id) của ClickHouse để người dùng không cần quan tâm đến cách DB đặt tên cột.
4.  **Execution:** Truy vấn trực tiếp vào ClickHouse và trả về kết quả JSON.

#### C. Luồng Webhook (Fan-out Flow):
1.  **Consumer:** Trench chạy các worker lắng nghe Kafka đồng thời với ClickHouse.
2.  **Filtering:** Khi có event mới, worker kiểm tra xem có Webhook nào đăng ký cho loại event này không.
3.  **Delivery:** Nếu khớp, worker sẽ gửi HTTP POST chứa dữ liệu event tới URL đích.

### Tổng kết
Trench là một giải pháp thay thế mã nguồn mở cho Segment/PostHog nhưng tập trung vào **hiệu năng thuần túy**. Nó tận dụng sức mạnh xử lý song song của Kafka và khả năng lưu trữ cột (columnar storage) của ClickHouse để giải quyết bài toán Big Data Analytics một cách tinh gọn.