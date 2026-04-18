Dựa trên cấu trúc thư mục và nội dung các tập tin quan trọng (README, AGENTS, cấu trúc project), dưới đây là phân tích chi tiết về dự án **Daytona**:

### 1. Công nghệ cốt lõi (Core Technology)

Daytona được xây dựng như một nền tảng **Infrastructure-as-a-Service (IaaS)** chuyên biệt cho việc thực thi mã nguồn (đặc biệt là mã do AI tạo ra). Các công nghệ chính bao gồm:

*   **Sandbox Isolation (Cô lập môi trường):** Sử dụng công nghệ container (OCI/Docker compatible) để tạo ra các "Sandboxes". Mỗi sandbox là một máy tính ảo hoàn chỉnh có kernel riêng, stack mạng, CPU/RAM/Disk riêng biệt, đảm bảo an toàn khi chạy mã lạ.
*   **Ngôn ngữ lập trình:**
    *   **TypeScript (NestJS):** Dùng cho tầng Control Plane (API Server) để quản lý logic nghiệp vụ, người dùng, quyền hạn.
    *   **Go:** Dùng cho tầng Compute Plane (`runner`, `daemon`, `cli`) vì hiệu năng cao, khả năng tương tác hệ thống thấp (low-level) tốt và khởi động nhanh.
*   **Hệ sinh thái lưu trữ & Tin nhắn:**
    *   **PostgreSQL:** Lưu trữ dữ liệu quan hệ (TypeORM).
    *   **Redis:** Dùng cho caching, distributed locking (Redlock) và xử lý queue/notification.
    *   **Kafka:** Xử lý luồng sự kiện (event streaming) cho các tác vụ bất đồng bộ như Audit Logs.
    *   **OpenSearch:** Chuyên dụng để lưu trữ và truy vấn Audit Logs quy mô lớn.
*   **Observability (Khả năng quan sát):** Tích hợp sâu với **OpenTelemetry (OTel)** để track traces, metrics và logs của quá trình thực thi mã.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Daytona chia hệ thống thành 3 tầng (Planes) rõ rệt:

*   **Interface Plane (Tầng giao diện):** Bao gồm Dashboard (React), CLI (Go) và các SDK đa ngôn ngữ (Python, JS, Go, Java, Ruby). Đây là nơi người dùng hoặc AI Agent ra lệnh.
*   **Control Plane (Tầng điều khiển - `apps/api`):** Đóng vai trò là "bộ não". Nó tiếp nhận yêu cầu từ Interface Plane, kiểm tra quyền hạn (Auth), định mức (Quota) và điều phối các lệnh đến tầng Compute.
*   **Compute Plane (Tầng tính toán - `apps/runner` & `apps/daemon`):** 
    *   `runner`: Chạy trên các node tính toán, chịu trách nhiệm quản lý vòng đời container.
    *   `daemon`: Chạy **bên trong** mỗi sandbox để thực thi lệnh trực tiếp và báo cáo trạng thái về.
    *   `snapshot-manager`: Quản lý việc lưu trữ trạng thái của sandbox để có thể phục hồi ngay lập tức.

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **Adapter Pattern (Mẫu thiết kế bộ điều hợp):** 
    *   Thấy rõ ở `apps/api/src/audit/adapters`, hệ thống có thể chuyển đổi linh hoạt giữa việc lưu Audit Log vào Database (TypeORM) hoặc OpenSearch.
    *   Tương tự với `runner-adapter`, cho phép API giao tiếp với các phiên bản Runner khác nhau (v0, v2).
*   **Monorepo & Orchestration:** Sử dụng **Nx** để quản lý hàng chục project khác nhau trong cùng một repo. Kỹ thuật này giúp chia sẻ code (ví dụ các DTO, interfaces) giữa các tầng API và Client rất dễ dàng.
*   **Automated SDK Generation:** Sử dụng **OpenAPI/Swagger** để định nghĩa API, sau đó dùng tool tự động generate ra SDK cho 5-6 ngôn ngữ khác nhau. Điều này đảm bảo tính nhất quán giữa Server và Client.
*   **Dependency Injection (DI):** Tận dụng tối đa DI của NestJS để giữ cho mã nguồn modular, dễ unit test (thể hiện qua các file `.spec.ts` dày đặc).
*   **Nix Flakes:** Cung cấp môi trường phát triển nhất quán (Reproducible builds) cho lập trình viên, đảm bảo mọi người (và cả AI Agent) đều dùng chung phiên bản Go/Node/Python mà không cần cài đặt thủ công.

### 4. Luồng hoạt động của hệ thống (System Workflow)

Lấy ví dụ luồng **"Tạo và thực thi mã trong Sandbox"**:

1.  **Request:** Người dùng gọi `sandbox.process.code_run()` từ Python SDK.
2.  **Authentication & Validation:** API Server (`apps/api`) nhận request, xác thực API Key, kiểm tra Quota (người dùng còn đủ CPU/RAM không).
3.  **Scheduling:** API tìm kiếm một `runner` đang rảnh trong `region` được yêu cầu.
4.  **Provisioning:** 
    *   API ra lệnh cho Runner khởi tạo container từ một Snapshot hoặc Image OCI.
    *   Runner cấu hình mạng và gắn (mount) các Volume cần thiết.
5.  **Execution:**
    *   API gửi lệnh thực thi đến `daemon` bên trong sandbox.
    *   `daemon` chạy mã, thu thập kết quả (stdout/stderr).
6.  **Streaming & Observability:** Kết quả được trả về API theo thời gian thực (WebSockets), đồng thời các trace log được gửi về OTel Collector để giám sát.
7.  **Audit:** Toàn bộ hành động được đẩy vào Kafka và lưu vào OpenSearch để quản trị viên có thể kiểm tra sau này.

### Tóm lại
Daytona không chỉ là một công cụ chạy code, mà là một **hệ điều hành đám mây (Cloud OS)** dành cho AI. Nó giải quyết bài toán: "Làm sao để cho phép AI tự viết và chạy code mà không làm sập hoặc làm lộ thông tin hệ thống chủ?". Dự án thể hiện trình độ kỹ thuật rất cao trong việc kết hợp giữa Cloud-native (Docker/Kubernetes concepts) và công cụ dành cho lập trình viên (SDKs/CLI).