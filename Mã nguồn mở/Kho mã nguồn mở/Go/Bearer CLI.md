Dựa trên các tệp mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về Bearer CLI - một công cụ Static Application Security Testing (SAST) hiện đại.

### 1. Công nghệ cốt lõi (Core Stack)

*   **Ngôn ngữ lập trình chính:** **Go (Golang)**. Đây là lựa chọn tối ưu cho các công cụ CLI nhờ tốc độ xử lý cao, khả năng song song hóa tốt và tạo ra các tệp thực thi duy nhất (binary) dễ phân phối.
*   **Engine phân tích cú pháp:** **Tree-sitter**. Bearer sử dụng Tree-sitter để xây dựng Cây cú pháp trừu tượng (AST). Công nghệ này giúp Bearer hỗ trợ đa ngôn ngữ (Ruby, JavaScript, Java, Python, Go...) một cách nhất quán và hiệu quả.
*   **Phân tích bảo mật:**
    *   **Taint Analysis (Phân tích vết):** Theo dõi luồng dữ liệu từ nguồn (Source - ví dụ: input của người dùng) đến điểm cuối (Sink - ví dụ: truy vấn SQL) để phát hiện lỗ hổng.
    *   **Secret Detection:** Tích hợp **Gitleaks** để quét các thông tin nhạy cảm (mật khẩu, API key) bị hard-code.
*   **Hệ thống quy tắc (Rule Engine):** Các quy tắc được định nghĩa dưới dạng **YAML**, cho phép mở rộng dễ dàng mà không cần can thiệp vào mã nguồn lõi.
*   **Tài liệu:** Sử dụng **Eleventy (11ty)**, Nunjucks và Tailwind CSS để tạo trang web tài liệu tĩnh từ các tệp dữ liệu YAML được sinh tự động từ mã nguồn Go.

### 2. Tư duy Kiến trúc (Architecture Thinking)

Kiến trúc của Bearer CLI được thiết kế theo hướng **"Data-first Security"** (Bảo mật ưu tiên dữ liệu) với các đặc điểm:

*   **Phân tách luồng xử lý:** Tách biệt rõ ràng giữa việc **Khám phá dữ liệu (Discovery)**, **Phân loại dữ liệu (Classification)** và **Áp dụng quy tắc (Rule Evaluation)**. Thay vì chỉ tìm các lỗi code thuần túy, Bearer cố gắng hiểu "dữ liệu nhạy cảm nằm ở đâu" trước.
*   **Hệ thống Recipes:** Sử dụng các tệp JSON (Recipes) để định nghĩa các thành phần bên thứ ba (Database, Cloud APIs, SaaS). Điều này giúp hệ thống nhận diện được luồng dữ liệu đi ra ngoài ứng dụng một cách chính xác.
*   **Tính module hóa:**
    *   `pkg/scanner`: Chứa lõi quét AST.
    *   `pkg/classification`: Chứa logic phân loại PII/PHI.
    *   `external/run`: Cung cấp một lớp trừu tượng (Facade) để gọi engine từ CLI hoặc các module khác.
*   **Differential Scanning (Quét vi sai):** Kiến trúc hỗ trợ quét chỉ những thay đổi trong Git (`--diff`), giúp tối ưu tốc độ cho quy trình CI/CD.

### 3. Kỹ thuật Lập trình Chính

*   **Pattern Matching trên AST:** Bearer định nghĩa một cú pháp truy vấn tùy chỉnh trong YAML (ví dụ: `$<CLIENT>.$<METHOD>($<...>)`) để khớp các mẫu code trên cây AST mà không cần dùng Regex phức tạp, giúp giảm thiểu False Positive.
*   **Snapshot Testing:** Trong thư mục `e2e/`, dự án sử dụng thư viện `cupaloy` để kiểm thử đầu ra của CLI. Kỹ thuật này chụp lại kết quả quét (Snapshot) và so sánh với lần chạy sau để đảm bảo không có lỗi hồi quy (regression).
*   **Facade Pattern:** Tệp `external/run/run.go` đóng gói toàn bộ logic phức tạp của engine vào các hàm đơn giản như `NewEngine`, `Run`, giúp việc bảo trì và tích hợp trở nên cực kỳ dễ dàng.
*   **Automated Doc Generation:** Sử dụng script Go (`gen-doc-yaml.go`) để trích xuất thông tin từ cấu trúc lệnh (Cobra) ra YAML, sau đó dùng 11ty để render ra tài liệu. Điều này đảm bảo tài liệu luôn đồng bộ 100% với các tính năng thực tế của CLI.

### 4. Luồng hoạt động của hệ thống (System Flow)

Luồng hoạt động của một phiên quét (`bearer scan`) diễn ra như sau:

1.  **Giai đoạn Khởi tạo:** Đọc cấu hình từ `bearer.yml` hoặc các flag CLI. Tải các quy tắc nội bộ và quy tắc tùy chỉnh từ thư mục bên ngoài.
2.  **Giai đoạn Parse (Phân tích cú pháp):** Quét các tệp tin trong thư mục chỉ định. Sử dụng Tree-sitter để chuyển đổi mã nguồn thành cây AST.
3.  **Giai đoạn Detection & Classification (Phát hiện & Phân loại):**
    *   Duyệt AST để tìm các biến, hàm, object.
    *   Sử dụng heuristics và pattern matching để xác định xem một biến có chứa dữ liệu nhạy cảm (Email, Credit Card, PII...) hay không.
    *   Gắn nhãn (Label) cho các nút trên cây AST.
4.  **Giai đoạn Rule Evaluation (Đánh giá quy tắc):**
    *   Chạy các quy tắc quét bảo mật (SAST) trên cây AST đã được gắn nhãn.
    *   Kiểm tra luồng dữ liệu nhạy cảm có đi vào các "Sink" nguy hiểm (logger, unencrypted socket, SQL query) mà không qua hàm làm sạch (Sanitizer) hay không.
5.  **Giai đoạn Tổng hợp & Báo cáo:** Thu thập các "Finding", tính toán mức độ nghiêm trọng (Severity) dựa trên ngữ cảnh dữ liệu và xuất ra định dạng yêu cầu (JSON, SARIF, HTML, CSV).

### 5. Điểm đặc biệt khác
*   **Hỗ trợ CI/CD sâu:** Có sẵn các cấu hình cho GitHub Actions (SARIF export), GitLab SAST, và hỗ trợ Reviewdog để comment trực tiếp lỗi vào Pull Request.
*   **Xử lý False Positive:** Cung cấp lệnh `bearer ignore` để tạo tệp `.ignore` dựa trên vân tay (fingerprint) của từng lỗi, giúp bỏ qua các cảnh báo không cần thiết một cách bền vững.