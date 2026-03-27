Dựa trên mã nguồn và cấu trúc thư mục của dự án **certctl**, dưới đây là phân tích chi tiết về Công nghệ cốt lõi, Tư duy Kiến trúc, Kỹ thuật lập trình và Luồng hoạt động của hệ thống.

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

*   **Backend:**
    *   **Ngôn ngữ:** Go (Golang) phiên bản 1.25. Đây là lựa chọn tối ưu cho hệ thống hạ tầng nhờ tính chất biên dịch tĩnh, xử lý đồng thời (concurrency) tốt và tiêu tốn ít tài nguyên.
    *   **Web Framework:** Sử dụng thư viện tiêu chuẩn `net/http` kết hợp với kiến trúc Router tự xây dựng (không phụ thuộc vào các framework nặng nề như Gin hay Echo), giúp kiểm soát tối đa hiệu năng.
    *   **Database:** PostgreSQL 16. Sử dụng thư viện `lib/pq` và `golang-migrate` để quản lý schema.
    *   **Logging:** `log/slog` (Structured Logging) - thư viện log hiện đại của Go giúp tích hợp dễ dàng với các hệ thống ELK/Loki.
*   **Frontend:**
    *   **Framework:** React với TypeScript.
    *   **Build Tool:** Vite (tốc độ nhanh hơn Webpack).
    *   **Styling:** Tailwind CSS.
    *   **State Management:** Hooks (không thấy dấu hiệu của Redux, cho thấy tư duy sử dụng React thuần túy để giảm độ phức tạp).
*   **Infrastructure & DevOps:**
    *   **Docker:** Multi-stage build (giảm kích thước image).
    *   **Protocol:** ACME v2 (cho Let's Encrypt), EST (RFC 7030) cho thiết bị IoT/Mobile, và MCP (Model Context Protocol) cho tích hợp AI.

---

### 2. Tư duy Kiến trúc (Architectural Patterns)

Hệ thống được xây dựng theo mô hình **Control Plane - Worker (Agent)**:

*   **Kiến trúc Pull-based:** Thay vì Server chủ động kết nối tới các máy chủ đích (Target) để cài đặt chứng chỉ (gây rủi ro bảo mật và khó đi qua tường lửa), các **Agent** cài tại máy đích sẽ định kỳ "pull" (kéo) lệnh từ Server về.
*   **Layered Architecture (Kiến trúc phân lớp):**
    *   `Handler` (API Layer): Tiếp nhận HTTP request, validate tham số.
    *   `Service` (Business Logic): Chứa logic nghiệp vụ chính (tính toán ngày hết hạn, điều phối job).
    *   `Repository` (Data Access): Abstraction lớp dữ liệu, giúp dễ dàng thay đổi DB nếu cần (hiện tại là Postgres).
*   **Interface-driven Design:**
    *   Dự án định nghĩa các Interface rất chặt chẽ cho `Issuer` (Nơi cấp chứng chỉ), `Target` (Nơi cài đặt), và `Notifier` (Kênh thông báo). Điều này cho phép mở rộng hệ thống (ví dụ: thêm một CA mới) chỉ bằng cách implement một interface mà không sửa logic cốt lõi.
*   **Security First:**
    *   **Key Isolation:** Khóa bí mật (Private Key) được sinh ra tại Agent và không bao giờ rời khỏi Agent. Server chỉ giữ Certificate (Public Key). Đây là tư duy thiết kế bảo mật cấp cao.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Idempotency (Tính bù trừ/Nhất quán):** Các script SQL (`migrations`, `seed`) sử dụng `IF NOT EXISTS` và `ON CONFLICT DO NOTHING`. Điều này đảm bảo khi hệ thống khởi động lại hoặc scale up, dữ liệu không bị hỏng.
*   **Dependency Injection (Tiêm phụ thuộc):** Các Service được khởi tạo và "bơm" vào Handler trong file `main.go`. Kỹ thuật này giúp code dễ dàng viết Unit Test (sử dụng Mock Service).
*   **Concurrency Control:** Sử dụng `Context` của Go để quản lý timeout và hủy bỏ các tác vụ nền (scheduler) khi hệ thống shutdown.
*   **Human-readable IDs:** Thay vì dùng UUID thuần túy, hệ thống dùng prefix (ví dụ: `mc-` cho Managed Certificate, `ag-` cho Agent). Đây là kỹ thuật giúp việc debug qua log trở nên cực kỳ dễ dàng cho vận hành.
*   **Asynchronous Processing:** Khi người dùng yêu cầu làm mới chứng chỉ, Server không xử lý ngay mà tạo một `Job` trong DB. Một tiến trình nền (Scheduler) sẽ điều phối Job này. Điều này giúp API phản hồi cực nhanh (status 202 Accepted).

---

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng Cấp mới/Gia hạn Chứng chỉ:
1.  **Trigger:** User (UI/API) hoặc Scheduler (đến hạn) yêu cầu Renew.
2.  **Job Creation:** Server tạo một Job trạng thái `AwaitingCSR`.
3.  **Agent Polling:** Agent (đang chạy trên máy chủ Nginx/Apache) gọi API `/work`.
4.  **Key Generation:** Agent sinh Private Key tại chỗ -> tạo CSR (Certificate Signing Request) -> gửi CSR lên Server.
5.  **Signing:** Server nhận CSR -> chuyển tới `Issuer` (ví dụ Let's Encrypt qua ACME) -> Nhận lại chứng chỉ đã ký.
6.  **Deployment:** Agent nhận chứng chỉ -> Ghi vào file trên ổ đĩa -> Gọi lệnh reload (Nginx -s reload) -> Báo cáo kết quả về Server.

#### B. Luồng Discovery (Khám phá):
1.  **Network Scan:** Server quét các dải IP/Port (443) để tìm các chứng chỉ "lạ" chưa được quản lý.
2.  **Filesystem Scan:** Agent quét các thư mục cấu hình trên máy chủ đích để tìm file `.pem`, `.crt`.
3.  **Triage:** Các chứng chỉ tìm được hiện lên Dashboard để Admin chọn "Claim" (Đưa vào quản lý) hoặc "Dismiss" (Bỏ qua).

#### C. Luồng AI/MCP:
1.  Người dùng hỏi Claude: "Chứng chỉ nào sắp hết hạn?"
2.  Claude gọi tool qua `MCP Server`.
3.  `MCP Server` gọi API của `certctl Server`, lấy dữ liệu và trả về cho AI xử lý ngôn ngữ tự nhiên.

---

### Tổng kết
**certctl** là một dự án có độ hoàn thiện kỹ thuật rất cao. Nó không chỉ đơn thuần là một công cụ quản lý, mà là một **Control Plane** thực thụ cho hạ tầng PKI (Public Key Infrastructure). Việc kết hợp giữa Go (hiệu năng), React (trải nghiệm người dùng), và tư duy bảo mật "Keys stay at the edge" cho thấy tác giả là người có kinh nghiệm sâu sắc về DevSecOps và hệ thống phân tán.