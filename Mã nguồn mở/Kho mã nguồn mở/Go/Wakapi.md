Dựa trên tệp tin mã nguồn của dự án **Wakapi** (phiên bản cập nhật đến tháng 3/2026), dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống này:

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án được xây dựng với tư duy tối giản nhưng hiệu quả cao, tập trung vào hiệu suất (Go) và khả năng tự vận hành (Self-hosted).

*   **Ngôn ngữ lập trình:** **Go (Golang)** - tận dụng khả năng thực thi nhanh, tiêu tốn ít tài nguyên và hỗ trợ concurrency (goroutines) mạnh mẽ.
*   **Web Framework:** **Chi (go-chi/chi/v5)** - một router nhẹ, tương thích tốt với thư viện chuẩn `net/http` của Go, giúp xây dựng các Middleware linh hoạt.
*   **ORM (Object-Relational Mapping):** **GORM** - hỗ trợ đa cơ sở dữ liệu (SQLite, MySQL, Postgres) giúp người dùng dễ dàng chuyển đổi môi trường lưu trữ.
*   **Cơ sở dữ liệu:** Mặc định dùng **SQLite** (cho sự đơn giản), nhưng hỗ trợ tốt **PostgreSQL** và **MySQL/MariaDB** cho các hệ thống lớn.
*   **Xác thực (Auth):** 
    *   **OAuth2/OIDC:** Đăng nhập qua các nhà cung cấp bên thứ ba (Google, GitHub...).
    *   **WebAuthn:** Hỗ trợ bảo mật sinh trắc học hoặc khóa vật lý (Passkeys).
    *   **API Key:** Dùng cho các client (IDE plugins) để gửi dữ liệu.
*   **Frontend:** 
    *   **Go Templates:** Server-side rendering để tối ưu tốc độ tải trang.
    *   **Tailwind CSS:** Xử lý giao diện hiện đại.
    *   **Petite-vue:** Một phiên bản siêu nhẹ của Vue.js để xử lý logic tương tác phía client mà không cần bundle nặng nề.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Wakapi áp dụng kiến trúc **Layered Architecture (Kiến trúc phân tầng)** kết hợp với **Service-oriented**, giúp tách biệt rõ ràng các trách nhiệm:

*   **Models:** Định nghĩa cấu trúc dữ liệu và các quy tắc nghiệp vụ cơ bản.
*   **Repositories:** Tầng giao tiếp trực tiếp với Database. Đây là nơi chứa các câu lệnh SQL hoặc GORM.
*   **Services:** Tầng chứa logic nghiệp vụ (Business Logic). Services sẽ gọi Repositories để lấy dữ liệu và xử lý chúng trước khi trả về cho Controller.
*   **Routes/Handlers:** Tầng tiếp nhận yêu cầu HTTP, kiểm tra đầu vào và gọi các Services tương ứng.
*   **Event-Driven (Tư duy sự kiện):** Sử dụng một Internal Event Bus (`leandro-lugaresi/hub`). Khi có một sự kiện xảy ra (ví dụ: tạo Heartbeat mới), hệ thống sẽ phát tín hiệu để các service khác (như Aggregation) xử lý mà không làm nghẽn luồng chính.
*   **Tính tương thích (Compatibility):** Wakapi được thiết kế như một "Drop-in replacement" cho WakaTime. Nó triển khai các API endpoint giống hệt WakaTime để các plugin trên VS Code, JetBrains có thể hoạt động mà không cần sửa code plugin.

---

### 3. Kỹ thuật lập trình nổi bật (Programming Techniques)

*   **Middleware Chaining:** Hệ thống sử dụng rất nhiều middleware để xử lý các tác vụ xuyên suốt như:
    *   `SharedDataMiddleware`: Giải quyết vấn đề truyền dữ liệu giữa các tầng middleware trong Chi context.
    *   `AuthenticateMiddleware`: Hỗ trợ đa phương thức xác thực (Cookie, API Key, Trusted Header).
    *   `WakatimeRelayMiddleware`: Một kỹ thuật thú vị giúp chuyển tiếp (proxy) dữ liệu từ Wakapi sang server WakaTime chính thức nếu người dùng muốn dùng cả hai.
*   **Database Migrations:** Tự xây dựng hệ thống migration mạnh mẽ (`migrations/`). Hệ thống kiểm tra lịch sử migration qua bảng `key_string_values` để đảm bảo schema luôn được cập nhật tự động khi người dùng nâng cấp phiên bản Docker.
*   **Background Jobs:** Sử dụng `artifex` (một thư viện worker pool của cùng tác giả) để xử lý các tác vụ nặng như:
    *   Tổng hợp dữ liệu (Aggregation) hàng đêm.
    *   Gửi email báo cáo hàng tuần.
    *   Dọn dẹp dữ liệu cũ (Housekeeping).
*   **Static Asset Embedding:** Sử dụng tính năng `embed` của Go để đóng gói toàn bộ CSS, JS, HTML vào trong một tệp thực thi (binary) duy nhất. Điều này cực kỳ hữu ích cho việc triển khai (deployment) - chỉ cần một file duy nhất là chạy được toàn bộ web server.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Hệ thống hoạt động qua 2 luồng chính:

#### Luồng ghi dữ liệu (Ingestion Flow):
1.  **Client (IDE Plugin)** gửi một "Heartbeat" (tín hiệu code) qua HTTP POST kèm API Key.
2.  **Middleware** xác thực API Key và kiểm tra quyền hạn.
3.  **Relay Middleware** (nếu bật) sẽ bí mật gửi một bản sao heartbeat này sang server WakaTime.com.
4.  **Heartbeat Service** tiếp nhận, chuẩn hóa dữ liệu (nhận diện ngôn ngữ lập trình, dự án).
5.  **Event Bus** phát tin hiệu "Heartbeat Created".
6.  Dữ liệu được lưu vào bảng `heartbeats`.

#### Luồng xử lý và hiển thị (Aggregation & UI Flow):
1.  **Aggregation Service** (chạy ngầm theo cron-job) quét bảng `heartbeats` để tính toán tổng thời gian code theo ngày, dự án, ngôn ngữ.
2.  Kết quả được lưu vào bảng `summaries` (tầng cache dữ liệu đã xử lý).
3.  Khi **Người dùng** truy cập Dashboard, hệ thống sẽ lấy dữ liệu từ bảng `summaries` thay vì quét hàng triệu dòng trong bảng `heartbeats`, giúp tốc độ hiển thị cực nhanh (Lightning fast).
4.  **Badges Service** tạo ra các tệp hình ảnh SVG động để người dùng nhúng vào GitHub Readme.

---

### Kết luận
Dự án **muety/wakapi** là một ví dụ điển hình về việc xây dựng sản phẩm **"Lean & Mean"**. Nó không sử dụng những framework quá đồ sộ mà tập trung vào việc tối ưu hóa cấu trúc dữ liệu và quy trình xử lý ngầm để đạt được hiệu suất tối đa trên phần cứng yếu (như Raspberry Pi hoặc các VPS rẻ tiền). Sự kết hợp giữa Go và SQLite/Postgres làm cho nó trở thành một công cụ tự vận hành (self-hosted) lý tưởng cho lập trình viên.