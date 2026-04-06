Dựa trên danh sách tập tin và tài liệu kiến trúc của Kestra, dưới đây là phân tích chuyên sâu về hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

Kestra là một nền tảng điều phối (orchestration) hiện đại, sử dụng các công nghệ tiên tiến nhất trong hệ sinh thái Java và Frontend:

*   **Backend (Java Stack):**
    *   **Java 25:** Sử dụng phiên bản Java rất mới, tận dụng các tính năng hiện đại như Records, Sealed Classes và Virtual Threads (thông qua Micronaut).
    *   **Micronaut Framework:** Framework chính cho Dependency Injection (DI), AOP, và Webserver. Micronaut được chọn vì tốc độ khởi động nhanh và tiêu tốn ít bộ nhớ hơn Spring.
    *   **Lombok:** Giảm thiểu mã boilerplate (Getter, Setter, Builder).
    *   **JOOQ & Flyway:** Quản lý truy vấn SQL type-safe và xử lý migration cơ sở dữ liệu.
*   **Frontend (Modern Web Stack):**
    *   **Vue 3 & Vite:** Framework UI và công cụ build tốc độ cao.
    *   **TypeScript:** Đảm bảo kiểu dữ liệu chặt chẽ cho toàn bộ logic frontend.
    *   **Element Plus:** Thư viện UI component chính.
    *   **Pinia:** Quản lý trạng thái (state management).
*   **Infrastructure & Storage:**
    *   **Dữ liệu:** Hỗ trợ đa cơ sở dữ liệu thông qua JDBC (Postgres, MySQL, H2).
    *   **Hàng đợi (Queueing):** Kiến trúc hàng đợi nội bộ để giao tiếp giữa các thành phần.
    *   **Docker & Kubernetes:** Hỗ trợ chạy task trong container.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Kestra được thiết kế theo hướng **Microkernel** và **Event-Driven**:

*   **Declarative Orchestration (Infrastructure as Code):** Mọi workflow (Flow) đều được định nghĩa bằng YAML. Điều này cho phép quản lý workflow như mã nguồn thực thụ (version control, CI/CD).
*   **Component-Based Architecture:** Hệ thống chia nhỏ thành các vai trò chuyên biệt:
    *   **Executor:** "Bộ não" quyết định task nào chạy tiếp theo dựa trên trạng thái hiện tại.
    *   **Worker:** "Cơ bắp" thực hiện việc chạy các task cụ thể.
    *   **Scheduler:** Theo dõi thời gian để kích hoạt các trigger theo lịch trình.
    *   **Indexer:** Đưa dữ liệu thực thi vào kho lưu trữ để tra cứu/hiển thị trên UI.
*   **Plugin-First:** Kestra không cố gắng làm mọi thứ. Nó cung cấp một khung (framework) để các Plugin (Python, Bash, SQL, Cloud) thực hiện logic nghiệp vụ. Nhân (Core) chỉ lo việc điều phối.
*   **Isolation (Cách ly):** Thiết kế cho phép chạy Worker tách biệt với Executor, giúp hệ thống có khả năng mở rộng (scaling) linh hoạt.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Facade Pattern cho Worker:** Trong `AGENTS.md` có ghi chú quan trọng: "Workers never depend on repositories". Thay vào đó, họ sử dụng các `MetaStore` hoặc `StateStore`. Điều này giúp Worker nhẹ hơn và không bị ràng buộc vào DB.
*   **Stateless Execution:** Các Worker được thiết kế stateless nhất có thể. Chúng nhận TaskRun từ hàng đợi, thực thi và trả kết quả về hàng đợi. Trạng thái của toàn bộ flow do Executor quản lý tập trung.
*   **Custom Annotation Processing:** Module `processor` cho thấy Kestra sử dụng kỹ thuật xử lý annotation tại thời điểm biên dịch để tự động đăng ký plugin và tạo tài liệu.
*   **Abstraction Layer cho Storage:** Sử dụng `StorageInterface` để trừu tượng hóa việc lưu trữ tập tin (Local, S3, GCS), giúp hệ thống linh hoạt trên mọi môi trường đám mây.
*   **Serialization chuyên sâu:** Sử dụng Jackson với các tùy chỉnh cho YAML và đặc biệt là định dạng **Amazon Ion** để xử lý dữ liệu trung gian hiệu quả.

### 4. Luồng hoạt động hệ thống (System Workflow)

Một chu kỳ thực thi của Kestra diễn ra như sau:

1.  **Định nghĩa (Authoring):** Người dùng viết YAML trên UI hoặc đẩy file qua API. Hệ thống kiểm tra cú pháp (Validation) và lưu vào Flow Repository.
2.  **Kích hoạt (Triggering):**
    *   **Schedule:** Scheduler phát hiện đến giờ chạy.
    *   **Event:** Một trigger (ví dụ: file mới trên S3) gửi tín hiệu về.
    *   **Manual:** Người dùng nhấn "Run" trên UI.
3.  **Điều phối (Execution):** Executor tạo ra một `Execution` object. Nó phân tích đồ thị task (DAG) và đẩy task đầu tiên cần chạy vào `Queue` dưới dạng `TaskRun`.
4.  **Thực thi (Worker Processing):**
    *   Worker đang lắng nghe Queue sẽ lấy `TaskRun`.
    *   Worker tạo môi trường (ví dụ: Docker container) và chạy logic của task.
    *   Kết quả (Output) và Logs được đẩy ngược lại Queue.
5.  **Cập nhật & Lặp lại:** Executor nhận kết quả từ Queue, cập nhật trạng thái Execution trong DB, và lặp lại bước 3 cho các task tiếp theo cho đến khi hoàn thành.
6.  **Ghi chỉ mục (Indexing):** Indexer lưu trữ kết quả cuối cùng để người dùng có thể xem lại lịch sử chạy trên Dashboard.

### Tổng kết
Kestra thể hiện một tư duy lập trình rất **pragmatic (thực dụng)**: Chọn Java vì sự ổn định và hiệu năng cho hệ thống điều phối lớn, nhưng dùng YAML và Vue để tạo ra trải nghiệm **Low-code/No-code** cực kỳ thân thiện cho người dùng cuối. Việc tách biệt rạch ròi giữa Executor và Worker là chìa khóa giúp nền tảng này có thể mở rộng lên hàng triệu workflow mỗi ngày.