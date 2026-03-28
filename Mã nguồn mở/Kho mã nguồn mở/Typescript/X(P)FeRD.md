Dưới đây là bản phân tích chi tiết về dự án **X(P)FeRD**, một ứng dụng chuyên biệt để quản lý và tạo hóa đơn điện tử theo tiêu chuẩn Đức (XRechnung và ZUGFeRD).

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án sử dụng một tập hợp công nghệ hiện đại, tập trung vào tính gọn nhẹ và hiệu suất cao:

*   **Frontend: Svelte 5 (Runes):** Sử dụng phiên bản mới nhất của Svelte với hệ thống phản hồi (reactivity) dựa trên Runes (`$state`, `$derived`, `$effect`). Điều này giúp mã nguồn giao diện cực kỳ ngắn gọn và hiệu quả.
*   **Backend: Node.js & Express:** Framework tiêu chuẩn để xây dựng RESTful API.
*   **Database: SQLite (better-sqlite3):** Lựa chọn tối ưu cho một ứng dụng tự host (self-hosted). Nó không cần server database riêng biệt, dữ liệu được lưu trong một file duy nhất, rất dễ sao lưu và di chuyển qua Docker.
*   **Xử lý văn bản chuyên biệt:**
    *   **@libpdf/core:** Thư viện dùng để dựng PDF từ các khối dữ liệu tùy chỉnh.
    *   **xmlbuilder2:** Dùng để xây dựng cấu trúc XML phức tạp theo chuẩn UBL 2.1 (XRechnung 3.0).
    *   **Zod:** Sử dụng xuyên suốt để kiểm soát kiểu dữ liệu và validate đầu vào API.
*   **Build Tooling:** **esbuild** được sử dụng thay cho Vite/Webpack để đóng gói Frontend với tốc độ cực nhanh.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của X(P)FeRD được thiết kế theo mô hình **Service-Oriented** kết hợp với **Shared Logic**:

*   **Cấu trúc Shared-first:** Thư mục `src/shared` chứa các định nghĩa về Types, hằng số (Code lists của chuẩn hóa đơn) và logic validation. Điều này đảm bảo Frontend và Backend luôn đồng bộ về quy tắc nghiệp vụ.
*   **Tách biệt nghiệp vụ (Decoupling):** 
    *   **Controllers:** Chỉ làm nhiệm vụ điều hướng và nhận/trả dữ liệu.
    *   **Services:** Chứa logic nghiệp vụ "nặng" như `XRechnungXmlService` (tạo XML) hay `ZUGFeRDService` (nhúng XML vào PDF).
    *   **Models:** Quản lý giao tiếp trực tiếp với SQLite.
*   **WYSIWYG-driven Design:** Kiến trúc cho phép lưu trữ các "khối" (Blocks) PDF dưới dạng tọa độ (x, y, width, height) trong DB, sau đó render lại tương ứng trên cả Canvas (Preview) và PDF thực tế.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Nhúng dữ liệu ZUGFeRD:** Một kỹ thuật khó là tạo ra file PDF/A-3. Ứng dụng thực hiện việc này bằng cách nhúng tệp XML hóa đơn vào bên trong file PDF dưới tên `factur-x.xml`, cho phép máy tính đọc được dữ liệu hóa đơn trong khi con người vẫn xem được tệp PDF thông thường.
*   **Xử lý Font chữ tùy chỉnh:** Ứng dụng có kỹ thuật trích xuất tên Font trực tiếp từ file binary (TTF/OTF) bằng cách parse bảng `name` của font, sau đó chuyển đổi sang Base64 để nhúng vào `@font-face` CSS động.
*   **Reactivity Store trong Svelte 5:** Sử dụng tệp `.svelte.ts` để tạo các store lưu trữ cài đặt người dùng (ngôn ngữ, định dạng ngày tháng) mang tính toàn cục và phản hồi ngay lập tức trên toàn bộ UI.
*   **Hệ thống Validation phân cấp:** Sử dụng Zod để validate từ những trường nhỏ nhất (IBAN, Email) cho đến toàn bộ cấu trúc hóa đơn phức tạp với nhiều ràng buộc pháp lý.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng xử lý chính của ứng dụng diễn ra như sau:

1.  **Giai đoạn Nhập liệu:** Người dùng có thể nhập thủ công qua Form hoặc **Import** một file XML XRechnung/UBL có sẵn. Hệ thống sẽ parse XML và chuyển đổi ngược lại thành Object hóa đơn trong ứng dụng.
2.  **Giai đoạn Thiết kế (Designer):** Người dùng chọn một `PdfTemplate`. Designer cho phép kéo thả các thành phần (Logo, bảng thông tin, số trang). Tọa độ này được lưu vào SQLite.
3.  **Giai đoạn Tính toán:** Khi lưu hóa đơn, Backend tự động tính toán lại các giá trị Net, Tax và Gross dựa trên danh sách các dòng hàng (Line Items) để đảm bảo độ chính xác tuyệt đối trước khi xuất.
4.  **Giai đoạn Xuất bản (Export):** 
    *   **XML:** `XRechnungXmlService` tạo file XML chuẩn.
    *   **PDF:** `PdfRenderService` lấy dữ liệu hóa đơn nhồi vào Template đã chọn.
    *   **Hybrid:** Kết hợp cả 2 để tạo file ZUGFeRD (PDF + XML nhúng).

---

### Tổng kết
**X(P)FeRD** là một dự án có độ hoàn thiện kỹ thuật cao trong lĩnh vực Fintech/E-invoicing. Điểm mạnh nhất của nó là khả năng đơn giản hóa các chuẩn nghiệp vụ cực kỳ phức tạp (XRechnung) thành một giao diện kéo thả trực quan, đồng thời đảm bảo tính tương thích tuyệt đối với các hệ thống kế toán tại Đức.