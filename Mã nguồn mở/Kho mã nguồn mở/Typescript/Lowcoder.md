Dựa trên cấu trúc thư mục và nội dung các tệp tin từ dự án **Lowcoder**, dưới đây là phân tích chi tiết về dự án này theo các yêu cầu của bạn:

---

### 1. Công Nghệ Cốt Lõi (Core Technologies)

Lowcoder là một hệ thống full-stack hiện đại, sử dụng kiến trúc Monorepo để quản lý nhiều gói phần mềm khác nhau:

*   **Frontend (Client):**
    *   **Ngôn ngữ:** TypeScript (chiếm tỉ trọng lớn), đảm bảo tính chặt chẽ về dữ liệu.
    *   **Framework:** React 18.
    *   **Build Tool:** Vite (cho tốc độ phát triển nhanh) và Rollup/Webpack cho việc đóng gói SDK.
    *   **State Management:** Redux phối hợp với Redux-Saga để quản lý các side-effects phức tạp (như gọi API, xử lý dữ liệu query).
    *   **UI Library:** Ant Design (v5) là nền tảng giao diện, cùng với Styled-components để tùy chỉnh style.
    *   **Editor:** CodeMirror được sử dụng để xây dựng trình soạn thảo code/truy vấn trong ứng dụng.
    *   **Visualization:** ECharts cho các thành phần biểu đồ.

*   **Backend (Server):**
    *   **Java Service (api-service):** Sử dụng Spring Boot/WebFlux (dựa trên các file `pom.xml` và cấu trúc reactive). Đây là nơi xử lý logic nghiệp vụ chính, quyền (RBAC), và quản lý metadata.
    *   **Node Service (node-service):** Sử dụng Node.js để thực thi các đoạn mã JavaScript của người dùng trong môi trường cô lập, đảm bảo hiệu năng và tính bảo mật.

*   **Lưu trữ & Hạ tầng:**
    *   **Database:** MongoDB (lưu trữ định nghĩa ứng dụng - DSL, cấu hình người dùng) và Redis (caching, rate limiting).
    *   **DevOps:** Docker, Docker Compose, Kubernetes (Helm Charts) cho triển khai đa nền tảng.

---

### 2. Tư Duy Kiến Trúc (Architectural Thinking)

Kiến trúc của Lowcoder tập trung vào khả năng **mở rộng** và **nhúng (embeddable)**:

*   **Kiến trúc dựa trên DSL (Domain Specific Language):** Mọi ứng dụng tạo ra bởi người dùng thực chất là một tệp JSON lớn (DSL). Giao diện người dùng, cấu hình truy vấn và logic liên kết đều được lưu trữ dưới dạng dữ liệu. Khi chạy, hệ thống sẽ "render" tệp JSON này thành ứng dụng thực tế.
*   **Kiến trúc Microservices/Modular:** Tách biệt rõ ràng giữa dịch vụ điều phối (Java) và dịch vụ thực thi mã (Node.js). Điều này giúp hệ thống an toàn hơn khi người dùng viết mã JS tùy chỉnh.
*   **Plugin-based Architecture:** Cho phép cộng đồng phát triển thêm các thành phần UI (`lowcoder-comps`) hoặc các nguồn dữ liệu mới (`lowcoder-plugins`) một cách độc lập mà không cần can thiệp vào lõi (core).
*   **Native Embed:** Khác với nhiều đối thủ dùng iFrame, Lowcoder phát triển một bộ SDK (`lowcoder-sdk`) cho phép nhúng ứng dụng trực tiếp vào trang web khác như một thành phần bản địa, giúp cải thiện hiệu suất và SEO.

---

### 3. Các Kỹ Thuật Chính (Key Techniques)

*   **Hệ thống Kéo-Thả (Drag-and-Drop):** Sử dụng `react-grid-layout` và các thư viện dnd-kit để xây dựng lưới (grid) linh hoạt, cho phép người dùng sắp xếp vị trí các thành phần UI một cách trực quan.
*   **Reactivity (Tính phản ứng):** Hệ thống sử dụng một cơ chế quan sát (observable) để tự động cập nhật UI khi dữ liệu từ các câu lệnh truy vấn (queries) thay đổi. Khi một `query.data` thay đổi, tất cả các component đang bind vào data đó sẽ re-render.
*   **Sandboxed JS Execution:** Mã JavaScript do người dùng viết được chạy trong một môi trường sandbox (thường là qua `node-service`) để ngăn chặn các cuộc tấn công XSS hoặc truy cập trái phép vào hệ thống máy chủ.
*   **RBAC (Role-Based Access Control):** Hệ thống phân quyền chi tiết đến từng ứng dụng, thư mục và nguồn dữ liệu (Datasource), hỗ trợ cả môi trường Enterprise.
*   **Versioning & Snapshots:** Kỹ thuật lưu trữ lịch sử ứng dụng (Snapshots) cho phép người dùng khôi phục lại các phiên bản cũ, tương tự như hệ thống Git.

---

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

Luồng hoạt động có thể chia làm 2 giai đoạn chính:

#### A. Giai đoạn Thiết kế (Design Phase):
1.  **Người dùng** đăng nhập và vào trình soạn thảo (App Editor).
2.  **Kéo thả:** Người dùng chọn component từ thư viện -> Hệ thống cập nhật trạng thái DSL trong Redux.
3.  **Cấu hình Data:** Người dùng kết nối với Database (Postgres, MySQL...) -> Thông tin cấu hình được mã hóa và lưu qua `api-service`.
4.  **Lưu trữ:** Mỗi thay đổi được gửi về `api-service` (Java) để lưu vào MongoDB dưới dạng JSON DSL.

#### B. Giai đoạn Thực thi (Runtime Phase):
1.  **Tải ứng dụng:** Khi người dùng mở link ứng dụng, Frontend tải tệp DSL tương ứng.
2.  **Khởi tạo:** `RootComp` sẽ quét toàn bộ DSL, khởi tạo các thành phần UI và thiết lập các liên kết dữ liệu.
3.  **Truy vấn dữ liệu (Query Flow):**
    *   Component trigger một Query.
    *   Yêu cầu gửi đến `api-service`.
    *   `api-service` lấy thông tin kết nối, gửi yêu cầu thực thi kèm tham số đến **Datasource Plugin**.
    *   Nếu có xử lý JavaScript hậu kỳ (transformer), `node-service` sẽ thực hiện.
4.  **Cập nhật giao diện:** Kết quả trả về Frontend -> Redux cập nhật -> Các component nhận dữ liệu mới và hiển thị cho người dùng.

---

### Tóm lại:
Lowcoder không chỉ là một trình kéo thả UI, mà là một hệ thống **orchestration** (điều phối) phức tạp, kết hợp giữa sự an toàn của Java ở backend và sự linh hoạt của React/Node.js ở frontend để biến dữ liệu thô thành các ứng dụng doanh nghiệp có khả năng tương tác cao.