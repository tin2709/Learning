Dựa trên các tệp tin và cấu trúc mã nguồn được cung cấp, dưới đây là phân tích chuyên sâu về hệ thống **Floci** (Local AWS Emulator).

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Floci được xây dựng với mục tiêu hiệu suất cực cao và dấu chân bộ nhớ (memory footprint) thấp, vượt trội hơn so với các đối thủ như LocalStack.

*   **Ngôn ngữ & Runtime:** Sử dụng **Java 25** (phiên bản mới nhất) kết hợp với **Quarkus 3.34.3**. Việc chọn Quarkus cho phép ứng dụng khởi chạy trong ~24ms nhờ khả năng biên dịch **Native Image** (GraalVM).
*   **Giao thức AWS (Wire Protocols):** Đây là phần quan trọng nhất. Floci không chỉ giả lập API mà thực hiện lại các giao thức truyền tải của AWS:
    *   **Query (XML):** SQS, SNS, IAM, RDS... (Xử lý qua `AwsQueryController`).
    *   **JSON 1.1 / CBOR:** DynamoDB, KMS, Kinesis... (Xử lý qua `AwsJson11Controller`).
    *   **REST (JSON/XML):** S3, Lambda, API Gateway.
*   **Docker Integration:** Sử dụng `docker-java` để quản lý vòng đời của các dịch vụ nặng (Lambda, RDS, ElastiCache, MSK).
*   **Công cụ xử lý dữ liệu:**
    *   **Jackson:** Xử lý JSON, CBOR và YAML.
    *   **BouncyCastle:** Dùng cho ACM (quản lý chứng chỉ) và KMS (mã hóa).
    *   **Apache Velocity:** Công cụ template cho VTL (Velocity Mapping Templates) trong API Gateway.
    *   **JSONata:** Công cụ truy vấn dữ liệu phức tạp cho Step Functions.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Floci tuân thủ nguyên tắc phân lớp rõ rệt và trừu tượng hóa dịch vụ:

*   **Mô hình Stateless vs Stateful:**
    *   **Stateless Services:** Các dịch vụ như IAM, STS, SSM được thực hiện "In-process" (trong cùng một tiến trình Java), giúp tốc độ phản hồi cực nhanh.
    *   **Stateful Services:** S3 và DynamoDB có lớp lưu trữ riêng biệt.
*   **Trừu tượng hóa lưu trữ (Storage Abstraction):** Hệ thống không gắn chặt vào một cơ sở dữ liệu. Thông qua `StorageBackend` và `StorageFactory`, người dùng có thể cấu hình:
    *   `memory`: Mất dữ liệu khi restart (nhanh nhất).
    *   `persistent`: Lưu file JSON/Binary xuống đĩa.
    *   `wal` (Write-Ahead Logging): Ghi nhật ký thay đổi để đảm bảo tính toàn vẹn.
    *   `hybrid`: Kết hợp giữa tốc độ bộ nhớ và sự bền vững của đĩa.
*   **Phân tách Controller và Logic:** Các `Controller` (hoặc `Handler`) chỉ chịu trách nhiệm giải mã giao thức AWS (parsing headers, targets), sau đó ủy thác logic thực tế cho các `Service` lớp dưới.
*   **Descriptor-based Catalog:** Sử dụng `ServiceDescriptor` để định nghĩa metadata của từng dịch vụ (tên, ARN pattern, giao thức, cổng). Điều này giúp việc thêm dịch vụ mới không cần thay đổi code ở các bộ lọc (Filters).

---

### 3. Kỹ thuật Lập trình chính (Main Programming Techniques)

*   **Dependency Injection (DI):** Sử dụng `@ApplicationScoped` và Constructor Injection của Quarkus để quản lý các thành phần hệ thống, giúp mã nguồn dễ kiểm thử (Unit Test).
*   **Custom JAX-RS Filters:** Hệ thống sử dụng một chuỗi các Filters để xử lý các tác vụ xuyên suốt (cross-cutting concerns):
    *   `AwsRequestIdFilter`: Tự động thêm ID yêu cầu vào header để tương thích với SDK.
    *   `IamEnforcementFilter`: Kiểm tra quyền (Policy) trước khi thực hiện logic.
    *   `S3VirtualHostFilter`: Xử lý routing dựa trên Hostname (ví dụ: `my-bucket.s3.localhost.floci.cloud`).
*   **Xử lý XML thủ công:** Thay vì dùng các framework XML nặng nề, Floci sử dụng `XmlBuilder` và `XmlParser` tự viết để tối ưu hiệu suất và đảm bảo định dạng đầu ra khớp tuyệt đối với AWS.
*   **Reflection-Safe Modeling:** Các lớp Model được thiết kế để tương thích với biên dịch Native (GraalVM), tránh sử dụng quá nhiều reflection động vốn gây chậm và lỗi khi chạy ở dạng nhị phân.
*   **Container Detector:** Kỹ thuật tự động phát hiện nếu Floci đang chạy bên trong Docker để cấu hình lại mạng (Networking) cho Lambda và RDS một cách chính xác.

---

### 4. Luồng hoạt động của hệ thống (System Workflow)

1.  **Tiếp nhận (Ingress):**
    *   Client (AWS SDK/CLI) gửi Request tới cổng `4566`.
    *   `Router` của Quarkus tiếp nhận. Nếu là S3 Virtual Host, `S3VirtualHostFilter` sẽ phân tách tên Bucket từ URL.

2.  **Định danh giao thức (Protocol Routing):**
    *   Dựa trên HTTP Header (như `X-Amz-Target`) hoặc đường dẫn, Request được dẫn tới Controller tương ứng.
    *   Ví dụ: Nếu có `X-Amz-Target: AmazonSSM.GetParameter`, nó sẽ đi qua `AwsJson11Controller`.

3.  **Xử lý Logic (Service Execution):**
    *   Controller gọi `SsmService`.
    *   `SsmService` truy vấn dữ liệu từ `StorageBackend`.
    *   Nếu dịch vụ là Lambda, Floci sẽ kiểm tra `WarmPool`. Nếu chưa có container, nó sẽ gọi Docker API để kéo Image (từ ECR giả lập hoặc thực) và chạy container.

4.  **Lưu trữ & Phản hồi (Storage & Response):**
    *   Dữ liệu được cập nhật vào Store. Nếu ở chế độ `persistent`, dữ liệu được serialize thành JSON/file.
    *   Kết quả được đóng gói theo đúng định dạng AWS (XML hoặc JSON).
    *   `AwsDateHeaderFilter` thêm các thông tin thời gian chuẩn AWS trước khi trả về Client.

### Kết luận
Floci là một minh chứng của việc áp dụng **Modern Java (J25)** và **Cloud-native framework (Quarkus)** để giải quyết bài toán giả lập hệ thống phức tạp. Sự kết hợp giữa việc tối ưu hóa giao thức (Wire protocol) và quản lý container thông minh (Docker API) giúp nó trở thành một lựa chọn thay thế mạnh mẽ cho LocalStack trong môi trường CI/CD và phát triển tại địa phương.