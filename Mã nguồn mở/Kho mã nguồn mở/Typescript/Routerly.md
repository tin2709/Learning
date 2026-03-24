Dưới đây là phân tích chi tiết về dự án **Routerly** dựa trên cấu trúc mã nguồn và tài liệu đi kèm:

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án được xây dựng theo mô hình **Monorepo** sử dụng **npm workspaces**, tập trung vào hiệu suất và khả năng tự triển khai (self-hosted).

*   **Ngôn ngữ & Runtime:** 
    *   **TypeScript (chủ đạo):** Đảm bảo an toàn về kiểu dữ liệu trên toàn bộ hệ thống (từ backend đến frontend và CLI).
    *   **Node.js 20+:** Sử dụng các tính năng mới nhất của Node.js như ESM (EcmaScript Modules).
*   **Backend (Service):**
    *   **Framework:** Nhiều khả năng dựa trên Fastify hoặc Express (trong thư mục `packages/service`).
    *   **Lưu trữ (Infrastructure-less):** Điểm đặc biệt nhất là **không dùng Database bên ngoài** (No Postgres/Redis). Hệ thống sử dụng tệp **JSON cục bộ** để lưu cấu hình và dữ liệu sử dụng (`usage.json`).
    *   **Bảo mật:** Sử dụng **AES-256** để mã hóa API Key của nhà cung cấp và **JWT** cho phiên làm việc của Dashboard.
*   **Frontend (Dashboard):**
    *   **React + Vite:** Xây dựng giao diện quản trị hiện đại, tốc độ cao.
    *   **Tailwind CSS:** Quản lý giao diện linh hoạt.
    *   **Recharts:** Xử lý biểu đồ theo dõi chi phí và lưu lượng theo thời gian thực.
*   **CLI (Admin Tool):**
    *   Sử dụng **Commander.js** và **Inquirer.js** để tạo giao diện dòng lệnh tương tác mạnh mẽ.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Routerly xoay quanh khái niệm **"Intelligent Gateway"** (Cổng kết nối thông minh):

*   **Kiến trúc Gateway/Proxy:** Routerly nằm giữa ứng dụng khách (Client) và các nhà cung cấp LLM (OpenAI, Anthropic...). Nó đóng vai trò là một điểm cuối (endpoint) duy nhất nhưng có khả năng điều hướng đa luồng.
*   **Tư duy Pluggable Policies (Chính sách cắm nóng):** Hệ thống định tuyến không cố định mà dựa trên một "ngăn xếp" (stack) các chính sách (policies). Mỗi yêu cầu được chấm điểm qua nhiều bộ lọc (Cost, Health, Performance, Capability) trước khi chọn model cuối cùng.
*   **Multi-tenant Isolation (Cô lập đa người dùng):** Sử dụng cấu trúc "Project". Mỗi project có Token riêng, ngân sách riêng và tập hợp model riêng, cho phép chia sẻ một instance Routerly cho nhiều đội nhóm/khách hàng mà không bị lẫn lộn dữ liệu.
*   **Zero-Ops / Zero-Infrastructure:** Tối giản hóa việc triển khai. Người dùng chỉ cần một file thực thi hoặc Docker image mà không cần cài đặt thêm các dịch vụ lưu trữ phức tạp.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Parallel Scoring (Chấm điểm song song):** Khi một request đến, hệ thống chạy đồng thời 9 loại routing policies để đánh giá các model ứng viên, sau đó tổng hợp điểm số (weighted scoring) để đưa ra quyết định nhanh nhất.
*   **Wire-Compatible Proxying:** Kỹ thuật xử lý stream và chuyển tiếp giao thức. Routerly có khả năng đọc và giả lập chính xác định dạng của OpenAI và Anthropic, cho phép người dùng thay đổi nhà cung cấp mà không cần sửa code ứng dụng (chỉ đổi URL).
*   **LLM-powered Routing (Định tuyến bằng AI):** Sử dụng một model LLM nhỏ (như GPT-4o-mini hoặc Ollama) để phân tích ý định của người dùng trong request hiện tại, từ đó quyết định xem có nên dùng một model mạnh (đắt) hay một model yếu (rẻ) để xử lý.
*   **Token Level Pricing:** Kỹ thuật tính toán chi phí chính xác theo thời gian thực dựa trên số lượng token input/output và bảng giá (pricing tiers) được cấu hình linh hoạt.

---

### 4. Luồng hoạt động hệ thống (System Operation Flow)

Quy trình xử lý một yêu cầu API diễn ra như sau:

1.  **Xác thực (Auth Phase):** 
    *   Routerly nhận request qua header `Authorization: Bearer sk-lr-...`.
    *   Hệ thống kiểm tra Token thuộc Project nào và kiểm tra hạn mức ngân sách (Budget check).
2.  **Định tuyến (Routing Phase):**
    *   Xác định danh sách các model ứng viên cho Project đó.
    *   Áp dụng các chính sách: Loại bỏ model đang lỗi (Health), loại bỏ model không đủ context (Context), ưu tiên model rẻ nhất (Cheapest).
    *   Nếu enable `llm policy`, một call phụ sẽ được thực hiện để AI tư vấn model tốt nhất.
3.  **Thực thi (Execution Phase):**
    *   Chuyển tiếp (Forward) request đến Provider thực tế (OpenAI/Anthropic).
    *   Hỗ trợ Streaming: Dữ liệu trả về được stream ngược lại cho Client ngay lập tức.
4.  **Hậu xử lý (Tracking Phase):**
    *   Sau khi request kết thúc, hệ thống lấy thông tin `usage` (số token đã dùng).
    *   Tính toán chi phí dựa trên metadata của model.
    *   Ghi bản ghi vào `usage.json` và cập nhật ngân sách còn lại của Project.

### Tổng kết
Routerly là một giải pháp **Middleware** thông minh dành cho kỷ nguyên AI. Nó kết hợp tính thực dụng (không DB, dễ cài đặt) với các kỹ thuật quản lý tài nguyên nâng cao (dynamic routing, budget enforcement), giúp doanh nghiệp kiểm soát chặt chẽ chi phí và tăng độ tin cậy khi sử dụng các dịch vụ AI.