Dựa trên mã nguồn và cấu trúc thư mục của dự án **Hatchet**, đây là phân tích chuyên sâu về công nghệ cốt lõi, kiến trúc và luồng hoạt động của hệ thống này:

### 1. Công nghệ cốt lõi (Core Technology Stack)

Hatchet được xây dựng như một hệ thống phân tán hiện đại với các lựa chọn công nghệ tập trung vào độ tin cậy:

*   **Ngôn ngữ lập trình:**
    *   **Backend:** Go (Golang) là ngôn ngữ chủ đạo cho Engine và API nhờ hiệu năng cao và khả năng xử lý concurrency tốt.
    *   **SDKs:** Đa dạng (Python, TypeScript, Go, Ruby) giúp hệ thống dễ dàng tích hợp vào nhiều loại ứng dụng khác nhau.
    *   **Frontend:** React/TypeScript với Vite, Tailwind CSS.
*   **Lưu trữ và Queue:**
    *   **Postgres (Trọng tâm):** Hatchet sử dụng Postgres không chỉ làm database mà còn là "Durable Task Queue". Thay vì dùng Redis (in-memory), việc dùng Postgres đảm bảo tính "Durable" (bền vững) - không mất task ngay cả khi hệ thống sập.
    *   **RabbitMQ:** Được sử dụng như một bus tin nhắn nội bộ giữa các service (API <-> Engine) để thông báo trạng thái thay đổi nhanh.
*   **Giao thức truyền thông:**
    *   **gRPC:** Sử dụng Protobuf để giao tiếp giữa SDK (Worker) và Engine. Điều này tối ưu hóa băng thông và độ trễ.
    *   **REST/OpenAPI:** Dùng cho giao diện Dashboard và các tích hợp bên ngoài qua API v1.
*   **Logic động (Expression Language):**
    *   **Google CEL (Common Expression Language):** Một công nghệ rất hay trong Hatchet. Nó cho phép người dùng viết các biểu thức logic (như `input.user_id`, `event.type == 'signup'`) để điều hướng workflow hoặc giới hạn concurrency mà không cần compile lại code.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Hatchet thể hiện tư duy của một nền tảng **Cloud-Native Task Orchestrator**:

*   **Multi-tenancy (Đa người dùng):** Hệ thống được thiết kế từ gốc để hỗ trợ nhiều Tenant (tổ chức) khác nhau. Mọi dữ liệu (Event, Workflow, Worker) đều được cô lập bằng `tenant_id`.
*   **Durable Execution (Thực thi bền bỉ):** Khác với các thư mục queue thông thường (như Celery/BullMQ), Hatchet ghi lại "lịch sử" thực thi. Nếu một worker chết giữa chừng, Engine biết chính xác nó đang dừng ở bước nào trong DAG để khởi động lại.
*   **Separation of Concerns (Phân tách trách nhiệm):**
    *   **API Server:** Xử lý xác thực (AuthN/AuthZ), CRUD các định nghĩa workflow và cung cấp dữ liệu cho Dashboard.
    *   **Engine Server:** Trái tim của hệ thống, xử lý logic lập lịch (scheduling), quản lý trạng thái của DAG và điều phối (dispatching) task đến các worker.
*   **Observability (Khả năng quan sát):** Tích hợp sâu OpenTelemetry (OTEL). Mỗi task run tạo ra các span/trace giúp lập trình viên debug được luồng chạy xuyên suốt từ lúc trigger đến lúc hoàn thành.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

Dự án này là một tài liệu tham khảo tốt cho các kỹ sư Go:

*   **Code Generation (Tối ưu hóa năng suất):**
    *   Sử dụng `sqlc` để tạo code Go từ truy vấn SQL thuần, giúp đảm bảo type-safe khi làm việc với DB.
    *   Sử dụng OpenAPI Generator để tạo code boilerplate cho API từ file `.yaml`.
*   **Middleware-driven Security:**
    *   Hệ thống AuthN/AuthZ được module hóa cực tốt (xem trong `api/v1/server/authn` và `authz`). Quyền hạn (RBAC) được cấu hình bằng file YAML (`rbac.yaml`) giúp quản lý tập trung các hành động như `WorkflowRunCancel`, `EventCreate`.
*   **Repository Pattern:** Toàn bộ logic truy cập dữ liệu được đóng gói trong `internal/services/controllers/task`, tách biệt logic nghiệp vụ khỏi tầng persistence.
*   **Encryption at Rest:** Có cơ chế mã hóa các thông tin nhạy cảm (như API tokens, webhook signing secrets) bằng khóa Master Key trước khi lưu vào Postgres.

### 4. Luồng hoạt động hệ thống (System Workflow)

Hệ thống hoạt động theo một vòng lặp sự kiện (Event Loop) chặt chẽ:

1.  **Ingestion (Nạp sự kiện):**
    *   Một ứng dụng bên ngoài gửi một **Event** qua REST API hoặc SDK.
    *   `Ingestor Service` nhận event, lưu vào Postgres và phát một thông báo qua RabbitMQ/Postgres Listen-Notify.
2.  **Triggering (Kích hoạt):**
    *   Engine nhận thấy có event mới, kiểm tra các `Workflow Definition` xem có cái nào khớp với event key này không (sử dụng CEL expressions).
    *   Nếu khớp, một `Workflow Run` và các `Task Runs` tương ứng sẽ được khởi tạo trong trạng thái `PENDING`.
3.  **Scheduling & Dispatching (Lập lịch & Điều phối):**
    *   Engine tìm kiếm các **Worker** đang rảnh (giao tiếp qua gRPC stream liên tục).
    *   Dựa trên các chiến lược (Sticky assignment - ưu tiên worker cũ, hoặc Rate limiting - giới hạn tốc độ), Engine gửi lệnh thực thi đến Worker.
4.  **Execution & Heartbeat (Thực thi):**
    *   Worker nhận task, thực thi logic nghiệp vụ.
    *   Trong quá trình chạy, Worker gửi "Heartbeat" về Engine để báo rằng "Tôi vẫn đang sống". Nếu mất heartbeat, Engine sẽ tự động requeue task đó cho worker khác.
5.  **Completion & State Update:**
    *   Worker chạy xong, gửi kết quả (Output) về Engine.
    *   Engine cập nhật trạng thái bước đó vào DB, kiểm tra xem bước tiếp theo trong DAG là gì để tiếp tục quy trình cho đến khi hoàn thành workflow.

### Tổng kết
Hatchet không chỉ là một cái queue, nó là một **Workflow OS**. Nó giải quyết bài toán khó nhất trong hệ thống phân tán: **"Làm sao để đảm bảo một chuỗi các tác vụ phức tạp luôn hoàn thành đúng trình tự, bất kể server hay network bị lỗi?"**. Việc sử dụng Postgres làm trung tâm khiến nó trở nên cực kỳ tin cậy (durable) và dễ vận hành hơn so với các hệ thống đòi hỏi quá nhiều thành phần phụ trợ (như Temporal hay Airflow).