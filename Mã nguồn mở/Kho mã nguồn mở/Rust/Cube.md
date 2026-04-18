Dựa trên cấu trúc mã nguồn và tài liệu đi kèm của **Cube (trước đây là Cube.js)**, dưới đây là phân tích chi tiết về dự án này:

### 1. Công nghệ cốt lõi (Core Technology)

Cube là một **Semantic Layer (Tầng ngữ nghĩa)** mã nguồn mở, được thiết kế để kết nối giữa các kho dữ liệu (Data Warehouse) và ứng dụng đầu cuối.

*   **Đa ngôn ngữ (Hybrid Architecture):** 
    *   **Rust (52.5%):** Được dùng để xây dựng các thành phần hiệu năng cao như `CubeStore` (OLAP storage engine) và `CubeSQL` (trình phân giải SQL). Việc chuyển dịch dần từ Node.js sang Rust cho thấy Cube ưu tiên xử lý dữ liệu cực nhanh và quản lý bộ nhớ tối ưu.
    *   **TypeScript/JavaScript (35%+):** Dùng cho tầng API Gateway, Schema Compiler (trình biên dịch schema) và các Client SDK (React, Vue, Angular).
*   **Hệ thống Driver đồ sộ:** Hỗ trợ hầu hết các cơ sở dữ liệu phổ biến (BigQuery, Snowflake, ClickHouse, Postgres, Athena...) thông qua các gói driver riêng biệt trong thư mục `packages/`.
*   **CubeStore:** Một công cụ lưu trữ phân tán viết bằng Rust, tối ưu cho việc lưu trữ và truy vấn các bản tổng hợp dữ liệu (pre-aggregations).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Cube xoay quanh tư duy **"Headless BI"** (BI không đầu): tách biệt hoàn toàn việc định nghĩa dữ liệu (Metrics/Dimensions) khỏi việc hiển thị dữ liệu.

*   **Tính trừu tượng (Abstraction):** Thay vì viết các câu lệnh SQL phức tạp trong code ứng dụng, lập trình viên định nghĩa "Data Model". Cube sẽ tự động dịch các yêu cầu này thành SQL tối ưu cho từng loại database.
*   **Kiến trúc 4 lớp chính:**
    1.  **Data Modeling:** Định nghĩa logic nghiệp vụ (measures, dimensions).
    2.  **Access Control:** Quản lý quyền truy cập dữ liệu ở mức dòng/cột.
    3.  **Caching & Pre-aggregations:** Lớp đệm để đảm bảo các truy vấn lớn trả về kết quả trong mili giây.
    4.  **APIs:** Cung cấp dữ liệu qua REST, GraphQL và SQL (Postgres-compatible).
*   **Tư duy Monorepo:** Sử dụng Lerna và Yarn Workspaces để quản lý hàng chục gói package, giúp duy trì tính đồng nhất giữa backend và frontend SDK.

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **SQL Rewriting & Transpilation:** Cube sử dụng các kỹ thuật biên dịch để chuyển đổi mã JavaScript/YAML (Data Model) thành các truy vấn SQL thô. Thành phần `cubejs-schema-compiler` là trái tim của quá trình này.
*   **Query Orchestration (Điều phối truy vấn):** Kỹ thuật quản lý hàng đợi (queueing) và lập lịch làm mới cache (refresh scheduler) để tránh làm quá tải cơ sở dữ liệu nguồn (Data Source).
*   **Postgres Protocol Emulation:** Thành phần `cubesql` (viết bằng Rust) mô phỏng giao thức Postgres, cho phép bất kỳ công cụ BI nào (Tableau, PowerBI) kết nối với Cube như một cơ sở dữ liệu Postgres thông thường.
*   **Distributed Computing (Tính toán phân tán):** CubeStore sử dụng kiến trúc Router-Worker (viết bằng Rust) để phân tán khối lượng công việc truy vấn dữ liệu lớn.

### 4. Luồng hoạt động của hệ thống (System Workflow)

Luồng đi của một yêu cầu dữ liệu điển hình:

1.  **Nhận yêu cầu (Incoming Request):** Một ứng dụng khách (ví dụ: dashboard React) gửi một "Query Object" qua REST API hoặc một câu lệnh SQL qua SQL API.
2.  **Xác thực và Phân quyền (Auth & Security):** Cube kiểm tra `securityContext` để xác định người dùng là ai và họ có quyền xem những dòng dữ liệu nào.
3.  **Biên dịch (Compilation):** `Schema Compiler` lấy Query Object, kết hợp với Data Model (JS/YAML) để tạo ra một bản kế hoạch truy vấn (Query Plan).
4.  **Kiểm tra Cache/Pre-aggregation:** 
    *   Nếu kết quả đã có trong cache hoặc có một bản `Pre-aggregation` (tổng hợp trước) trong `CubeStore`, Cube sẽ lấy dữ liệu từ đó ngay lập tức.
5.  **Thực thi truy vấn gốc (Source Execution):** Nếu không có cache, Cube gửi lệnh SQL đã tối ưu đến cơ sở dữ liệu gốc (ví dụ: Snowflake).
6.  **Xử lý kết quả (Post-processing):** Dữ liệu trả về được định dạng lại (type conversion, pivoting) và gửi lại cho ứng dụng khách.

### Tóm lại
Cube là một dự án có độ phức tạp kỹ thuật rất cao, kết hợp giữa **tính linh hoạt của hệ sinh thái JavaScript** và **hiệu năng mạnh mẽ của Rust**. Nó giải quyết vấn đề "phân mảnh logic" trong các hệ thống phân tích bằng cách tập trung toàn bộ định nghĩa dữ liệu vào một tầng duy nhất.