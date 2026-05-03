Dựa trên cấu trúc mã nguồn của dự án **Azimutt**, đây là phân tích chi tiết về các khía cạnh công nghệ và kiến trúc của hệ thống:

### 1. Công nghệ cốt lõi (Core Stack)
Azimutt là một ứng dụng Full-stack hiện đại, sử dụng sự kết hợp độc đáo giữa các ngôn ngữ có tính an toàn hệ thống cao:

*   **Backend:** **Elixir** & **Phoenix Framework**. Elixir (chạy trên máy ảo Erlang - BEAM) cung cấp khả năng xử lý song song và chịu lỗi cực tốt, phù hợp cho việc quản lý nhiều phiên làm việc của người dùng.
*   **Database chính:** **PostgreSQL** (thông qua Ecto) để lưu trữ thông tin người dùng, tổ chức, metadata của dự án và sự kiện tracking.
*   **Frontend (Editor):** **Elm**. Đây là lựa chọn công nghệ quan trọng nhất. Elm là ngôn ngữ hàm thuần túy (purely functional) biên dịch sang JS, giúp loại bỏ hoàn toàn lỗi runtime. Điều này cực kỳ quan trọng đối với một ứng dụng đồ họa phức tạp như trình vẽ sơ đồ thực thể (ERD).
*   **Frontend (Glue/Interop):** **TypeScript**. Được dùng làm lớp đệm (Ports) để kết nối Elm với các API của trình duyệt (Clipboard, LocalStorage, Logging).
*   **Styling:** **Tailwind CSS**.
*   **Infrastructure/Services:** 
    *   **S3 (Waffle/ExAws):** Lưu trữ các file JSON dung lượng lớn của đồ án.
    *   **Stripe:** Xử lý thanh toán.
    *   **Mailgun:** Gửi email thông báo.
    *   **Sentry:** Giám sát lỗi.

### 2. Tư duy Kiến trúc (Architectural Mindset)
Dự án được tổ chức theo mô hình **Monorepo** (quản lý bằng `pnpm` và `workspace`), chia nhỏ hệ thống thành các module chức năng:

*   **`backend/`**: Một ứng dụng Phoenix truyền thống (MVC), đóng vai trò API Server, quản lý xác thực (Auth), tổ chức (Organizations), và các dịch vụ nền (Services).
*   **`frontend/`**: Chứa lõi của Azimutt (Editor), được viết bằng Elm để quản lý trạng thái sơ đồ (state) một cách chặt chẽ.
*   **`libs/`**: Các thư viện TypeScript dùng chung để parse các định dạng schema khác nhau (`aml`, `parser-sql`, `parser-dbml`, `parser-prisma`). Cách tiếp cận này giúp Azimutt có khả năng mở rộng (Extensibility) cao.
*   **`gateway/`**: Một server Node.js riêng biệt đóng vai trò proxy, cho phép ứng dụng web kết nối an toàn tới các cơ sở dữ liệu cục bộ hoặc trong mạng nội bộ của người dùng.
*   **`cli/`**: Công cụ dòng lệnh hỗ trợ developer tự động hóa việc trích xuất schema.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **AML (Azimutt Markup Language):** Dự án tự định nghĩa một DSL (Domain Specific Language) tối giản để người dùng thiết kế database bằng code nhanh hơn kéo thả. Việc xây dựng Lexer/Parser cho AML là một kỹ thuật lõi.
*   **Functional Programming:** Tận dụng tối đa lập trình hàm ở cả Elixir và Elm. Điều này giúp code dễ dự đoán, dễ kiểm thử và xử lý các cấu trúc dữ liệu đồ thị (Graph) của database schema một cách hiệu quả.
*   **Pattern Matching (Elixir):** Sử dụng triệt để trong backend để xử lý các loại kết nối database và các sự kiện webhook từ Stripe.
*   **JS/Elm Ports:** Kỹ thuật giao tiếp bất đồng bộ giữa Elm (logic lõi) và TypeScript (các tính năng platform) để đảm bảo an toàn dữ liệu mà vẫn linh hoạt.
*   **Ecto Multi/Changesets:** Đảm bảo tính toàn vẹn dữ liệu khi thực hiện các thao tác phức tạp như mời thành viên vào tổ chức hoặc tạo dự án mới.

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Nhập dữ liệu (Ingestion):** Người dùng cung cấp SQL DDL hoặc file AML.
    *   Các thư viện trong `libs/` sẽ thực hiện parse dữ liệu này thành một mô hình JSON chuẩn của Azimutt.
2.  **Khởi tạo Editor:** 
    *   Backend (Elixir) tải thông tin dự án từ PostgreSQL hoặc S3.
    *   Dữ liệu được đẩy xuống Frontend. Elm nhận dữ liệu và khởi tạo trạng thái đồ họa.
3.  **Xử lý sơ đồ:** 
    *   Mọi tương tác (di chuyển bảng, ẩn hiện cột, tạo quan hệ) đều đi qua vòng lặp **Model-Update-View** của Elm.
    *   Trạng thái sơ đồ được lưu tạm thời vào LocalStorage qua TypeScript Ports để tránh mất dữ liệu.
4.  **Kết nối trực tiếp (Live Data):** 
    *   Nếu người dùng muốn xem dữ liệu thực tế, Browser gửi yêu cầu qua `gateway`.
    *   `gateway` kết nối tới DB đích, trích xuất dữ liệu mẫu và trả về trình duyệt để hiển thị ngay trên sơ đồ ERD.
5.  **Persistence:** 
    *   Khi người dùng nhấn "Save", trạng thái JSON mới nhất được gửi về Elixir API.
    *   Backend kiểm tra quyền (RBAC) và giới hạn gói cước (Plan limits) trước khi ghi đè vào S3 và cập nhật metadata vào Postgres.

### Tổng kết
Azimutt là một hệ thống tinh vi kết hợp giữa **sự ổn định của Elixir** và **sự an toàn tuyệt đối của Elm**. Nó không chỉ là một công cụ vẽ hình, mà là một nền tảng xử lý ngôn ngữ (SQL/AML) và quản lý trạng thái dữ liệu quy mô lớn.