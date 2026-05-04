Dựa trên tài liệu và cấu trúc mã nguồn của **Jitsu 2.0**, dưới đây là phân tích chi tiết về hệ thống:

### 1. Công nghệ cốt lõi (Core Technology)
Jitsu 2.0 sử dụng mô hình **Polyglot Microservices** (đa ngôn ngữ), tận dụng thế mạnh riêng biệt của Go và Node.js:

*   **Go (Hệ thống Bulker):** Được sử dụng cho lõi thực thi tác vụ nặng. 
    *   **Bulkerapp:** Công cụ ingestion lõi, xử lý streaming và batching dữ liệu vào các Warehouse (BigQuery, Snowflake, ClickHouse, Postgres...).
    *   **Kafka:** Xương sống của hệ thống, dùng để làm hàng đợi thông điệp (message queue), đảm bảo dữ liệu không bị mất (Reliability) và hỗ trợ mở rộng quy mô ngang.
*   **TypeScript/Node.js (Hệ thống Rotor & Console):**
    *   **Rotor:** Chạy trên Node.js, chịu trách nhiệm định tuyến event, biến đổi dữ liệu (transformation) và thực thi các logic hàm.
    *   **Next.js:** Sử dụng cho webapp Console (UI Admin) để quản lý cấu hình và theo dõi luồng dữ liệu.
    *   **Deno:** Được sử dụng làm `functions-server` để chạy các hàm do người dùng định nghĩa (UDF) trong môi trường sandbox an toàn và cô lập.
*   **Infrastructure:** 
    *   **Docker & Kubernetes:** Triển khai qua Docker Compose hoặc Helm Chart. 
    *   **Prisma:** ORM để quản lý cơ sở dữ liệu cấu hình (Postgres).
    *   **Redis:** Dùng làm cache và quản lý trạng thái luồng dữ liệu.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Hệ thống được thiết kế theo tư duy **"Event-Driven & Durable Pipeline"**:

*   **Tách biệt Ingestion và Destination:** Luồng nhận dữ liệu (Ingest) tách biệt hoàn toàn với luồng ghi dữ liệu (Bulker). Kafka đóng vai trò là "vùng đệm" giúp hệ thống chịu tải cao và cho phép retry khi đích đến gặp sự cố.
*   **Kiến trúc hướng Batch & Stream:** Tư duy kiến trúc của Bulker cho phép linh hoạt giữa ghi dữ liệu tức thời (Streaming) cho các DB hỗ trợ tốt, hoặc gom nhóm dữ liệu (Batching) cho các Cloud Warehouse để tối ưu chi phí và hiệu năng.
*   **Sandboxed Execution:** Việc tích hợp Deno và Web Workers cho thấy tư duy bảo mật khi cho phép người dùng viết mã tùy chỉnh (UDF) mà không làm ảnh hưởng đến độ ổn định của hệ thống lõi.
*   **Durable Failover:** Có cơ chế lưu trữ "Dead Letter Queue" và log failover trên S3, kèm theo công cụ reprocessor (Admin service) để xử lý lại dữ liệu lỗi.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)
*   **Go Interfaces & Implementations:** Trong `bulkerlib`, Jitsu sử dụng interface rất mạnh mẽ để trừu tượng hóa các loại database khác nhau (SQL, File Storage, API based), giúp việc thêm một destination mới trở nên dễ dàng.
*   **High-Performance JSON Handling:** Sử dụng `jsoniter` và `jsonorder` trong Go để xử lý JSON với hiệu suất cao nhất mà vẫn giữ được thứ tự các field khi cần thiết.
*   **Monorepo Management:** Sử dụng `pnpm workspaces` cho Node.js và `Go workspaces` cho Go. Điều này cho phép chia sẻ các thư viện dùng chung (như `juava`, `kafkabase`) giữa các service một cách nhất quán.
*   **Multi-stage Docker Builds:** File `all.Dockerfile` thể hiện kỹ thuật tối ưu hóa ảnh Docker, tách biệt builder và runner để giảm kích thước image cuối và tăng tính bảo mật.
*   **UDF (User Defined Functions) Pipeline:** Kỹ thuật đóng gói các hàm TypeScript thành các gói thực thi được trên môi trường Deno/Rotor.

### 4. Luồng hoạt động hệ thống (System Operation Flow)

Hệ thống hoạt động theo một quy trình 4 giai đoạn chính:

1.  **Giai đoạn Ingest (Thu thập):**
    *   Dữ liệu từ SDK (JS, React, v.v.) hoặc HTTP API gửi đến service `Ingest` (Go).
    *   `Ingest` kiểm tra API Key và đẩy ngay vào Kafka topic thô (raw events).
2.  **Giai đoạn Process & Route (Xử lý & Định tuyến):**
    *   `Rotor` (Node.js) tiêu thụ dữ liệu từ Kafka.
    *   Thực thi các hàm biến đổi (Transformations) và các logic làm sạch dữ liệu.
    *   Sau khi xử lý, `Rotor` đẩy dữ liệu đã định dạng vào các Kafka topic riêng biệt cho từng đích đến (Destinations).
3.  **Giai đoạn Bulker (Ghi dữ liệu):**
    *   `Bulker` tiêu thụ dữ liệu từ các topic destination.
    *   Dựa trên cấu hình, nó sẽ quyết định ghi ngay (Stream) hoặc gom lại (Batch) vào bộ nhớ/file tạm.
    *   Thực hiện "Flattening" JSON (phẳng hóa dữ liệu) và tự động quản lý schema (tự tạo bảng/thêm cột vào Warehouse).
4.  **Giai đoạn Management & Recovery (Quản lý & Phục hồi):**
    *   `Console` cung cấp giao diện để cấu hình và xem logs.
    *   Nếu ghi dữ liệu thất bại, dữ liệu được đẩy vào `Dead Letter Queue` hoặc lưu trên S3.
    *   Dịch vụ `Admin` (failover reprocessor) có thể được kích hoạt để đọc log lỗi và đẩy lại vào pipeline.

### Tổng kết
Jitsu 2.0 là một hệ thống **Data Engineering** hiện đại, ưu tiên tính bền vững của dữ liệu (Data Durability) và khả năng mở rộng. Sự kết hợp giữa Go (hiệu suất ghi) và Node.js (linh hoạt trong biến đổi dữ liệu) tạo nên một giải pháp thay thế mạnh mẽ cho các dịch vụ trả phí như Segment.