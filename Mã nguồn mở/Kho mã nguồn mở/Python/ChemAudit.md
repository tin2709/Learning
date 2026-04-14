Dựa trên mã nguồn và tài liệu kỹ thuật của dự án **ChemAudit**, dưới đây là phân tích chuyên sâu về hệ thống:

### 1. Công nghệ cốt lõi (Core Technology Stack)

Hệ thống được xây dựng trên sự kết hợp giữa các công nghệ hiện đại cho Web và các thư viện chuyên ngành hóa tin học (Cheminformatics):

*   **Hóa tin học (Cheminformatics):**
    *   **RDKit (Python & JS):** "Trái tim" của hệ thống, xử lý mọi tác vụ từ đọc SMILES/InChI, tính toán đặc tính phân tử, đến vẽ cấu trúc 2D (qua RDKit.js trên Frontend).
    *   **chembl-structure-pipeline:** Chuẩn hóa cấu trúc theo tiêu chuẩn của cơ sở dữ liệu ChEMBL.
    *   **OPSIN:** Một thư viện Java (chạy qua JAR) để chuyển đổi tên gọi IUPAC sang cấu trúc hóa học.
*   **Backend:** 
    *   **FastAPI:** Framework hiệu suất cao dựa trên Python 3.11+, tận dụng `async/await` cho các tác vụ I/O.
    *   **Pydantic v2:** Đảm bảo tính toàn vẹn của dữ liệu thông qua việc kiểm chứng (validation) nghiêm ngặt tại các endpoint API.
    *   **Celery & Redis:** Xử lý các tác vụ nặng (batch processing) bất đồng bộ. Redis đóng vai trò là Broker cho Celery, Cache lưu kết quả và là kho lưu trữ cho Rate Limiting.
*   **Frontend:**
    *   **React 18 & TypeScript:** Xây dựng giao diện người dùng kiểu Type-safe.
    *   **Tailwind CSS & Framer Motion:** Tạo giao diện hiện đại với hiệu ứng mượt mà (Claymorphism design).
*   **Hệ sinh thái AI:**
    *   **MCP (Model Context Protocol):** Cung cấp giao thức để các trợ lý AI (như Claude) có thể gọi trực tiếp hơn 60 công cụ hóa học của ChemAudit như một "kỹ năng" mở rộng.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án áp dụng mô hình **Phân lớp (Layered Architecture)** kết hợp với **Kiến trúc hướng dịch vụ (Service-Oriented)**:

*   **Tách biệt tác vụ (Separation of Concerns):** Hệ thống tách biệt rõ ràng giữa các yêu cầu tức thời (Single Validation - phản hồi nhanh) và các yêu cầu xử lý hàng triệu phân tử (Batch Validation - thông qua hàng đợi Celery).
*   **Thiết kế cho khả năng mở rộng (Scalability):** Sử dụng các file cấu hình YAML (`small.yml` đến `xl.yml`) để định nghĩa giới hạn tài nguyên, số lượng worker và kích thước file tối đa tùy theo môi trường triển khai.
*   **Bảo mật đa lớp (Defense in Depth):** 
    *   **Rate Limiting:** Giới hạn theo IP và API Key (sử dụng SlowAPI).
    *   **Session Isolation:** Cách ly dữ liệu giữa các phiên làm việc của người dùng bằng Row-Level Security (RLS) của PostgreSQL và HttpOnly Cookies.
    *   **Input Sanitization:** Làm sạch dữ liệu hóa học đầu vào để tránh các lỗi từ thư viện RDKit cấp thấp.
*   **Tính snapshot:** Các Permalink được thiết kế để chụp lại trạng thái dữ liệu tại thời điểm chia sẻ, đảm bảo kết quả không thay đổi ngay cả khi dữ liệu gốc trong Redis hết hạn (TTL).

### 3. Kỹ thuật Lập trình chính (Main Programming Techniques)

*   **Registry Pattern:** `CheckRegistry` được sử dụng để đăng ký hơn 15 loại kiểm tra cấu trúc hóa học khác nhau. Kỹ thuật này giúp dễ dàng thêm các loại kiểm tra mới mà không cần sửa đổi mã nguồn cốt lõi của Engine.
*   **Factory Pattern:** `ExporterFactory` quản lý việc tạo ra các tệp xuất dữ liệu (CSV, Excel, SDF, JSON, PDF) dựa trên yêu cầu của người dùng.
*   **Dependency Injection:** Tận dụng hệ thống `Depends` của FastAPI để quản lý kết nối Database, kiểm tra API Key và xác thực quyền sở hữu Job.
*   **Batch Chunking:** Kỹ thuật chia nhỏ các bộ dữ liệu lớn (lên đến 1 triệu phân tử) thành các "chunk" để xử lý song song trên nhiều worker, tránh gây nghẽn hệ thống.
*   **WebSocket State Management:** Quản lý trạng thái tiến độ thực tế (Progress tracking) thông qua Redis Pub/Sub, cho phép Frontend nhận cập nhật theo thời gian thực mà không cần polling liên tục.

### 4. Luồng hoạt động của hệ thống (System Workflow)

**A. Luồng xử lý đơn lẻ (Single Molecule):**
1. Người dùng nhập SMILES/InChI hoặc vẽ cấu trúc.
2. Request gửi đến API -> Middleware kiểm tra Rate Limit/API Key.
3. `MoleculeParser` kiểm tra định dạng và tính hợp lệ ban đầu.
4. `ValidationEngine` chạy chuỗi kiểm tra (Valence, Aromaticity, Stereo...).
5. Trả về kết quả kèm theo các gợi ý sửa lỗi (Diagnostics).

**B. Luồng xử lý hàng loạt (Batch Processing):**
1. Người dùng upload file (CSV/SDF).
2. API ghi nhận file, tạo `job_id` và lưu thông tin sở hữu vào Redis.
3. Một Task được đẩy vào Celery.
4. Worker lấy Task, parse file, chia nhỏ dữ liệu và thực hiện tính toán song song.
5. Trong quá trình chạy, Worker cập nhật tiến độ vào Redis.
6. Giao diện người dùng nhận cập nhật qua WebSocket.
7. Khi hoàn tất, hệ thống tổng hợp dữ liệu (Aggregation) và thông báo qua Email/Webhook nếu có cấu hình.

**C. Luồng tích hợp dữ liệu (External Integration):**
1. Nhận Identifier (ví dụ: "Aspirin").
2. `Universal Resolver` gọi đồng thời các API bên ngoài (PubChem, ChEMBL, COCONUT, Wikidata).
3. `Comparator Service` thực hiện đối soát cấu trúc giữa các nguồn để tìm điểm khác biệt về hóa học lập thể (stereochemistry).

### Kết luận
ChemAudit là một nền tảng hóa tin học toàn diện, kết hợp chặt chẽ giữa logic nghiệp vụ hóa học khắt khe và kiến trúc phần mềm hiện đại, có khả năng mở rộng tốt và tích hợp sâu với các công nghệ AI mới nhất.