Dựa trên thông tin từ kho lưu trữ, dưới đây là phân tích chi tiết về dự án **CK (Java Code Metrics Calculator)**:

### 1. Công nghệ cốt lõi (Core Technologies)
*   **Ngôn ngữ chính:** **Java (100%)**, yêu cầu tối thiểu Java 8 (hiện tại đang hỗ trợ Compliance Java 11).
*   **Thư viện phân tích cú pháp:** **Eclipse JDT (Java Development Tools) Core**. Đây là thành phần quan trọng nhất, được dùng để xây dựng Cây cú pháp trừu tượng (AST) và giải quyết các Binding (liên kết kiểu dữ liệu) mà không cần biên dịch mã nguồn.
*   **Phản chiếu (Reflection):** **Reflections Library**. Được sử dụng để tự động tìm kiếm và nạp các lớp đo lường (metric classes) tại thời điểm thực thi, giúp hệ thống có tính mở rộng cao.
*   **Xử lý dữ liệu:** 
    *   **Google Guava:** Dùng để phân đoạn (partitioning) danh sách tệp tin.
    *   **Apache Commons CSV:** Dùng để ghi kết quả đo lường ra định dạng CSV.
    *   **Apache Commons IO/Lang3:** Các tiện ích xử lý chuỗi và vào/ra tệp.
*   **Kiểm thử & Chất lượng:** JUnit 5, AssertJ, Jacoco (đo độ bao phủ), và tích hợp Codecov.

### 2. Tư duy Kiến trúc (Architectural Thinking)
*   **Phân tích tĩnh (Static Analysis):** CK được thiết kế để hoạt động trực tiếp trên mã nguồn (`.java`) mà không cần tệp đã biên dịch (`.class`), giúp tiết kiệm thời gian và tài nguyên trong môi trường CI/CD.
*   **Chiến lược "Chia để trị" (Batch Processing):** Để xử lý các dự án khổng lồ, CK sử dụng kỹ thuật **File Partitioning**. Dựa trên bộ nhớ RAM khả dụng, nó sẽ chia dự án thành nhiều đợt (ví dụ: 100 tệp mỗi đợt) để tránh lỗi `OutOfMemoryError`.
*   **Thiết kế hướng mở (Pluggable Architecture):** Kiến trúc cho phép thêm các chỉ số đo lường mới một cách dễ dàng. Bất kỳ lớp nào triển khai giao diện `ClassLevelMetric` hoặc `MethodLevelMetric` đều được `MetricsFinder` tự động nhận diện.
*   **Quản lý phân cấp:** CK tách biệt rõ ràng giữa các mức độ đo lường: Lớp (Class), Phương thức (Method), và Biến/Trường (Variable/Field). Nó xử lý chính xác các trường hợp phức tạp như lớp lồng nhau (Inner classes) và lớp ẩn danh (Anonymous classes).

### 3. Các kỹ thuật chính (Key Techniques)
*   **Visitor Pattern:** Sử dụng `CKVisitor` (kế thừa `ASTVisitor` của JDT) để duyệt qua các nút trên cây AST. Kỹ thuật này giúp tách biệt cấu trúc mã nguồn khỏi thuật toán đo lường.
*   **Sắp xếp Topo (Topological Sort):** Sử dụng `DependencySorter` để xử lý các chỉ số có sự phụ thuộc lẫn nhau. Thông qua annotation `@RunAfter`, CK đảm bảo chỉ số A được tính toán trước nếu chỉ số B cần kết quả của A (ví dụ: TCC/LCC cần kết quả từ RFC).
*   **Giải quyết liên kết (Binding Resolution):** CK thiết lập môi trường (Environment) bao gồm cả các tệp JAR phụ thuộc để JDT có thể hiểu được các kiểu dữ liệu từ thư viện bên ngoài, giúp tăng độ chính xác cho các chỉ số như CBO (Coupling Between Objects).
*   **Xử lý văn bản thông minh:** 
    *   **WordCounter:** Tách từ dựa trên CamelCase và dấu gạch dưới, loại bỏ từ khóa Java.
    *   **LOCCalculator:** Tính toán số dòng code thực tế bằng cách loại bỏ dòng trống và các loại comment (single line, block, javadoc).

### 4. Tóm tắt luồng hoạt động (Workflow Summary)
1.  **Khởi tạo (Runner/Runner CLI):** Nhận tham số đầu vào (đường dẫn dự án, có dùng JAR không, có đo biến không).
2.  **Khám phá (MetricsFinder):** Quét classpath để tìm tất cả các lớp thực hiện đo lường.
3.  **Quét tệp (FileUtils):** Tìm kiếm tất cả các tệp `.java` và `.jar` trong thư mục chỉ định, loại trừ các thư mục bị bỏ qua (như `.git`).
4.  **Phân đoạn (Partitioning):** Chia danh sách tệp thành các nhóm nhỏ dựa trên cấu hình bộ nhớ.
5.  **Thực thi (MetricsExecutor & CKVisitor):** 
    *   Với mỗi nhóm tệp, khởi tạo `ASTParser`.
    *   Tạo AST và duyệt cây bằng `CKVisitor`.
    *   Tại mỗi nút (Class/Method), các metric tương ứng sẽ thu thập dữ liệu.
6.  **Broadcast kết quả (CKNotifier):** Sau khi mỗi lớp được phân tích xong, kết quả được gửi đến `CKNotifier`.
7.  **Xuất dữ liệu (ResultWriter):** Runner nhận kết quả và sử dụng `ResultWriter` để ghi vào 3 tệp CSV riêng biệt (class.csv, method.csv, variable.csv).

CK là một công cụ mạnh mẽ, kết hợp giữa khả năng phân tích sâu của Eclipse JDT và tính linh hoạt của các mẫu thiết kế hướng đối tượng, tạo ra một giải pháp đo lường mã nguồn Java chuẩn mực và hiệu quả.