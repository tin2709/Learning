Dựa trên các tệp tin và cấu trúc mã nguồn bạn đã cung cấp, dưới đây là phân tích chuyên sâu về hệ thống **CapRover** (trước đây là CaptainDuckDuck) – một nền tảng PaaS (Platform as a Service) mã nguồn mở.

---

### 1. Công nghệ cốt lõi (Core Technologies)

CapRover được xây dựng dựa trên sự kết hợp của các công cụ hạ tầng mạnh mẽ và ngôn ngữ lập trình hiện đại:

*   **Ngôn ngữ & Runtime:** 
    *   **TypeScript (95.5%):** Toàn bộ Backend được viết bằng TypeScript, đảm bảo tính chặt chẽ về kiểu dữ liệu (type-safety) và dễ bảo trì.
    *   **Node.js:** Runtime chính cho Backend (sử dụng Node 22 trong các Dockerfile mới nhất).
*   **Hạ tầng Containerization:**
    *   **Docker Swarm:** Đây là "xương sống" của hệ thống. Khác với nhiều công cụ chỉ dùng Docker đơn lẻ, CapRover tận dụng Swarm để quản lý clustering, tự động hồi phục (self-healing) và điều phối container.
    *   **Dockerode:** Thư viện Node.js để giao tiếp trực tiếp với Docker Socket API.
*   **Networking & Proxy:**
    *   **Nginx:** Đóng vai trò là Reverse Proxy và Load Balancer. CapRover tạo cấu hình Nginx động cho từng ứng dụng.
    *   **Let's Encrypt (Certbot):** Tự động hóa hoàn toàn việc cấp phát và gia hạn chứng chỉ SSL/TLS.
*   **Database (Internal):**
    *   **File-based Store:** Sử dụng hệ thống lưu trữ dựa trên tệp JSON (`src/datastore/`) thay vì một database phức tạp, giúp việc backup và di trú (migration) cực kỳ đơn giản.
*   **Giám sát (Monitoring):**
    *   **NetData & GoAccess:** Cung cấp thông số thời gian thực và phân tích log truy cập.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của CapRover đi theo triết lý **"Abstractions over Complexity"** (Trừu tượng hóa sự phức tạp):

*   **Kiến trúc Manager-Worker (Internal):** Mã nguồn được chia thành các "Manager" chuyên biệt (`CertbotManager`, `ServiceManager`, `BackupManager`). Mỗi Manager chịu trách nhiệm duy nhất cho một miền logic, giúp hệ thống module hóa cao.
*   **Kiến trúc "No Lock-in":** Đây là tư duy cực kỳ quan trọng. CapRover được thiết kế để nếu bạn xóa CapRover đi, các ứng dụng của bạn (đang chạy dưới dạng Docker Swarm Services) vẫn hoạt động bình thường. CapRover chỉ đóng vai trò là "người điều khiển" (Orchestrator).
*   **Template-driven:** Thay vì code cứng (hard-code) các cấu hình, CapRover sử dụng EJS templates (`template/*.ejs`) để sinh ra cấu hình Nginx. Điều này cho phép người dùng nâng cao tùy chỉnh (Custom Nginx Config) mà không làm hỏng logic hệ thống.
*   **Single Source of Truth:** Mọi trạng thái của hệ thống được lưu trong `config-captain.json`. Khi hệ thống khởi động lại, nó sẽ đọc tệp này và đồng bộ hóa lại trạng thái với Docker Swarm.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Dependency Injection (DI) & Providers:** Sử dụng các pattern như `UserManagerProvider`, `DataStoreProvider` để quản lý việc khởi tạo và cung cấp các instance xuyên suốt ứng dụng.
*   **Asynchronous Orchestration:** Xử lý các tác vụ nặng như Build Docker Image, Clone Git Repository bằng `async/await` và hệ thống hàng đợi (Queue) để tránh nghẽn luồng chính.
*   **Middleware-based API:** Sử dụng Express.js với cấu trúc Router/Handler rõ ràng (`src/routes/` và `src/handlers/`), tách biệt logic xử lý HTTP và logic nghiệp vụ.
*   **Security hardening:** 
    *   Sử dụng JWT cho xác thực.
    *   Mã hóa dữ liệu nhạy cảm (như Registry Password) bằng `Encryptor.ts`.
    *   Chặn Brute-force login.
*   **Tận dụng tối đa Docker API:** Thay vì chạy lệnh shell (như `docker run`), CapRover gọi trực tiếp Docker API qua socket để có sự kiểm soát chi tiết hơn về log và trạng thái container.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng đi của một yêu cầu triển khai ứng dụng (Deployment Workflow):

1.  **Tiếp nhận (Input):** Người dùng đẩy mã nguồn qua Git Webhook, tệp Tarball, hoặc Docker Image.
2.  **Xây dựng (ImageMaker):** 
    *   Hệ thống kiểm tra `captain-definition` để biết cách build.
    *   Sử dụng Docker Engine để build image.
    *   Nếu có Registry riêng, Image sẽ được đẩy (push) lên đó.
3.  **Điều phối (ServiceManager):** 
    *   Cập nhật Docker Swarm Service với Image mới.
    *   Thiết lập các biến môi trường (Env Vars), Volume, và Port mapping.
4.  **Cấu hình mạng (LoadBalancerManager):**
    *   Cập nhật tệp cấu hình Nginx để trỏ tên miền (ví dụ: `myapp.domain.com`) vào Service vừa tạo trong mạng nội bộ của Docker.
    *   Reload Nginx mà không gây gián đoạn (zero-downtime).
5.  **Bảo mật (CertbotManager):** 
    *   Nếu HTTPS được bật, hệ thống sẽ thực hiện thử thách (challenge) với Let's Encrypt và cập nhật chứng chỉ vào Nginx.
6.  **Giám sát:** Log được đẩy ra `BuildLog.ts` để người dùng theo dõi thời gian thực qua Web GUI.

---

### Tổng kết
CapRover là một ví dụ điển hình về việc đóng gói (encapsulation) các công nghệ hạ tầng phức tạp thành một công cụ đơn giản cho lập trình viên. Nó không cố gắng thay thế Docker mà đóng vai trò là một lớp **Management Layer** thông minh, tập trung vào hiệu suất của developer (Developer Experience - DX).