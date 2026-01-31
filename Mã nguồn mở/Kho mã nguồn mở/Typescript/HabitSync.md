Dưới đây là phân tích chi tiết về dự án **HabitSync** dựa trên mã nguồn và tài liệu đi kèm:

---

### 1. Công Nghệ Cốt Lõi (Core Technologies)

Dự án sử dụng mô hình **Client-Server** tách biệt hoàn toàn, tối ưu cho việc tự triển khai (self-hosting) và mở rộng đa nền tảng:

*   **Backend (habitsync-api):**
    *   **Ngôn ngữ & Framework:** **Java 21** kết hợp với **Spring Boot 3.4+**. Đây là sự lựa chọn hiện đại, tận dụng hiệu năng của Virtual Threads và bảo mật mạnh mẽ.
    *   **Cơ sở dữ liệu:** Mặc định sử dụng **H2** (lưu trữ dạng file cho nhẹ) nhưng hỗ trợ đầy đủ **PostgreSQL** cho môi trường sản xuất. Sử dụng **Flyway** để quản lý di cư dữ liệu (DB Migration).
    *   **Xác thực & Bảo mật:** Hỗ trợ đa dạng từ **OIDC (OpenID Connect)** với luồng **PKCE** (cho Mobile/PWA), **OAuth2**, đến **Basic Auth** (BCrypt) và **API Key**.
    *   **Lập lịch (Scheduling):** Sử dụng **Quartz Scheduler** để xử lý các tác vụ ngầm như tính toán thử thách hàng tháng và gửi thông báo nhắc nhở.
*   **Frontend (habitsync-ui):**
    *   **Framework:** **React Native** kết hợp với **Expo**. Điều này cho phép dự án xuất bản đồng thời dưới dạng **Web App (PWA)** và **Mobile App (Android)** từ cùng một cơ sở mã.
    *   **Quản lý trạng thái:** Sử dụng **React Query (TanStack Query)** để đồng bộ hóa dữ liệu từ server và **Context API** cho các trạng thái toàn cục (Auth, Theme, Alert).
    *   **Ngôn ngữ:** **TypeScript** giúp kiểm soát kiểu dữ liệu chặt chẽ giữa Client và API.
*   **Triển khai (DevOps):**
    *   **Docker & Docker Compose:** Sử dụng **Multi-stage build** (Node builder -> Maven builder -> Runtime image) để tối ưu dung lượng ảnh Docker.
    *   **Apprise:** Tích hợp engine thông báo mạnh mẽ, hỗ trợ hơn 80 dịch vụ (Discord, Telegram, Gotify, Email...).

---

### 2. Tư Duy Kiến Trúc (Architectural Thinking)

HabitSync không chỉ là một ứng dụng quản lý cá nhân mà được thiết kế như một **Nền tảng Xã hội (Social Platform)**:

*   **API-First Architecture:** Backend cung cấp hệ thống RESTful API hoàn chỉnh với tài liệu Swagger. Mọi tính năng trên UI đều thông qua API, cho phép bên thứ ba (như Home Assistant) tích hợp dễ dàng.
*   **Kiến trúc Đa phát hành (Multi-issuer Support):** Cho phép một hệ thống chấp nhận đăng nhập từ nhiều nguồn OIDC khác nhau (ví dụ: vừa dùng Authelia cá nhân vừa dùng Google Auth).
*   **Tách biệt logic tính toán (Computation Logic):** Logic tính toán tiến độ habit (Daily, Weekly, Monthly, X times per Y days) được đóng gói trong các Service riêng biệt (`CachingHabitProgressService`), tách rời khỏi luồng xử lý HTTP.
*   **Thiết kế hướng người dùng (User-centric):** Cơ chế phê duyệt tài khoản mới (`NEEDS-CONFIRMATION`) cho phép chủ sở hữu kiểm soát ai được tham gia vào server riêng của mình.

---

### 3. Các Kỹ Thuật Chính (Key Techniques)

*   **Cơ chế Caching thông minh:** Sử dụng `@Cacheable` của Spring Cache cho các tác vụ nặng như:
    *   Tính toán tỷ lệ hoàn thành (Percentage) của habit.
    *   Đếm số lượng habit liên kết.
    *   Hệ thống tự động xóa cache (`evict`) ngay khi có bản ghi (record) mới được tạo.
*   **Quản lý Quyền hạn Phức tạp:** `PermissionChecker` thực hiện kiểm tra quyền truy cập chéo giữa các người dùng trong các Habit chia sẻ (Shared Habits) và Thử thách (Challenges).
*   **Xử lý tệp tin Docker linh hoạt:** Sử dụng script `docker-entrypoint.sh` để map linh hoạt `PUID` và `PGID`, giúp giải quyết vấn đề phân quyền ghi file trên Linux khi gắn Volume (vấn đề phổ biến trong self-hosting).
*   **Logic Habit "Âm" (Negative Habits):** Kỹ thuật đảo ngược logic tính toán để theo dõi việc "giảm bớt" các hành vi xấu, không chỉ đơn thuần là đạt được mục tiêu dương.
*   **Hệ thống thông báo dựa trên Trigger:** Thay vì gửi thông báo rác, hệ thống tính toán các ngưỡng (threshold) hoặc sự kiện "vượt mặt" (overtake) giữa bạn bè để gửi thông báo thúc đẩy động lực.

---

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

1.  **Luồng Đăng nhập:** Người dùng chọn OIDC hoặc Basic Auth -> Server xác thực -> Nếu là người dùng mới, tài khoản vào trạng thái "Chờ phê duyệt" -> Quản trị viên duyệt -> Cấp JWT Token.
2.  **Luồng Ghi nhận Habit:** Người dùng tương tác trên App -> Gửi POST request tới `/api/record` -> Server lưu bản ghi vào DB -> Xóa Cache cũ -> Tính toán lại tiến độ thời gian thực.
3.  **Luồng Chia sẻ & Xã hội:**
    *   Người dùng tạo mã chia sẻ (Share Code).
    *   Bạn bè nhập mã -> Hệ thống tạo liên kết (Link) giữa các Habit cá nhân vào một `SharedHabit`.
    *   Mỗi khi một người hoàn thành, thông báo được gửi đến những người còn lại thông qua Apprise.
4.  **Luồng Thử thách hàng tháng:**
    *   Đầu tháng: Quartz tự động chọn Thử thách có lượt vote cao nhất để kích hoạt.
    *   Trong tháng: Người dùng tham gia và ghi nhận kết quả.
    *   Cuối tháng: Quartz tính toán thứ hạng (Placement) -> Lưu kết quả vào `ChallengeResult` -> Cấp Huy chương (Medals) cho người dùng.

### Tổng kết
**HabitSync** là một dự án có độ hoàn thiện cao về mặt kỹ thuật, kết hợp giữa sự ổn định của hệ sinh thái **Java/Spring** và sự linh hoạt của **React Native/PWA**, cung cấp một giải pháp theo dõi thói quen vừa riêng tư vừa có tính kết nối xã hội cao.