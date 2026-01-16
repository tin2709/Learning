Dưới đây là phân tích chi tiết về dự án **Signature PDF** (một phần mềm nguồn mở cho phép ký, tổ chức và nén PDF) dựa trên mã nguồn bạn đã cung cấp:

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án sử dụng mô hình kết hợp giữa sức mạnh xử lý của các công cụ dòng lệnh (CLI) phía máy chủ và khả năng tương tác của trình duyệt:

*   **Backend:** PHP sử dụng micro-framework **Fat-Free (F3)**. Đây là lựa chọn giúp hệ thống nhẹ, xử lý định tuyến (routing) nhanh chóng mà không cần quá nhiều tài nguyên.
*   **Frontend:** 
    *   **Fabric.js:** Thư viện chính để tạo lớp Canvas phía trên PDF, cho phép người dùng vẽ, viết văn bản hoặc chèn hình ảnh chữ ký.
    *   **PDF.js (Mozilla):** Dùng để hiển thị nội dung file PDF trực tiếp trên trình duyệt mà không cần plugin ngoài.
    *   **Bootstrap:** Framework CSS để xây dựng giao diện người dùng (UI) tương thích đa thiết bị.
*   **Công cụ hệ thống (CLI Tools - Trái tim của dự án):**
    *   **PDFtk:** Công cụ chủ đạo để thực hiện các thao tác "nặng" như gộp file (`merge`), xoay trang (`rotate`), và đặc biệt là kỹ thuật `multistamp` (chồng lớp chữ ký lên file PDF gốc).
    *   **Ghostscript (gs):** Xử lý nén và giảm dung lượng PDF với các profile chất lượng khác nhau.
    *   **ImageMagick & Potrace:** Chuyển đổi chữ ký từ dạng ảnh bitmap (PNG/JPG) sang dạng vector (SVG) để đảm bảo độ sắc nét khi in ấn.
    *   **Poppler-utils (pdfsig):** Thực hiện việc ký số (Digital Signature) và xác thực chứng chỉ.
    *   **GnuPG (GPG):** Mã hóa các file PDF lưu trữ tạm thời trên máy chủ để bảo mật.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Engineering & Architectural Thinking)

*   **Kiến trúc "Wrapper":** Thay vì cố gắng xử lý nhị phân file PDF bằng PHP (vốn rất chậm và tốn tài nguyên), dự án đóng vai trò là một "vỏ bọc" thông minh. PHP tiếp nhận yêu cầu từ người dùng, sau đó gọi các tiến trình hệ thống (CLI) đã được tối ưu hóa bằng C/C++ để xử lý file.
*   **Xử lý phân lớp (Layering Strategy):** Khi một PDF có nhiều người ký, hệ thống không ghi đè trực tiếp. Mỗi chữ ký được lưu thành một file SVG/PDF riêng lẻ (layer). Khi cần xem file cuối cùng, máy chủ sẽ dùng lệnh `pdftk` để chồng tất cả các lớp này lên nhau. Điều này cho phép thực hiện tính năng "Ký chung" (Multi-signature) mà không làm hỏng file gốc.
*   **Security-by-Design (Bảo mật theo thiết kế):**
    *   **Encryption:** Nếu kích hoạt chế độ lưu trữ, file sẽ được mã hóa đối xứng bằng GPG với key được lưu trong Cookie người dùng hoặc Hash URL. Server không giữ key giải mã vĩnh viễn.
    *   **Privacy:** File được lưu trong thư mục tạm và có cơ chế Cron tự động xóa sau một khoảng thời gian (Retention period).
*   **Tính mở rộng (Extensibility):** Hệ thống dịch thuật (i18n) sử dụng chuẩn `Gettext` (.po/.mo), giúp dễ dàng thêm ngôn ngữ mới mà không can thiệp vào code logic.

---

### 3. Các kỹ thuật chính nổi bật (Key Technical Highlights)

*   **Vectorization (Vector hóa chữ ký):** Dự án sử dụng `Potrace` để biến các nét vẽ tự do của người dùng thành định dạng vector. Điều này giúp chữ ký không bị "vỡ hạt" dù PDF được phóng to ở bất kỳ mức độ nào.
*   **Digital Signature (NSS):** Tích hợp với Network Security Services (NSS) để hỗ trợ ký số thực thụ bằng chứng chỉ (certificates), giúp tài liệu có giá trị pháp lý cao hơn là chỉ chèn một tấm ảnh chữ ký thông thường.
*   **Client-side PDF Manipulation:** Việc xoay trang, sắp xếp thứ tự trang được xử lý ngay tại trình duyệt thông qua JS, sau đó gửi danh sách các thao tác về server để thực hiện bằng lệnh `pdftk` một lần duy nhất, giúp giảm tải băng thông.
*   **Xử lý Metadata:** Sử dụng thư viện `pdf-lib` (phía JS) và CLI để chỉnh sửa các thông tin ẩn của file như Tác giả, Tiêu đề, Từ khóa mà không làm thay đổi nội dung trang.

---

### 4. Luồng hoạt động của dự án (Project Workflow)

1.  **Giai đoạn Upload & Render:** Người dùng tải PDF lên -> Server lưu tạm -> Trình duyệt dùng `PDF.js` hiển thị PDF lên màn hình.
2.  **Giai đoạn Tương tác:** Người dùng chọn công cụ (Ký, Viết chữ, Đóng dấu) -> `Fabric.js` tạo một lớp Canvas trong suốt đè lên PDF -> Người dùng vẽ/nhập liệu trên Canvas đó.
3.  **Giai đoạn Xử lý dữ liệu:** 
    *   Khi bấm "Hoàn thành", dữ liệu từ Canvas được xuất ra định dạng SVG.
    *   Dữ liệu này được gửi lên server qua POST request.
4.  **Giai đoạn Assembly (Lắp ráp):**
    *   Server nhận SVG -> Dùng `rsvg-convert` chuyển thành một file PDF trắng có chứa chữ ký ở đúng tọa độ.
    *   Dùng lệnh `pdftk multistamp` để "dập" file PDF chữ ký này lên file PDF gốc.
    *   Nếu có yêu cầu nén, `Ghostscript` sẽ được gọi để tối ưu dung lượng.
5.  **Giai đoạn Phân phối:** Server trả về file PDF cuối cùng cho người dùng tải về hoặc tạo một liên kết chia sẻ (Shared link) được mã hóa để người khác cùng vào ký.
6.  **Giai đoạn Dọn dẹp:** Một script `cron` sẽ chạy định kỳ để kiểm tra các file hết hạn dựa trên file `.expire` và xóa bỏ chúng để giải phóng bộ nhớ và bảo mật dữ liệu.

---
**Tóm lại:** Đây là một dự án có tư duy kỹ thuật thực dụng, tận dụng tối đa các công cụ mã nguồn mở lâu đời và ổn định trên Linux để giải quyết một bài toán phức tạp về định dạng PDF trên nền tảng Web.