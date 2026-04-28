OpenMetadata là một nền tảng quản lý siêu dữ liệu (metadata) hợp nhất, mã nguồn mở, được thiết kế theo tiêu chuẩn hiện đại để giải quyết các vấn đề về khám phá, quan sát và quản trị dữ liệu.

Dưới đây là phân tích chi tiết về hệ thống OpenMetadata dựa trên kho mã nguồn:

### 1. Công nghệ cốt lõi (Core Technology Stack)
Hệ thống được xây dựng dựa trên sự kết hợp của ba ngôn ngữ chính, phục vụ các mục đích chuyên biệt:
*   **Backend (Java 21):** Sử dụng framework **Dropwizard** (kết hợp Jetty, Jersey, Jackson) để xây dựng RESTful API hiệu suất cao. Việc sử dụng **JDBI3** thay vì các ORM nặng nề (như Hibernate) giúp kiểm soát truy vấn SQL tối ưu hơn.
*   **Ingestion Framework (Python 3.10-3.12):** Sử dụng **Pydantic v2** để kiểm tra dữ liệu và **SQLAlchemy** để kết nối với hàng chục nguồn dữ liệu khác nhau. Hệ thống này thường được chạy trên **Apache Airflow**.
*   **Frontend (React/TypeScript):** Sử dụng **Zustand** để quản lý state và đang chuyển dịch sang thư viện component riêng dựa trên **Tailwind CSS v4** và **react-aria-components**.
*   **Lưu trữ & Tìm kiếm:** Siêu dữ liệu được lưu trữ trong **MySQL/PostgreSQL** và được đánh chỉ mục trong **Elasticsearch/OpenSearch** để tìm kiếm toàn văn và gợi ý.
*   **Giao thức siêu dữ liệu:** Sử dụng **JSON Schema** làm nguồn chân lý duy nhất (Single Source of Truth) để tự động tạo mã nguồn (code gen) cho cả Java, Python và TypeScript.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của OpenMetadata tuân thủ triết lý "Schema-first" và "Metadata-as-Code":
*   **Kiến trúc Schema-First:** Mọi thực thể (Table, Pipeline, Dashboard) đều được định nghĩa bằng JSON Schema. Điều này đảm bảo tính nhất quán tuyệt đối giữa API, cơ sở dữ liệu và giao diện người dùng.
*   **Mô hình Thực thể (Entity Model):** Hệ thống quản lý siêu dữ liệu dưới dạng một đồ thị (Graph). Thực thể được xác định bằng **FQN (Fully Qualified Name)** theo cấu trúc phân cấp (ví dụ: `service.database.schema.table`).
*   **Hybrid Migration System:** Kết hợp giữa migration SQL thuần túy và parser của Flyway để xử lý sự khác biệt giữa các phương ngôn SQL (MySQL vs Postgres) mà vẫn đảm bảo tính toàn vẹn dữ liệu.
*   **Kiến trúc Phân lớp:**
    *   *Resource Layer:* Xử lý HTTP request/response.
    *   *Repository Layer:* Chứa logic nghiệp vụ và quản lý quan hệ thực thể.
    *   *DAO Layer:* Giao tiếp trực tiếp với DB qua JDBI.
*   **Tách biệt lưu trữ và tìm kiếm:** Metadata luôn được ghi vào DB quan hệ trước, sau đó một hệ thống sự kiện sẽ cập nhật chỉ mục tìm kiếm (Search Index) một cách không đồng bộ.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)
*   **Topology Pattern (Python):** Trong khung ingestion, kỹ thuật này định nghĩa thứ tự duyệt qua các thực thể dữ liệu (Service -> Database -> Schema -> Table) một cách khai báo (declarative), giúp việc viết connector mới rất nhanh chóng.
*   **Either Monad (Python):** Sử dụng để xử lý lỗi trong luồng thu thập dữ liệu mà không làm sập toàn bộ tiến trình (Fail-safe).
*   **JSON Patch (Java):** Sử dụng tiêu chuẩn RFC 6902 để cập nhật từng phần thực thể. Điều này giúp tránh xung đột dữ liệu khi nhiều người dùng/hệ thống cùng cập nhật một thực thể.
*   **ANTLR4:** Sử dụng để parse các ngôn ngữ truy vấn và đặc biệt là parse các FQN phức tạp, đảm bảo tốc độ và độ chính xác cao.
*   **Kafka-Grade Coding Standards:** Backend áp dụng các quy tắc khắt khe như: Phương thức không quá 15 dòng, tối đa 3 tầng lồng nhau (nesting), không sử dụng "magic strings".

### 4. Luồng hoạt động hệ thống (System Operation Flow)

1.  **Giai đoạn Ingestion (Thu thập):**
    *   Ingestion Framework (Python) kết nối với nguồn dữ liệu (ví dụ: Snowflake).
    *   Duyệt qua cấu trúc dựa trên **Topology**.
    *   Tạo ra các đối tượng Metadata và gửi về Backend qua REST API.
2.  **Giai đoạn API Processing (Xử lý):**
    *   Backend nhận yêu cầu, xác thực qua **RBAC (Role-Based Access Control)**.
    *   Repository kiểm tra các mối quan hệ (Relationships) và thực hiện logic nghiệp vụ.
    *   Dữ liệu được lưu vào SQL DB.
3.  **Giai đoạn Event & Search (Sự kiện & Tìm kiếm):**
    *   Sau khi lưu thành công, một **ChangeEvent** được tạo ra.
    *   Hệ thống lắng nghe sự kiện sẽ đẩy dữ liệu mới vào **Elasticsearch/OpenSearch**.
    *   Đồng thời gửi thông báo qua Webhook (Slack, MS Teams) hoặc cập nhật Activity Feed thời gian thực qua WebSocket.
4.  **Giai đoạn Consumption (Tiêu thụ):**
    *   Người dùng tìm kiếm trên UI, React gọi API tìm kiếm.
    *   Các công cụ bên thứ ba (Data Quality, Profiler) truy vấn Backend để thực hiện các bài kiểm tra dữ liệu dựa trên siêu dữ liệu đã thu thập.

### Tổng kết
OpenMetadata không chỉ là một kho lưu trữ, mà là một **Metadata Operating System**. Nó tự động hóa việc kết nối các thành phần rời rạc trong hệ sinh thái dữ liệu thông qua một lớp giao tiếp chuẩn hóa duy nhất dựa trên JSON Schema.