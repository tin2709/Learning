Dựa trên cấu trúc thư mục và nội dung các file quan trọng của dự án **ToolJet**, dưới đây là phân tích chi tiết về các khía cạnh kỹ thuật của nền tảng low-code này:

### 1. Công nghệ cốt lõi (Core Technology)

ToolJet được xây dựng theo mô hình **Monorepo** (quản lý nhiều gói phần mềm trong một kho mã nguồn duy nhất) với các công nghệ hiện đại:

*   **Backend:** Sử dụng **Node.js** với framework **NestJS** (TypeScript). Đây là lựa chọn tối ưu cho tính module hóa cao và khả năng mở rộng. Hệ thống sử dụng **TypeORM** để quản lý cơ sở dữ liệu.
*   **Frontend:** Sử dụng **React** kết hợp với **Tailwind CSS** và **SCSS**. Trình biên dịch mã là **Webpack**.
*   **Cơ sở dữ liệu:**
    *   **PostgreSQL:** Lưu trữ dữ liệu hệ thống (users, apps, definitions).
    *   **Redis:** Dùng cho caching và xử lý các tác vụ thời gian thực (Multiplayer editing).
    *   **ToolJet DB:** Một giải pháp no-code DB tích hợp dựa trên **PostgREST**, cho phép người dùng tạo bảng dữ liệu mà không cần kiến thức về SQL.
*   **Khả năng mở rộng (Extensibility):** Hệ thống Plugin mạnh mẽ cho phép tích hợp hơn 80 nguồn dữ liệu (SaaS, DBs, APIs). Sử dụng **oclif** để xây dựng ToolJet CLI giúp lập trình viên tự tạo plugin.
*   **Thực thi mã:** Tích hợp **Pyodide** để chạy Python trực tiếp trên trình duyệt và các sandbox an toàn để thực thi JavaScript.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của ToolJet tập trung vào tính **linh hoạt** và **độ tin cậy của dữ liệu**:

*   **Metadata-Driven UI:** Toàn bộ ứng dụng low-code không phải là mã nguồn cứng mà là một cấu trúc JSON (định nghĩa trong `definitions`). Frontend engine sẽ đọc JSON này để render các thành phần (widgets) và thiết lập luồng dữ liệu tương ứng.
*   **Kiến trúc Plugin (Decoupling):** Logic kết nối với các nguồn dữ liệu bên ngoài (như Airtable, S3, MongoDB) được tách rời hoàn toàn khỏi lõi của server. Điều này giúp việc thêm mới nguồn dữ liệu không làm ảnh hưởng đến tính ổn định của hệ thống chính.
*   **Proxy-only Data Flow:** Để đảm bảo bảo mật, mọi truy vấn từ trình duyệt người dùng đến nguồn dữ liệu (DB của khách hàng) đều đi qua một Proxy Server (lõi ToolJet). Thông tin nhạy cảm (API Keys, Passwords) được mã hóa bằng **AES-256-GCM** và không bao giờ lộ ra phía client.
*   **Real-time Collaboration:** Kiến trúc hỗ trợ chỉnh sửa đồng thời (Multiplayer) dựa trên WebSockets và các giải pháp giải quyết xung đột dữ liệu (như Yjs, được gợi ý qua file `yjs.gateway.ts`).

### 3. Kỹ thuật lập trình chính (Primary Programming Techniques)

*   **Migration Layer kép:** Dự án duy trì hai loại migration: 
    *   *Schema Migrations:* Thay đổi cấu trúc bảng trong DB.
    *   *Data Migrations:* Cập nhật cấu trúc định nghĩa ứng dụng JSON (ví dụ: khi một widget cũ được nâng cấp thuộc tính mới).
*   **Dependency Tracking:** Kỹ thuật theo dõi sự phụ thuộc giữa các thành phần. Nếu Component A sử dụng dữ liệu từ Query B, hệ thống tự động tính toán để cập nhật Component A ngay khi Query B có kết quả mới.
*   **Inversion of Control (IoC):** Tận dụng tối đa Dependency Injection của NestJS để quản lý các dịch vụ (services) và repositories, giúp mã nguồn dễ kiểm thử (unit test) và bảo trì.
*   **Hệ thống xử lý lỗi tập trung:** Sử dụng các Filters trong NestJS (như `all-exceptions-filter.ts`) để bắt và chuẩn hóa lỗi trả về cho client, đảm bảo trải nghiệm người dùng không bị gián đoạn.

### 4. Luồng hoạt động hệ thống (System Operational Flow)

Luồng hoạt động có thể chia làm hai giai đoạn chính:

#### A. Giai đoạn Thiết kế (Design Time)
1.  Người dùng sử dụng **App Builder** (Canvas) kéo thả widget.
2.  Mọi thao tác thay đổi thuộc tính hoặc tạo truy vấn sẽ cập nhật vào **Editor Store** (phía client).
3.  Hệ thống gửi các thay đổi (diff) về Server thông qua Socket hoặc API để lưu trữ vào PostgreSQL dưới dạng định nghĩa JSON (App Version).

#### B. Giai đoạn Thực thi (Runtime)
1.  Khi người dùng cuối mở App, Frontend tải về định nghĩa JSON của phiên bản đã release.
2.  **Execution Engine** bắt đầu khởi tạo trạng thái ban đầu, giải mã các biến môi trường và hằng số.
3.  **Luồng dữ liệu:**
    *   Người dùng kích hoạt sự kiện (như nhấn nút).
    *   Frontend gửi yêu cầu thực thi truy vấn lên Server.
    *   Server tìm đúng Plugin tương ứng, nạp thông tin xác thực đã mã hóa, thực thi truy vấn đến nguồn dữ liệu đích (ví dụ: SQL Server).
    *   Kết quả trả về Server -> Server xử lý (Transformations) -> Trả về Client.
    *   Client cập nhật trạng thái toàn cục và render lại các widget liên quan.

### Tổng kết
ToolJet là một hệ thống low-code có kiến trúc rất chặt chẽ, chú trọng vào việc tách biệt giữa logic hiển thị và logic xử lý dữ liệu. Việc sử dụng Monorepo giúp đội ngũ phát triển đồng bộ nhanh chóng giữa CLI, Server, Frontend và hệ thống Plugin dày đặc.