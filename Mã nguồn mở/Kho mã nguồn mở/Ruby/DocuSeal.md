Dựa trên mã nguồn và cấu trúc thư mục của **DocuSeal**, đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống này:

### 1. Công nghệ cốt lõi (Core Tech Stack)

DocuSeal được xây dựng theo mô hình "Full-stack Framework" mạnh mẽ, tối ưu cho việc xử lý tài liệu:

*   **Backend:** **Ruby on Rails** (phiên bản mới nhất, hỗ trợ Ruby 4.0.1). Sử dụng triết lý "Convention over Configuration".
*   **Frontend:** Sự kết hợp giữa **Hotwire (Turbo/Stimulus)** cho các trang quản trị đơn giản và **Vue.js 3** cho các thành phần tương tác phức tạp như *Template Builder* (trình kéo thả field) và *Submission Form*.
*   **Xử lý PDF:**
    *   **HexaPDF:** Thư viện Ruby để đọc, chỉnh sửa và tạo annotation trên PDF.
    *   **Pdfium:** Sử dụng thông qua C-bindings để render ảnh preview cực nhanh từ các trang PDF.
*   **Machine Learning:** **ONNX Runtime**. DocuSeal tích hợp sẵn một model AI (`model_704_int8.onnx`) để tự động nhận diện các ô nhập liệu (Field Detection) trong file PDF trắng.
*   **Lưu trữ:** **ActiveStorage** hỗ trợ đa nền tảng (Disk, AWS S3, Google Cloud Storage, Azure).
*   **Hệ điều hành/Container:** Chạy trên **Alpine Linux** để tối ưu dung lượng Docker image.

### 2. Tư duy Kiến trúc (Architectural Thinking)

DocuSeal áp dụng kiến trúc **Monolithic hiện đại** với các đặc điểm:

*   **Hybrid Rendering:**
    *   Sử dụng Server-side Rendering (Rails) cho Dashboard và Settings để SEO tốt và load nhanh.
    *   Sử dụng Client-side Rendering (Vue.js) cho trình soạn thảo tài liệu (WYSIWYG) để đảm bảo trải nghiệm mượt mà không cần load lại trang.
*   **Multi-tenancy & Testing Mode:** Hệ thống hỗ trợ chế độ đa người dùng và đặc biệt là tính năng "Testing Account" (impersonation) giúp người quản trị giả lập môi trường của khách hàng mà không cần mật khẩu.
*   **Security-First:** 
    *   Mọi thông tin cấu hình nhạy cảm (SMTP, API Keys) được lưu trong bảng `encrypted_configs` và mã hóa ở mức database.
    *   Tài liệu được bảo vệ bằng cơ chế **Signed URL** (thông qua `signed_id_verifier`) có thời gian hết hạn (TTL) để tránh rò rỉ link.
*   **Background Processing:** Mọi tác vụ nặng (tạo PDF, gửi Webhook, gửi Email, reindex tìm kiếm) đều được đưa vào hàng đợi **Sidekiq** (Redis) để đảm bảo không làm nghẽn UI.

### 3. Kỹ thuật lập trình chính (Key Techniques)

*   **Xử lý chữ ký số (eSignature):** Thay vì chỉ chèn ảnh vào PDF, DocuSeal sử dụng `HexaPDF` để tạo các trường chữ ký kỹ thuật số thực thụ, hỗ trợ cả chứng chỉ AATL (Adobe Approved Trust List).
*   **Logic điều kiện và công thức (Conditional Logic & Formulas):** Cho phép ẩn/hiện các trường dựa trên giá trị của trường khác hoặc tính toán con số ngay trong form (sử dụng Vue.js xử lý reactive trên frontend).
*   **Webhook & Event System:** Một hệ thống logging sự kiện (`SubmissionEvents`) rất chi tiết, ghi lại từ lúc người dùng mở mail, xem form cho đến khi ký xong. Dữ liệu này được dùng để gửi Webhook đến hệ thống bên thứ ba.
*   **Full-text Search:** Tích hợp tính năng tìm kiếm toàn văn ngay cả trên dữ liệu người dùng nhập vào form (`SearchEntry`), giúp quản lý hàng vạn bản ký kết dễ dàng.
*   **Rate Limiting:** Triển khai cơ chế giới hạn tần suất gửi email/sms dựa trên IP và Submitter ID để chống spam (File `lib/rate_limit.rb`).

### 4. Luồng hoạt động hệ thống (System Workflow)

#### Bước 1: Tạo Template (Trình thiết kế)
1.  Người dùng tải lên file PDF/Docx.
2.  Backend gọi `pdfium` để tạo ảnh preview cho từng trang.
3.  Model AI (ONNX) quét tài liệu để gợi ý vị trí các trường.
4.  Người dùng kéo thả thêm các loại trường (Chữ ký, Ngày tháng, Checkbox). Cấu hình này được lưu dưới dạng JSON trong bảng `templates`.

#### Bước 2: Gửi yêu cầu ký (Submission)
1.  Hệ thống tạo bản ghi `Submission` và các `Submitter`.
2.  Nếu cấu hình "Preserved Order", hệ thống sẽ gửi lời mời cho người thứ nhất, đợi người đó ký xong mới gửi cho người thứ hai.
3.  Email/SMS được gửi đi thông qua background job.

#### Bước 3: Người nhận ký tên
1.  Người nhận mở link (Slug an toàn).
2.  Xác thực 2FA (nếu có).
3.  Vue.js form hiển thị PDF preview và các lớp overlay để người dùng nhập liệu/vẽ chữ ký.
4.  Khi nhấn Submit, dữ liệu được gửi về API, hệ thống kiểm tra tính hợp lệ (Validation).

#### Bước 4: Hoàn tất và Đóng dấu
1.  Sau khi người cuối cùng ký, `Submissions::EnsureResultGenerated` được kích hoạt.
2.  Hệ thống gộp tất cả chữ ký, thông tin nhập liệu và **Audit Trail** (nhật ký hành trình) vào file PDF cuối cùng.
3.  Gửi bản sao PDF cho các bên và kích hoạt Webhook "submission.completed".

### Tổng kết
DocuSeal là một ví dụ điển hình về việc sử dụng **Ruby on Rails** để quản lý logic nghiệp vụ phức tạp một cách gọn gàng, kết hợp với **Vue.js** cho các tính năng tương tác cao và **AI/C-libraries** cho các tác vụ xử lý tài liệu chuyên sâu. Cấu trúc mã nguồn rất chặt chẽ, chú trọng vào tính ổn định và khả năng mở rộng thông qua API/Webhook.