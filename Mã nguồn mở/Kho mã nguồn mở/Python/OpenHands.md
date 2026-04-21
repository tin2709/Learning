OpenHands (trước đây là OpenDevin) là một nền tảng đại lý phần mềm (AI Software Agent) mã nguồn mở vô cùng mạnh mẽ. Qua phân tích mã nguồn và cấu trúc thư mục từ tệp bạn cung cấp, dưới đây là cái nhìn sâu sắc về hệ thống này:

### 1. Công nghệ cốt lõi (Core Stack)

OpenHands được xây dựng dựa trên sự kết hợp giữa hiệu suất của Python và tính linh hoạt của hệ sinh thái web hiện đại:

*   **Backend:** Sử dụng **Python 3.12+**. 
    *   **Framework:** **FastAPI** và **Uvicorn** cho hệ thống REST API và WebSocket (Socket.io).
    *   **Quản lý phụ thuộc:** **Poetry** (cho cả bản OSS và bản Enterprise).
    *   **Hệ thống AI:** **LiteLLM** là "trái tim" để kết nối với hàng trăm mô hình ngôn ngữ khác nhau (Claude, GPT, Llama...), hỗ trợ tính năng quan trọng như **Function Calling** và **Prompt Caching**.
*   **Frontend:** Kiến trúc **React** hiện đại.
    *   **Build Tool:** **Vite** kết hợp với **TypeScript**.
    *   **Quản lý trạng thái & Cache:** **TanStack Query (React Query)** và **Zustand**.
    *   **CSS:** **Tailwind CSS**.
*   **Hệ thống Sandbox (Môi trường thực thi):** 
    *   **Docker:** Sử dụng các container cô lập để Agent chạy mã code an toàn.
    *   **Kubernetes:** Hỗ trợ mở rộng quy mô lớn (Enterprise).
    *   **Thứ ba:** Tích hợp với các runtime chuyên dụng như E2B, Modal, Daytona.
*   **Enterprise Suite:**
    *   **Auth:** **Keycloak** (OAuth2/OIDC).
    *   **Database:** **PostgreSQL** với **SQLAlchemy** (ORM) và **Alembic** (Migrations).
    *   **Billing:** Tích hợp **Stripe**.
    *   **Queue/Cache:** **Redis**.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của OpenHands đi theo hướng **"Đa tầng và Có thể kết hợp" (Layered & Composable)**:

*   **Tách biệt Runtime và Logic:** OpenHands tách biệt hoàn toàn giữa "Bộ não" (Agent logic) và "Cánh tay" (Runtime/Sandbox). Agent chỉ gửi các Action (chạy lệnh, sửa file) và nhận lại Observation. Điều này giúp hệ thống cực kỳ an toàn vì code do AI viết không bao giờ chạy trực tiếp trên máy chủ của bạn.
*   **Kiến trúc hướng sự kiện (Event-Driven):** Mọi tương tác giữa Người dùng - Agent - Môi trường đều thông qua `EventStream`. Các sự kiện được lưu trữ trong `EventStore`, cho phép tính năng **Replay** (xem lại quá trình giải quyết vấn đề).
*   **Thiết kế "Enterprise-First":** Thư mục `enterprise/` cho thấy sự chuẩn bị kỹ lưỡng cho môi trường doanh nghiệp với Multi-tenant (nhiều tổ chức), RBAC (phân quyền dựa trên vai trò), và hệ thống quản lý Integration (GitHub, Slack, Jira) phức tạp.
*   **Abstraction Layer cho LLM:** Thông qua LiteLLM, OpenHands không bị phụ thuộc vào bất kỳ nhà cung cấp AI nào, cho phép chuyển đổi linh hoạt giữa OpenAI, Anthropic hay các mô hình tự host.

### 3. 
*   **Microagents & Skills:** Hệ thống sử dụng các tệp Markdown (`skills/`, `.openhands/microagents/`) để nạp kiến thức chuyên sâu cho AI theo ngữ cảnh mà không cần đào tạo lại mô hình.
*   **Hệ thống Condenser (Nén ngữ cảnh):** Khi lịch sử hội thoại quá dài, hệ thống sử dụng các kỹ thuật như **LLM Summarization** hoặc **Observation Masking** để nén dữ liệu, giúp tiết kiệm token và giữ cho AI không bị lạc hướng.
*   **Maintenance Task Processor:** (Trong bản Enterprise) Có một trình xử lý tác vụ bảo trì định kỳ (`run_maintenance_tasks.py`) để xử lý các việc như làm sạch dữ liệu, đồng bộ hóa người dùng từ Keycloak.
*   **Prompt Engineering với Jinja2:** Sử dụng template engine Jinja2 một cách triệt để để xây dựng các prompt hệ thống phức tạp, cho phép thay đổi hành vi của Agent linh hoạt thông qua cấu hình.

### 4.  (System Flow)

1.  **Khởi động (Startup):** 
    *   Người dùng đăng nhập (qua Keycloak trong Enterprise). 
    *   Client yêu cầu tạo một Conversation. 
    *   App Server kích hoạt việc tạo một **Sandbox** (Docker container).
2.  **Vòng lặp Action-Observation (The Agent Loop):**
    *   **User Input:** Người dùng gửi yêu cầu (ví dụ: "Sửa lỗi login").
    *   **Agent Thinking:** Agent nhận context, sử dụng LLM để phân tích và đưa ra một **Action** (ví dụ: `run_command("ls")` hoặc `edit_file(...)`).
    *   **Runtime Execution:** Action này được gửi vào Sandbox. Sandbox thực thi và trả về một **Observation** (ví dụ: danh sách file hoặc nội dung lỗi).
    *   **Observation Loop:** Agent nhận Observation, cập nhật `State` và tiếp tục suy nghĩ cho đến khi hoàn thành nhiệm vụ.
3.  **Tích hợp ngoại vi (Integration Webhooks):**
    *   Khi có một Issue mới trên GitHub, một Webhook được gửi đến **Integration Router**.
    *   Router gọi **Manager** tương ứng để tự động tạo một Conversation và gán Agent vào xử lý mà không cần con người can thiệp thủ công.
4.  **Kết thúc & Phản hồi:** 
    *   Agent phát ra `AgentFinishAction`.
    *   Trong Enterprise, hệ thống có thể tự động tạo Pull Request trên GitHub/GitLab với mã đã sửa.

### Kết luận
OpenHands không chỉ là một công cụ chat với AI, mà là một **hệ điều hành cho các lập trình viên AI**. Nó kết hợp chặt chẽ giữa quản lý tài nguyên (Sandbox), quản lý tri thức (Microagents) và quản lý quy trình (Integrations) để tạo ra một cộng tác viên AI thực thụ. Mức độ chi tiết trong các file migration và hệ thống auth cho thấy đây là một sản phẩm đã sẵn sàng cho quy mô thương mại lớn.