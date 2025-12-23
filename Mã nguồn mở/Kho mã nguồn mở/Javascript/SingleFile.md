Dựa trên các tệp mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chuyên sâu về công nghệ, kiến trúc và quy trình hoạt động của dự án **SingleFile**.

---

### 1. Công nghệ cốt lõi (Core Technologies)

*   **WebExtensions API:** Sử dụng chuẩn API mở rộng trình duyệt (tương thích Chrome, Firefox, Safari, Edge). Dự án sử dụng `manifest.json` phiên bản 2 (với các nỗ lực chuyển đổi sang v3 thông qua các nhánh khác).
*   **DOM Serialization:** Kỹ thuật then chốt để chuyển đổi trạng thái thực tế của cây DOM hiện tại thành một chuỗi văn bản HTML tĩnh.
*   **Resource Inlining (Nhúng tài nguyên):** Sử dụng **Data URIs (Base64)** để nhúng trực tiếp hình ảnh, font chữ, và các tệp âm thanh/video vào trong một tệp HTML duy nhất.
*   **Rollup.js:** Công cụ đóng gói (bundler) chính được sử dụng để tối ưu hóa mã nguồn, hỗ trợ nhiều định dạng đầu ra (UMD, IIFE, ES modules) cho các môi trường khác nhau (Content Script, Background, Worker).
*   **JavaScript hiện đại (ES2025):** Sử dụng các tính năng mới nhất của JS và công cụ ESLint để đảm bảo chất lượng code.

### 2. Tư duy kiến trúc (Architectural Design)

Kiến trúc của SingleFile được thiết kế theo mô hình **Module hóa và Đa tầng (Layered Architecture)**:

*   **Tách biệt Core và UI:** Logic xử lý lưu trang được đóng gói trong `single-file-core` (một dependency riêng), trong khi phần mở rộng trình duyệt (`src/`) chỉ tập trung vào giao diện, quản lý tab và tích hợp hệ thống.
*   **Kiến trúc Đa luồng (Background vs. Content Scripts):**
    *   **Content Scripts:** Chạy trong ngữ cảnh trang web để trích xuất DOM, bắt sự kiện scroll (để kích hoạt lazy-load) và thực hiện các thay đổi tạm thời.
    *   **Background Scripts:** Đóng vai trò "nhà điều phối" (orchestrator), quản lý quyền lưu tệp, giao tiếp với các API đám mây (Google Drive, GitHub, S3) và xử lý hàng đợi lưu trữ.
*   **Tính trừu tượng hóa tài nguyên (Resource Abstraction):** SingleFile không chỉ lưu HTML; nó xây dựng một "cây khung" (Frame tree) để quản lý các iframe lồng nhau, đảm bảo tính toàn vẹn của trang web phức tạp.
*   **Khả năng mở rộng (Extensibility):** Hỗ trợ nhiều "Destinations" (điểm đến) khác nhau thông qua các module riêng biệt trong `src/lib/` (Dropbox, GitHub, WebDAV, và cả giao thức mới như MCP - Model Context Protocol).

### 3. Các kỹ thuật chính (Key Techniques)

*   **Xử lý tài nguyên trì hoãn (Lazy-loading Handling):** SingleFile tự động giả lập sự kiện cuộn trang (`dispatch "scroll" event`) và thay đổi kích thước viewport để buộc trình duyệt tải các hình ảnh/nội dung chưa hiển thị trước khi bắt đầu lưu.
*   **Tối ưu hóa dung lượng:**
    *   **CSS Minification:** Sử dụng các thư viện như `UglifyCSS` và `csstree` để loại bỏ các rule CSS không sử dụng (Unused styles).
    *   **HTML Compression:** Loại bỏ các thẻ script và các thành phần ẩn để giảm dung lượng file cuối cùng.
*   **Định dạng Polyglot (Self-extracting ZIP):** Một kỹ thuật cực kỳ thông minh cho phép tệp lưu trữ vừa là một tệp HTML hợp lệ để xem ngay, vừa là một tệp ZIP chứa tài nguyên gốc (không bị encode base64) để giải nén khi cần.
*   **Cơ chế Hooking:** Cho phép thực thi các script người dùng (user scripts) trước và sau khi lưu để tùy chỉnh nội dung trang (ví dụ: xóa quảng cáo).
*   **Proof of Existence:** Tích hợp với blockchain (thông qua Woleet) để tạo mã băm (SHA256), chứng minh nội dung trang web tồn tại vào một thời điểm cụ thể.

### 4. Tóm tắt luồng hoạt động (Operational Workflow)

1.  **Kích hoạt (Trigger):** Người dùng nhấn nút extension hoặc hệ thống Auto-save được kích hoạt.
2.  **Chuẩn bị (Preprocessing):**
    *   Tiêm (Inject) script vào trang web.
    *   Kích hoạt lazy-load nội dung.
    *   Xử lý các Shadow DOM và Iframe.
3.  **Thu thập (Collection):** Xây dựng danh sách tất cả tài nguyên cần thiết (CSS, Images, Fonts).
4.  **Xử lý & Nhúng (Processing & Inlining):**
    *   Tải các tài nguyên còn thiếu.
    *   Chuyển đổi tài nguyên sang Base64 hoặc chuẩn bị cho cấu trúc ZIP.
    *   Lọc bỏ CSS/HTML dư thừa theo cấu hình người dùng.
5.  **Tuần tự hóa (Serialization):** Hợp nhất tất cả thành một chuỗi HTML duy nhất. Nếu người dùng chọn ghi chú (Annotate), một trình biên tập (Editor) sẽ được mở ra trước bước này.
6.  **Xuất bản (Output):**
    *   Gửi file đến API Download của trình duyệt.
    *   Hoặc tải lên các dịch vụ đám mây (GitHub, GDrive, v.v.).
    *   Hiển thị thông báo hoàn tất trên thanh InfoBar.

### Tổng kết
SingleFile không đơn thuần là công cụ "Save Page As" của trình duyệt. Nó là một **cỗ máy đóng gói web** phức tạp, ưu tiên quyền riêng tư (xử lý local 100%) và tính trung thực của nội dung (faithful copy), biến một trang web động thành một tài liệu lưu trữ tĩnh bền vững.