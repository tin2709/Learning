Dưới đây là phân tích chi tiết về dự án **Exercise Diary** dựa trên mã nguồn bạn đã cung cấp. Đây là một ứng dụng web tự lưu trữ (self-hosted) viết bằng ngôn ngữ Go, được thiết kế để theo dõi quá trình tập luyện với giao diện trực quan hóa dạng HeatMap tương tự GitHub.

---

### 1. Công Nghệ Cốt Lõi (Core Technologies)

*   **Ngôn ngữ lập trình:** **Go (Golang) 1.23+**. Sử dụng ưu điểm về hiệu năng và khả năng biên dịch thành file thực thi duy nhất.
*   **Web Framework:** **Gin Gonic**. Một framework nhẹ, tốc độ cao để xử lý các yêu cầu HTTP và Routing.
*   **Cơ sở dữ liệu:** **SQLite** (thông qua driver `modernc.org/sqlite`). Đây là driver thuần Go (CGO-free), giúp việc build ứng dụng đa nền tảng (cross-platform) cực kỳ dễ dàng mà không cần cài đặt thư viện C.
*   **Quản lý cấu hình:** **Viper**. Cho phép đọc cấu hình từ file YAML, biến môi trường (Environment Variables) và tham số dòng lệnh.
*   **Giao diện (Frontend):**
    *   **Bootstrap 5 (Bootswatch Themes):** Cung cấp giao diện hiện đại, hỗ trợ nhiều theme (Dark/Light).
    *   **Chart.js & chartjs-chart-matrix:** Sử dụng để vẽ biểu đồ cân nặng và biểu đồ nhiệt (HeatMap).
    *   **Go Templates:** Sử dụng engine mặc định của Go để render HTML từ phía server.
*   **Đóng gói & Triển khai:** **Docker & Docker Compose**, hỗ trợ Multi-stage build để tối ưu dung lượng ảnh (sử dụng image `scratch` rất nhỏ gọn).

---

### 2. Tư Duy Kiến Trúc (Architectural Thinking)

Dự án tuân thủ cấu trúc thư mục tiêu chuẩn của Go (`/cmd`, `/internal`), giúp tách biệt rõ ràng giữa logic khởi chạy và logic nghiệp vụ:

*   **Tính đóng gói (Encapsulation):** Toàn bộ logic xử lý nằm trong thư mục `internal/`. Người dùng bên ngoài chỉ tương tác thông qua `cmd/ExerciseDiary/main.go`.
*   **Portable (Khả năng di động):** Sử dụng `embed.FS` để nhúng trực tiếp các file HTML, CSS, JS và Version vào trong file nhị phân duy nhất. Bạn chỉ cần 1 file duy nhất để chạy toàn bộ server web.
*   **Stateless logic vs Persistent storage:** Logic xử lý không lưu trạng thái (Stateless), toàn bộ dữ liệu được đẩy vào SQLite. Điều này giúp ứng dụng dễ dàng sao lưu (chỉ cần copy file `sqlite.db`).
*   **Tư duy thực dụng (Pragmatic approach):** Thay vì sử dụng các framework Frontend phức tạp (React/Vue), tác giả chọn SSR (Server Side Rendering) kết hợp với Vanilla JS. Điều này giảm độ phức tạp và phù hợp với một công cụ quản lý cá nhân.

---

### 3. Các Kỹ Thuật Chính (Key Techniques)

*   **Xử lý số thập phân:** Sử dụng thư viện `shopspring/decimal` cho các trường dữ liệu cân nặng (`Weight`). Điều này cực kỳ quan trọng trong lập trình tài chính hoặc thể hình để tránh sai số dấu phẩy động (floating-point error) của kiểu `float64`.
*   **Xác thực và Bảo mật:**
    *   **Bcrypt:** Mã hóa mật khẩu người dùng trước khi lưu vào cấu hình.
    *   **Session Token (UUID):** Sử dụng Cookie và UUID để duy trì phiên đăng nhập thay vì lưu mật khẩu trực tiếp trong mỗi request.
    *   **Middleware:** Sử dụng Middleware trong Gin để kiểm tra quyền truy cập cho tất cả các route nhạy cảm.
*   **Database Mapping:** Sử dụng thư viện `sqlx` để map trực tiếp kết quả truy vấn SQL vào các Go Structs, giúp code ngắn gọn và dễ bảo trì hơn so với `database/sql` thuần.
*   **Tối ưu Docker Image:** Sử dụng kỹ thuật Multi-stage build. Stage 1 dùng `golang:alpine` để build code, Stage 2 dùng `scratch` (image rỗng) để chạy file thực thi. Kết quả là Docker image có kích thước cực nhỏ.
*   **Hỗ trợ mạng nội bộ (Offline mode):** Kỹ thuật cho phép người dùng trỏ đường dẫn node-bootstrap qua tham số `-n`, giúp ứng dụng chạy hoàn toàn trong mạng LAN mà không cần internet để tải CDN (CSS/JS).

---

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

1.  **Khởi tạo (Initialization):**
    *   Ứng dụng bắt đầu tại `main.go`, đọc các flag dòng lệnh (đường dẫn thư mục data, node path).
    *   Viper load cấu hình từ `config.yaml` hoặc biến môi trường.
    *   Kết nối SQLite và tự động tạo bảng (Auto-migrate) nếu chưa tồn tại qua `db.Create()`.
2.  **Xử lý yêu cầu (Request Handling):**
    *   Khi người dùng truy cập, Gin Router định tuyến đến các Handler tương ứng (ví dụ: `indexHandler`).
    *   Nếu bật `AUTH`, Middleware sẽ kiểm tra Cookie hợp lệ trước khi cho phép vào Handler.
3.  **Luồng dữ liệu (Data Flow):**
    *   **Xem dữ liệu:** Handler gọi hàm `Select` từ package `db` -> Lấy dữ liệu từ SQLite -> Đổ vào struct `GuiData` -> Render qua Go Template -> Trả về HTML cho trình duyệt.
    *   **Ghi dữ liệu:** Người dùng gửi Form (POST) -> Handler nhận dữ liệu -> Chuyển đổi kiểu dữ liệu (String sang Decimal/Int) -> Gọi hàm `Insert` vào SQLite -> Redirect người dùng về trang chủ.
4.  **Trực quan hóa (Visualization):**
    *   Tại phía Client (Trình duyệt), các đoạn script JS (`heatmap.js`, `weight-chart.js`) nhận dữ liệu thô từ Go Template (được nhúng dưới dạng JSON) và gọi thư viện Chart.js để vẽ biểu đồ.

### Tổng kết
Đây là một dự án có kiến trúc **Clean & Simple**, cực kỳ phù hợp cho những ai muốn học cách xây dựng một ứng dụng web hoàn chỉnh bằng Go từ việc quản lý Database, Auth, Cấu hình đến việc tối ưu hóa triển khai bằng Docker.