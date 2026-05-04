Dưới đây là phân tích chi tiết về hệ thống **Open Wearables** dựa trên cấu trúc mã nguồn và tài liệu kỹ thuật của dự án:

### 1. Công nghệ cốt lõi (Core Technology)

Hệ thống được xây dựng trên một ngăn xếp công nghệ hiện đại, ưu tiên hiệu suất và khả năng mở rộng:

*   **Backend:** 
    *   **Python 3.13+ & FastAPI:** Lựa chọn Python mới nhất kết hợp với framework web hiệu năng cao để xử lý các yêu cầu API bất đồng bộ.
    *   **SQLAlchemy 2.0 & PostgreSQL:** Sử dụng mô hình ORM tiên tiến nhất của SQLAlchemy để quản lý dữ liệu quan hệ phức tạp.
    *   **Celery & Redis:** Hệ thống hàng đợi tác vụ (Task Queue) mạnh mẽ để xử lý việc đồng bộ dữ liệu nặng từ các nhà cung cấp (Garmin, Oura,...) ở chế độ nền.
*   **Frontend:**
    *   **React 19 & TypeScript:** Sử dụng phiên bản React mới nhất với kiểu dữ liệu chặt chẽ.
    *   **TanStack (Router & Query):** Quản lý luồng điều hướng và trạng thái dữ liệu (caching, fetching) một cách chuyên nghiệp.
    *   **Tailwind CSS & shadcn/ui:** Đảm bảo giao diện hiện đại, dễ tùy chỉnh.
*   **Hạ tầng & AI:**
    *   **MCP (Model Context Protocol):** Sử dụng FastMCP để cho phép các mô hình AI (như Claude, GPT) có thể truy cập và phân tích dữ liệu sức khỏe của người dùng một cách an toàn.
    *   **Svix:** Quản lý việc gửi Webhook ra bên ngoài một cách tin cậy.
    *   **Docker & Docker Compose:** Đóng gói toàn bộ hệ thống để dễ dàng tự triển khai (Self-hosting).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Open Wearables thể hiện tư duy **"Normalization & Decoupling"** (Chuẩn hóa và Tách biệt):

*   **Kiến trúc phân lớp (Layered Architecture):** Mã nguồn được chia rõ rệt thành:
    *   *Routes:* Tiếp nhận yêu cầu.
    *   *Services:* Xử lý logic nghiệp vụ.
    *   *Repositories:* Thao tác trực tiếp với cơ sở dữ liệu.
    *   *Models/Schemas:* Định nghĩa cấu trúc dữ liệu và xác thực (Validation).
*   **Chiến lược Nhà cung cấp (Provider Strategy Pattern):** Đây là điểm cốt lõi. Mỗi nhà cung cấp (Garmin, Apple Health, Whoop) có một "Strategy" riêng để xử lý đặc thù của API đó, nhưng cuối cùng đều trả về một mô hình dữ liệu chuẩn hóa (Unified Data Model). Điều này giúp việc thêm nhà cung cấp mới không làm ảnh hưởng đến logic chung của hệ thống.
*   **Ưu tiên Xử lý Bất đồng bộ:** Dữ liệu sức khỏe thường rất lớn và tốn thời gian để fetch/normalize. Hệ thống đẩy toàn bộ việc này cho Celery Workers, giúp API luôn phản hồi nhanh (Status 202 Accepted).
*   **Self-hosting & Privacy:** Kiến trúc được thiết kế để chạy độc lập (single organization), không phức tạp hóa bằng mô hình multi-tenancy (đa khách hàng), giúp đảm bảo quyền riêng tư tối đa.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Dependency Injection (DI):** Tận dụng tối đa hệ thống DI của FastAPI để quản lý các phiên làm việc của database (`DbSession`) và xác thực API Key.
*   **Mã hóa dữ liệu nhạy cảm:** Sử dụng **Fernet (Cryptography)** để mã hóa các Access Token và Client Secret của người dùng trước khi lưu vào DB.
*   **Xác thực và phân quyền (RBAC):** Tách biệt rõ ràng giữa tài khoản Developer (quản lý dashboard) và User (người dùng thiết bị đeo).
*   **Pydantic V2:** Sử dụng chặt chẽ các Schema để xác thực dữ liệu đầu vào và định dạng dữ liệu đầu ra, đảm bảo tính nhất quán của API.
*   **Mixin Pattern:** Sử dụng các Mixin trong lớp Service để tái sử dụng logic (ví dụ: logic tính toán điểm số sức khỏe - Health Scores).

### 4. Luồng hoạt động hệ thống (System Operation Flow)

Luồng dữ liệu trong hệ thống diễn ra theo các bước sau:

1.  **Kết nối (Auth Flow):**
    *   Người dùng (User) thông qua liên kết từ App/Dashboard thực hiện OAuth với nhà cung cấp (ví dụ: Strava).
    *   Backend nhận Callback, trao đổi Code lấy Token, mã hóa và lưu vào DB.
2.  **Đồng bộ dữ liệu (Sync Flow):**
    *   **Cách 1 (Pull):** Celery Beat kích hoạt các tác vụ định kỳ -> Worker gọi API nhà cung cấp -> Lấy dữ liệu thô -> Chuẩn hóa -> Lưu vào bảng `data_point_series`.
    *   **Cách 2 (Push):** Webhook từ nhà cung cấp gửi đến -> Backend đẩy vào hàng đợi -> Worker xử lý tương tự như trên.
    *   **Cách 3 (SDK):** Mobile App đẩy dữ liệu trực tiếp qua endpoint `/sdk/sync`.
3.  **Tính toán điểm số (Scoring Flow):**
    *   Sau khi dữ liệu thô được lưu, các tác vụ tính toán (Sleep Score, Resilience Score) sẽ chạy dựa trên các thuật toán tại `app/algorithms/` để tạo ra các thông tin chi tiết (insights).
4.  **Truy xuất & Thông báo (Consumption Flow):**
    *   Ứng dụng bên thứ ba truy vấn API chuẩn hóa để lấy dữ liệu.
    *   Nếu có sự kiện quan trọng, hệ thống thông qua **Svix** sẽ bắn Webhook ra các URL đã cấu hình trước đó.

### Tổng kết
Open Wearables là một hệ thống **Intermediary (Trung gian)** được thiết kế rất bài bản. Nó giúp lập trình viên không cần quan tâm đến sự khác biệt giữa các API của từng hãng thiết bị đeo, mà chỉ cần làm việc với một giao diện API duy nhất, sạch sẽ và sẵn sàng cho các ứng dụng AI.