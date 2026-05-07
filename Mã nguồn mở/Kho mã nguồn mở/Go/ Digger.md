Digger (vừa tái định danh thành **OpenTaco**) là một giải pháp IAO (Infrastructure as Code Orchestration) mã nguồn mở độc đáo. Khác với Terraform Cloud hay Spacelift vốn cung cấp hạ tầng tính toán riêng, Digger tận dụng chính hạ tầng CI/CD sẵn có của người dùng (như GitHub Actions, GitLab CI).

Dưới đây là phân tích chuyên sâu về hệ thống này:

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

*   **Ngôn ngữ chủ đạo:** **Go (Golang)** chiếm ~75% dự án. Go được chọn nhờ khả năng thực thi song song (concurrency) tốt, biên dịch ra file nhị phân tĩnh (static binary) dễ dàng chạy trong các môi trường runner và CLI.
*   **Hệ thống Backend:** Sử dụng framework **Gin** để xử lý API và Webhook. **GORM** được dùng làm ORM để tương tác với cơ sở dữ liệu (PostgreSQL/SQLite).
*   **Quản lý Di cư dữ liệu (Migration):** Sử dụng **Atlas** (`atlas.hcl`), một công cụ hiện đại giúp quản lý schema cơ sở dữ liệu dưới dạng mã (Declarative Migration).
*   **Frontend & UI:** Xây dựng bằng **TypeScript & React (Vite)** kết hợp với **Tailwind CSS**. Giao diện này chủ yếu đóng vai trò dashboard giám sát và cấu hình chính sách.
*   **Cơ chế Thực thi IaC:** Hỗ trợ đa dạng các công cụ bao gồm **Terraform, OpenTofu, Terragrunt, và Pulumi**.
*   **Chính sách (Policy):** Tích hợp **Open Policy Agent (OPA)** để thực thi các quy tắc về an toàn hạ tầng và phân quyền dựa trên RBAC.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Digger dựa trên triết lý **"Hạ hạ tầng hóa Orchestrator"**:

*   **Tách biệt Control Plane và Data Plane:**
    *   **Control Plane (Orchestrator):** Là backend trung tâm nhận sự kiện từ VCS (GitHub/GitLab). Nó không giữ secret của đám mây và không thực thi code Terraform. Nhiệm vụ chính là: Quản lý khóa (locking) ở cấp độ PR và kích hoạt runner.
    *   **Data Plane (CLI):** Chạy trực tiếp bên trong CI của bạn. Nó giữ các biến môi trường nhạy cảm và thực thi các lệnh IaC. Điều này giúp bảo mật tối đa vì thông tin nhạy cảm không bao giờ rời khỏi môi trường CI của bạn.
*   **Kiến trúc dựa trên Sự kiện (Event-Driven):** Mọi thứ bắt đầu từ Webhook. Digger chuyển dịch dần từ cơ chế callback sang xử lý webhook hoàn toàn để tăng tính phản hồi và khả năng phục hồi (resilience).
*   **Thiết kế "Plugin-like" cho CI Backends:** Mã nguồn trong `backend/ci_backends` cho thấy tư duy trừu tượng hóa. Digger coi các hệ thống CI như những interface có thể hoán đổi (GitHub Actions, Jenkins, GitLab).

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Programming Techniques)

*   **Quản lý Race Condition bằng Khóa PR (PR-level Locking):** Digger triển khai cơ chế khóa riêng trên Database (`backend/locking`) để ngăn chặn việc hai PR cùng tác động lên một tài nguyên hạ tầng cùng lúc, ngay cả khi Terraform state lock chưa được kích hoạt.
*   **Xử lý Webhook linh hoạt với Go-Retry:** Trong file `agent-tasks/webhook-based-repo-management.md`, ta thấy việc sử dụng `github.com/sethvargo/go-retry`. Đây là kỹ thuật lập trình phòng thủ để xử lý trường hợp Webhook đến trước khi quá trình cài đặt (callback) kịp hoàn tất trong DB.
*   **Sử dụng Go Workspaces:** Dự án sử dụng `go.work` để quản lý monorepo với hàng chục module con (`cli`, `backend`, `libs`, `taco`,...). Điều này cho phép phát triển đồng thời nhiều thành phần mà không gặp lỗi phụ thuộc vòng.
*   **Template-driven CLI:** CLI của Digger không chỉ chạy lệnh mà còn có khả năng tạo mã Terraform dựa trên các template (`sandbox-sidecar/templates`), giúp tự động hóa việc khởi tạo môi trường.

### 4. Luồng Hoạt động Hệ thống (System Operation Flows)

#### A. Luồng Tự động hóa PR (PR Automation Flow):
1.  **Trigger:** Người dùng comment `digger plan` trên GitHub PR.
2.  **Orchestrator:** Nhận Webhook -> Xác thực chữ ký -> Kiểm tra chính sách OPA -> Xác định các project bị ảnh hưởng (Impacted Projects).
3.  **Locking:** Orchestrator tạo một "lock" trong DB cho các project đó để đảm bảo tính tuần tự.
4.  **Dispatch:** Orchestrator gọi GitHub API để kích hoạt một workflow GitHub Action cụ thể.
5.  **Execution:** CLI chạy trong GitHub Action nhận "Spec" từ Orchestrator, thực hiện lệnh Terraform và đẩy kết quả comment ngược lại PR.

#### B. Luồng Phát hiện Sai lệch (Drift Detection Flow):
1.  **Scheduler:** Một dịch vụ chạy ngầm (`Dockerfile_drift`) hoặc Cron job trong DB kích hoạt định kỳ.
2.  **Scan:** Hệ thống quét toàn bộ các project hạ tầng đã đăng ký.
3.  **Alert:** Nếu phát hiện sự khác biệt giữa thực tế (Cloud) và mã nguồn (Git), Digger sẽ gửi thông báo qua Slack hoặc tạo một GitHub Issue mới.

### 5. Tổng kết
Digger/OpenTaco đại diện cho thế hệ TACOs (Terraform Automation and Collaboration Software) mới: **Tập trung vào bảo mật và tiết kiệm chi phí bằng cách không chiếm quyền sở hữu compute**. 

Điểm đặc sắc nhất là cách họ đóng gói logic phức tạp của việc quản lý trạng thái hạ tầng vào một backend siêu nhẹ, trong khi vẫn duy trì sự linh hoạt tối đa cho lập trình viên thông qua CLI mạnh mẽ. Sự dịch chuyển từ Digger sang OpenTaco cũng cho thấy tham vọng trở thành một hệ điều hành thực thi hạ tầng toàn diện hơn là chỉ một công cụ hỗ trợ PR.