Dagu là một engine điều phối luồng công việc (workflow engine) hiện đại, tập trung vào sự đơn giản, khả năng tự đóng gói và tích hợp trí tuệ nhân tạo (AI). Dưới đây là phân tích chi tiết:

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Backend (Ngôn ngữ chính):** **Go (Golang)**. Đây là lựa chọn tối ưu để tạo ra một file thực thi (binary) duy nhất, có hiệu suất cao, tiêu tốn ít tài nguyên (dưới 128MB RAM) và hỗ trợ đa nền tảng tốt.
    *   *Chi router:* Dùng cho hệ thống REST API.
    *   *Cobra & Viper:* Dùng để xây dựng giao diện dòng lệnh (CLI) và quản lý cấu hình.
    *   *gRPC:* Sử dụng cho giao tiếp hiệu năng cao giữa Coordinator và Worker trong chế độ phân tán.
*   **Frontend:** **React 19 & TypeScript**.
    *   *Tailwind CSS 4:* Cho giao diện người dùng hiện đại và linh hoạt.
    *   *Monaco Editor:* Trình soạn thảo mã nguồn tích hợp (giống VS Code) để chỉnh sửa file YAML trực tiếp trên trình duyệt.
    *   *Xterm.js:* Để hiển thị terminal trực tiếp trên web.
*   **AI Integration:** Hỗ trợ các model LLM hàng đầu như **Anthropic, OpenAI, Google Gemini** để tự động hóa việc viết, sửa và gỡ lỗi workflow.
*   **Lưu trữ (Persistence):** **File-based storage**. Không yêu cầu cơ sở dữ liệu ngoài (như Postgres/MySQL) hay message broker (như Redis/RabbitMQ). Mọi trạng thái được lưu trực tiếp vào các file cục bộ (JSON/YAML) hoặc SQLite nhúng.

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Triết lý Zero-Ops:** Mục tiêu là loại bỏ gánh nặng vận hành. Chỉ cần một file binary duy nhất (`dagu start-all`) là có đầy đủ Web UI, Scheduler và Executor.
*   **Kiến trúc DAG (Directed Acyclic Graph):** Các công việc được tổ chức theo đồ thị có hướng không chu trình. Điều này cho phép tính toán các bước chạy song song (parallel) dựa trên sự phụ thuộc (dependencies) một cách khoa học.
*   **Cơ chế "Shared-Nothing" & Distributed:** Hệ thống có thể chạy ở chế độ một máy (local) hoặc mở rộng theo mô hình **Coordinator - Worker**. Coordinator quản lý hàng đợi và trạng thái, trong khi nhiều Worker trên các máy khác nhau có thể đăng ký để nhận và thực thi tác vụ.
*   **AI-Native Design:** AI không chỉ là tính năng bổ sung mà được nhúng sâu vào hệ thống thông qua các "Agent step type" và "Skills", cho phép các luồng công việc có thể tự ra quyết định hoặc tương tác với LLM như một mắt xích trong quy trình.
*   **Air-gapped Ready:** Kiến trúc không phụ thuộc bên ngoài giúp Dagu có thể chạy hoàn hảo trong các môi trường cô lập, bảo mật cao.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Dependency Resolution (Giải quyết phụ thuộc):** Thuật toán tự động phân tích đồ thị để quyết định bước nào đủ điều kiện chạy ngay lập tức, tối ưu hóa thời gian thực hiện bằng cách chạy song song tối đa.
*   **Git Sync:** Kỹ thuật đối soát (reconciliation) trạng thái giữa thư mục workflow cục bộ và một repository Git, giúp quản lý phiên bản workflow một cách chuyên nghiệp (Workflow as Code).
*   **SSE (Server-Sent Events):** Sử dụng để đẩy dữ liệu thời gian thực (log, trạng thái step) từ server xuống Web UI mà không cần người dùng reload trang.
*   **Variable Expansion & Expression Evaluation:** Cho phép nhúng các biến môi trường, kết quả của các bước trước đó vào tham số của bước sau bằng cú pháp `${step_id.outputs.result}`.
*   **Container & Remote Execution:** Tích hợp sẵn Executor cho Docker (chạy step trong container) và SSH (chạy step trên server từ xa), giúp Dagu điều phối được mọi loại script/ngôn ngữ.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Định nghĩa (Authoring):** Người dùng viết file YAML (hoặc dùng AI Agent để tạo từ ngôn ngữ tự nhiên).
2.  **Tải & Xác thực (Load & Validate):** CLI hoặc Web API nạp file YAML, kiểm tra tính đúng đắn của đồ thị (không có vòng lặp, đúng cú pháp).
3.  **Lập lịch (Scheduling):** Scheduler kiểm tra các điều kiện cron hoặc trigger để quyết định thời điểm thực hiện.
4.  **Điều phối (Dispatching):**
    *   *Chế độ Local:* Runner trực tiếp khởi tạo các tác vụ.
    *   *Chế độ Phân tán:* Coordinator đưa tác vụ vào hàng đợi, Worker dùng gRPC để kéo (poll) tác vụ về máy mình.
5.  **Thực thi (Execution):** Tùy loại tác vụ (Shell, Docker, SQL, AI...), Executor tương ứng sẽ được gọi để chạy lệnh và bắt log.
6.  **Cập nhật trạng thái (State Update):** Kết quả thực thi (Thành công/Thất bại/Log) được ghi lại vào file storage và thông báo cho UI qua SSE.
7.  **Kết thúc:** Runner kiểm tra các bước tiếp theo dựa trên kết quả vừa nhận được cho đến khi hoàn thành toàn bộ đồ thị.

Dagu là giải pháp hoàn hảo cho các đội ngũ muốn có sức mạnh của Airflow hay Jenkins nhưng không muốn tốn thời gian cấu hình hệ thống phức tạp.