Dựa trên mã nguồn và cấu trúc thư mục của dự án **AstronRPA** (phát triển bởi iFlytek), dưới đây là phân tích chi tiết về Công nghệ cốt lõi, Tư duy kiến trúc, Kỹ thuật lập trình và Luồng hoạt động của hệ thống:

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án là một hệ thống **Hybrid (Đa ngôn ngữ)** kết hợp sức mạnh của Java và Python để xử lý các bài toán khác nhau:

*   **Backend Management (Java/Spring Boot):** Sử dụng cho các dịch vụ cần sự ổn định cao và quản lý doanh nghiệp (Enterprise Management).
    *   `robot-service`: Quản lý danh sách robot, lập lịch (scheduling), quản lý task.
    *   `rpa-auth`: Quản lý định danh (IAM), tích hợp Casdoor (Auth as a Service) và UAP.
*   **AI & Gateway Services (Python/FastAPI):** Tận dụng hệ sinh thái AI mạnh mẽ của Python.
    *   `ai-service`: Tích hợp LLM (DeepSeek, iFlytek Spark), OCR (nhận diện chữ viết), CAPTCHA solver.
    *   `openapi-service`: Cung cấp API cho bên thứ ba và hỗ trợ giao thức **MCP (Model Context Protocol)** - một tiêu chuẩn mới cho việc kết nối AI Agent với các công cụ bên ngoài.
*   **Execution Engine (Python 3.13):** Lõi thực thi RPA nằm ở phía Client.
    *   Sử dụng các thư viện như `playwright`/`selenium` (Web), `pywin32`/`uiautomation` (Desktop GUI).
    *   **Computer Use Agent:** Tích hợp các mô hình AI có khả năng tự quan sát màn hình và điều khiển chuột/phím như người thật.
*   **Frontend (Vue 3 + TypeScript + Electron):**
    *   `web-app`: Giao diện thiết kế quy trình (Designer) kéo thả.
    *   `electron-app`: Ứng dụng desktop để chạy robot và tương tác với hệ điều hành Windows.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của AstronRPA đi theo mô hình **Cloud-Edge-Client**:

1.  **Kiến trúc hướng Microservices:** Hệ thống chia nhỏ các chức năng thành các service độc lập (`ai`, `openapi`, `robot`, `resource`). Điều này cho phép mở rộng (scaling) riêng lẻ, ví dụ: cụm AI service có thể chạy trên GPU server trong khi Robot service chạy trên CPU server thông thường.
2.  **Thiết kế Atomic Components (Nguyên tử hóa):** Mọi hành động của Robot (Click, Type, Open Browser) được đóng gói thành các `atomic-action` (trong thư mục `engine/components`). Tư duy này giúp người dùng không cần biết code vẫn có thể lắp ghép quy trình (Low-code/No-code).
3.  **Hệ thống tích điểm (Points System):** Trong `ai-service`, kiến trúc tích hợp cơ chế quản lý points (`PointAllocation`, `PointConsumption`). Đây là tư duy thiết kế dành cho mô hình SaaS hoặc môi trường doanh nghiệp để kiểm soát chi phí sử dụng tài nguyên AI/OCR đắt đỏ.
4.  **Tích hợp Native AI Agent:** Khác với RPA truyền thống chạy theo kịch bản cứng (if-else), AstronRPA coi AI là "bộ não". Nó hỗ trợ gọi ngược (callback) giữa RPA quy trình và AI Agent, cho phép AI tự ra quyết định dựa trên dữ liệu thu thập được từ màn hình.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Asynchronous Programming (Lập trình bất đồng bộ):**
    *   Phía Python (`FastAPI`): Sử dụng `async/await` triệt để cho các thao tác IO-bound (gọi API LLM, truy vấn Database).
    *   Phía Java: Sử dụng `Feign Client` để giao tiếp giữa các service một cách khai báo.
*   **WebSocket & Real-time Communication:**
    *   Sử dụng WebSocket để đẩy trạng thái thực thi từ Robot (Client) lên Server theo thời gian thực. Kỹ thuật này giúp người quản lý biết chính xác Robot đang dừng ở bước nào.
*   **Middleware & Tracing:**
    *   Sử dụng `contextvars` trong Python để triển khai `RequestID`. Mọi log từ lúc bắt đầu request đến khi kết thúc (qua nhiều service) đều mang cùng một ID, giúp debug cực nhanh.
*   **AOP (Aspect-Oriented Programming):**
    *   Trong Java (`robot-service`), dự án sử dụng Annotation tùy chỉnh như `@ApiLog`, `@RightCheck` để tách biệt logic nghiệp vụ và logic kiểm tra quyền/ghi log.
*   **Plugin Architecture:**
    *   Hệ thống cho phép cài đặt thêm browser-extension để tương tác sâu vào DOM của trình duyệt mà không bị chặn bởi các cơ chế bảo mật thông thường.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Quy trình từ lúc thiết kế đến khi thực thi diễn ra như sau:

1.  **Giai đoạn Thiết kế (Design):**
    *   Người dùng mở ứng dụng Desktop (Electron), sử dụng Designer (Vue 3) để kéo thả các component.
    *   Designer tạo ra một file JSON/YAML mô tả luồng công việc (Workflow Metadata).
    *   Workflow này được lưu trữ tại `resource-service`.

2.  **Giai đoạn Kích hoạt (Trigger):**
    *   Có nhiều cách kích hoạt: Chạy thủ công, đặt lịch (Cron job trong `robot-service`), hoặc gọi qua API (thông qua `openapi-service`).
    *   Nếu gọi qua AI Agent (MCP), Agent sẽ gửi yêu cầu đến Server để tìm Robot phù hợp.

3.  **Giai đoạn Thực thi (Execution):**
    *   Server gửi lệnh thực thi xuống Client thông qua kết nối **WebSocket** bền vững.
    *   **RPA Engine (Python)** tại Client nhận lệnh, tải script và các phụ thuộc cần thiết.
    *   Engine bắt đầu điều khiển thiết bị (chuột, phím, trình duyệt) theo các `atomic-actions`.
    *   Nếu gặp CAPTCHA hoặc cần phân tích dữ liệu phức tạp, Engine sẽ gọi ngược lên `ai-service` để xử lý.

4.  **Giai đoạn Giám sát (Monitoring):**
    *   Trong quá trình chạy, Robot liên tục gửi log, ảnh chụp màn hình (nếu lỗi) về Server.
    *   Dữ liệu thực thi được lưu vào MySQL và hiển thị trên Dashboard cho quản trị viên.

### Tóm lại
**AstronRPA** không chỉ là một công cụ automation đơn thuần mà là một nền tảng **AI-Powered Automation**. Nó kết hợp tư duy quản trị chặt chẽ của Java (Enterprise) với sự linh hoạt, thông minh của Python (AI) để giải quyết bài toán tự động hóa ở quy mô lớn.