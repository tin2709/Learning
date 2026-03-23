Chào bạn, đây là bản phân tích chi tiết về dự án **Cerbos** dựa trên cấu trúc mã nguồn và tài liệu bạn cung cấp. Cerbos là một hệ thống quản lý quyền truy cập (Authorization) hiện đại, tách biệt logic phân quyền ra khỏi mã nguồn ứng dụng.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Ngôn ngữ lập trình:** **Go (Golang)** chiếm hơn 93%, tận dụng lợi thế về hiệu suất xử lý song song và khả năng đóng gói thành binary gọn nhẹ.
*   **Giao thức giao tiếp:**
    *   **gRPC & Protobuf:** Sử dụng làm phương thức giao tiếp chính để đảm bảo tốc độ cao và định nghĩa API chặt chẽ.
    *   **Connect RPC:** Một thư viện hiện đại hỗ trợ cả gRPC, gRPC-Web và HTTP/JSON trong cùng một handler.
    *   **HTTP/REST (với gRPC-Gateway):** Cho phép các ứng dụng không dùng gRPC vẫn có thể gọi API.
*   **Công cụ xử lý Policy:**
    *   **CEL (Common Expression Language):** Một ngôn ngữ biểu thức của Google dùng để viết các điều kiện (conditions) trong policy. Nó an toàn, nhanh và nhẹ.
*   **Lưu trữ & Dữ liệu:**
    *   **Drivers đa dạng:** Hỗ trợ nạp policy từ Disk, Git, S3 (Blob), MySQL, Postgres, SQLite.
    *   **BadgerDB:** Database dạng Key-Value nhúng để lưu trữ audit logs cục bộ.
*   **DevOps & Infrastructure:**
    *   **Docker:** Có các Dockerfile tối ưu cho cả server và CLI (`cerbosctl`).
    *   **Helm:** Cung cấp Chart để triển khai dễ dàng trên Kubernetes.
    *   **Just:** Sử dụng `justfile` thay cho `Makefile` để quản lý các lệnh build/test.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Cerbos được xây dựng dựa trên mô hình **Stateless PDP (Policy Decision Point)**:

*   **Tách biệt (Decoupling):** Thay vì viết `if (user.role == 'admin')` trong code ứng dụng, ứng dụng chỉ hỏi Cerbos: "Người dùng A có được làm hành động X trên tài nguyên Y không?". Logic thực sự nằm trong các file YAML.
*   **Kiến trúc Stateless:** Bản thân Cerbos PDP không lưu trạng thái phiên làm việc, giúp nó có thể mở rộng (scale) ngang cực kỳ dễ dàng bằng Load Balancer.
*   **Cấu trúc Monorepo:** Quản lý cả Server, CLI, SDK và tài liệu trong một kho lưu trữ để đảm bảo tính đồng bộ giữa API và các công cụ thực thi.
*   **Plugin-based Storage:** Kiến trúc cho phép Cerbos theo dõi (watch) sự thay đổi của policy từ nhiều nguồn khác nhau (ví dụ: tự động reload khi file trên Git thay đổi) mà không cần khởi động lại server.
*   **Audit-first:** Tư duy quản trị và tuân thủ (compliance) thể hiện qua việc tích hợp sẵn hệ thống ghi log quyết định (decision logs) vào Kafka hoặc Database.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Code Generation:** Sử dụng `buf` và `protoc` mạnh mẽ để tạo mã nguồn từ file `.proto` (`api/genpb`). Điều này giúp duy trì sự nhất quán tuyệt đối giữa Client và Server.
*   **Hash-based Indexing:** Sử dụng kỹ thuật hash (`hashpb`) để so sánh và lập chỉ mục các policy nhanh chóng, giúp việc tra cứu policy khi có request đạt tốc độ cực cao.
*   **Lazy Evaluation:** Trong mã nguồn có đề cập đến "lazily-evaluated variables", giúp tối ưu hiệu suất bằng cách chỉ tính toán các biến khi chúng thực sự cần thiết trong biểu thức CEL.
*   **Abstraction Layer (Internal/Storage):** Sử dụng các Interface trong Go để trừu tượng hóa việc nạp dữ liệu. Bất kể policy đến từ Git hay S3, Engine lõi vẫn xử lý như nhau thông qua lớp `storage.Store`.
*   **Trường phái Defensive Programming:** Kiểm tra lỗi chặt chẽ, sử dụng linter (`.golangci.yaml`) và hệ thống test suite đồ sộ (unit test, integration test, E2E) với các "golden files" (kết quả mong đợi mẫu).

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng đi của một yêu cầu phân quyền trong Cerbos diễn ra như sau:

1.  **Nạp Policy (Startup/Runtime):**
    *   Cerbos khởi động và kết nối với nguồn lưu trữ (ví dụ: thư mục `/policies`).
    *   Nó phân tích các file YAML, kiểm tra cú pháp và xây dựng một bảng chỉ mục (Index) trong bộ nhớ.
2.  **Tiếp nhận Request:**
    *   Ứng dụng gửi một request đến Cerbos API (gRPC hoặc HTTP).
    *   Request bao gồm: **Principal** (Ai?), **Action** (Làm gì?), **Resource** (Trên cái gì?) và **Context** (Thông tin bổ sung).
3.  **Xử lý Engine:**
    *   **Match:** Engine tìm kiếm các policy phù hợp với `Resource` và `Principal`.
    *   **Evaluate:** Nếu policy có `condition`, Engine sẽ gọi trình thực thi CEL để tính toán biểu thức dựa trên `Context` và thuộc tính của Resource.
4.  **Ra quyết định (Decision):**
    *   Kết quả trả về là `EFFECT_ALLOW` hoặc `EFFECT_DENY`.
    *   Đồng thời đính kèm các `Outputs` (dữ liệu bổ sung cho ứng dụng) nếu được cấu hình.
5.  **Ghi log (Audit):**
    *   Quyết định cùng toàn bộ context được đẩy vào hệ thống Audit Log để phục vụ tra cứu sau này.

### Tổng kết
Cerbos là một ví dụ mẫu mực về cách xây dựng **Security Infrastructure** hiện đại: Sử dụng Go để đạt tốc độ, YAML để thân thiện với con người, gRPC để tối ưu giao tiếp và CEL để linh hoạt trong logic. Dự án này cực kỳ phù hợp cho các hệ thống Microservices phức tạp cần một cơ chế phân quyền tập trung và nhất quán.