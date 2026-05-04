Dưới đây là phân tích chi tiết về hệ thống **DataLens** dựa trên mã nguồn và tài liệu bạn cung cấp, tập trung vào 4 khía cạnh: Công nghệ cốt lõi, Tư duy kiến trúc, Kỹ thuật lập trình và Luồng hoạt động.

---

### 1. Công nghệ cốt lõi (Core Technology)

Hệ thống DataLens là một nền tảng BI (Business Intelligence) đa ngôn ngữ, kết hợp giữa sự linh hoạt của Node.js và sức mạnh xử lý dữ liệu của Python.

*   **Ngôn ngữ lập trình:**
    *   **Python:** Sử dụng cho các dịch vụ Backend (Data API, Control API). Python được chọn vì hệ sinh thái thư viện xử lý dữ liệu mạnh mẽ và khả năng làm việc với các hệ quản trị cơ sở dữ liệu (DBMS) khác nhau.
    *   **Node.js (TypeScript/JavaScript):** Sử dụng cho UI, UnitedStorage (US), Auth và MetaManager. Phù hợp cho các tác vụ I/O non-blocking và quản lý metadata.
    *   **SQL (PL/pgSQL):** Được sử dụng sâu trong PostgreSQL để khởi tạo schema, quản lý extension (`pg_trgm`, `btree_gin`, `uuid-ossp`) và demo data.
*   **Cơ sở hạ tầng & Điều phối (Orchestration):**
    *   **Docker & Docker Compose:** Công cụ chính để đóng gói và triển khai nhanh.
    *   **Kubernetes (K8s) & Helm:** Dành cho triển khai quy mô lớn (Production), hỗ trợ quản lý cấu hình phức tạp qua `values.yaml`.
    *   **Temporal:** Một Workflow Engine mạnh mẽ dùng để quản lý các tác vụ chạy ngầm, dài hơi và có trạng thái (như xuất/nhập workbook).
*   **Lưu trữ & Dữ liệu:**
    *   **PostgreSQL:** "Trái tim" lưu trữ metadata (UnitedStorage) và trạng thái workflow (Temporal).
    *   **Highcharts & D3.js:** Thư viện render biểu đồ. DataLens đang có lộ trình chuyển dần từ Highcharts sang D3.js (Open Source).
*   **Infrastructure as Code (IaC):** Sử dụng **Terraform (OpenTofu)** để quản lý tài nguyên trên đám mây (VPC, K8s cluster, S3 bucket, Security Groups).

---

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của DataLens được thiết kế theo hướng **Microservices** phân rã theo chức năng (Functional Decomposition):

*   **Tách biệt giữa Giao diện và Logic xử lý (Decoupling UI/Backend):** UI đóng vai trò là một SPA (Single Page Application) kết hợp với một lớp proxy Node.js để xử lý bảo mật và tiền xử lý dữ liệu nhẹ.
*   **Kiến trúc Stateless (Statelessness):** Các API được thiết kế không lưu trạng thái. Toàn bộ trạng thái hệ thống và metadata được đẩy xuống lớp **UnitedStorage (US)**. Điều này giúp hệ thống dễ dàng scale ngang.
*   **Lớp trừu tượng hóa dữ liệu (Data Abstraction Layer):** Backend không truy vấn trực tiếp theo kiểu cứng nhắc mà tạo ra một "Abstract Dataset". Lớp Data-API sẽ dịch các yêu cầu từ UI thành các câu lệnh SQL tương ứng cho từng loại Database nguồn (ClickHouse, PostgreSQL, MySQL...).
*   **Kiến trúc Workflow-based:** Đối với các tác vụ phức tạp như Export Workbook, thay vì xử lý đồng bộ dễ gây timeout, hệ thống đẩy vào **Temporal**. Điều này đảm bảo tính tin cậy (reliability) – nếu một worker chết, task sẽ được retry từ bước gần nhất.
*   **Security-First:** Sử dụng cặp khóa RSA (Private/Public Key) để ký và xác thực JWT giữa các dịch vụ. Dịch vụ Auth tách biệt hoàn toàn để quản lý Identity và Role-based Access Control (RBAC).

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Cấu hình qua biến môi trường (Environment-Driven Configuration):** Hệ thống tận dụng triệt để file `.env` và các script shell (`init.sh`) để kiểm soát mọi hành vi của container mà không cần sửa code.
*   **Kỹ thuật bảo mật Metadata:**
    *   Sử dụng `crypto.py` dựa trên thư viện **Fernet** (Cryptography) để mã hóa mật khẩu của các kết nối cơ sở dữ liệu nguồn trước khi lưu vào metadata của UnitedStorage.
*   **Structured Logging & Debugging:**
    *   Sử dụng `pretty-log.py` để định dạng lại log JSON phức tạp thành dạng dễ đọc cho lập trình viên trong quá trình phát triển (Dev mode).
*   **Automation Scripting:**
    *   File `init.sh` là một ví dụ điển hình về kỹ thuật **Automation Glue**. Nó tự động sinh secret, tạo RSA key, kiểm tra phiên bản Docker, và cấu hình mạng (IPv6) chỉ bằng các tham số dòng lệnh.
*   **Database Migration & Seeding:**
    *   Sử dụng các shell script (`init-db-*.sh`) để tự động kiểm tra sự tồn tại của database, extension và thực hiện seeding dữ liệu demo ngay khi container khởi động.

---

### 4. Luồng hoạt động hệ thống (System Operation Flow)

#### A. Luồng Khởi động (Bootstrap Flow):
1.  Người dùng chạy `./init.sh --up`.
2.  Script sinh các khóa RSA, mật khẩu PostgreSQL và lưu vào `.env`.
3.  Docker Compose khởi động **Postgres** đầu tiên.
4.  Container Postgres chạy script `init-postgres.sh` để tạo ra hàng loạt database con: `pg-us-db`, `pg-auth-db`, `pg-temporal-db`...
5.  **Temporal** khởi động, thực hiện `setup-schema` trên database của nó.
6.  Các dịch vụ API (`control-api`, `data-api`) chờ Postgres sẵn sàng mới bắt đầu phục vụ.

#### B. Luồng Truy vấn Dữ liệu (Data Query Flow):
1.  **UI:** Người dùng kéo thả biểu đồ.
2.  **UI-API:** Nhận yêu cầu, xác thực JWT thông qua dịch vụ **Auth**.
3.  **Control-API:** Lấy metadata của Dataset từ **UnitedStorage**.
4.  **Data-API:** Kết hợp metadata và yêu cầu từ UI để sinh mã SQL (ClickHouse/Postgres...).
5.  **Database Nguồn:** Thực thi SQL và trả về kết quả thô.
6.  **Data-API:** Hậu xử lý dữ liệu (tính toán công thức, format) và trả về JSON cho UI.

#### C. Luồng Quản lý Metadata (Workbook Export):
1.  Người dùng nhấn "Export Workbook".
2.  **UI-API** gửi yêu cầu đến **MetaManager**.
3.  **MetaManager** khởi tạo một workflow trên **Temporal**.
4.  **Temporal Worker** thực hiện các bước: đọc dữ liệu từ UnitedStorage -> đóng gói thành file JSON -> trả về kết quả.

### Tổng kết
DataLens là một hệ thống có kiến trúc **Chặt chẽ (Robust)** và **Khả mở (Scalable)**. Điểm sáng lớn nhất là việc kết hợp giữa **UnitedStorage** (quản lý metadata tập trung) và **Temporal** (quản lý quy trình nghiệp vụ phức tạp), giúp biến một công cụ BI vốn thường nặng nề trở nên linh hoạt và tin cậy trong môi trường Cloud Native.