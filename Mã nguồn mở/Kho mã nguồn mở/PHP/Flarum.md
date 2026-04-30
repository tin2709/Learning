Dựa trên các tệp tin mã nguồn và cấu trúc của Flarum (phiên bản skeleton v2.0.0-beta), dưới đây là phân tích chi tiết về dự án này:

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Backend:** Sử dụng **PHP** làm ngôn ngữ chủ đạo. Dù là một hệ thống độc lập, Flarum tận dụng rất nhiều thành phần mạnh mẽ từ hệ sinh thái **Laravel** (như Database, Container, Filesystem) nhưng được tinh chỉnh để tối ưu hóa tốc độ.
*   **Frontend:** Một điểm đặc biệt của Flarum là không dùng React hay Vue, mà sử dụng **Mithril.js**. Đây là một framework JavaScript cực kỳ nhỏ gọn (tiny footprint) và hiệu năng cao, giúp giao diện người dùng (UI) mượt mà như một ứng dụng đơn trang (SPA).
*   **Quản lý gói (Dependency Management):** Sử dụng **Composer** cho PHP. Toàn bộ tính năng của Flarum (từ bài đăng, thẻ tag đến thông báo) đều được quản lý dưới dạng các gói mở rộng (extensions).
*   **Cấu hình máy chủ:** Cung cấp sẵn cấu hình tối ưu cho **Nginx** (`.nginx.conf`), **Apache** (`.htaccess`) và **IIS** (`web.config`), tập trung mạnh vào nén Gzip và bộ nhớ đệm (caching).

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Kiến trúc Skeleton (Bộ khung):** Repository này thực chất chỉ là một "bộ khung". Logic cốt lõi nằm ở gói `flarum/core`. Điều này cho phép người dùng cập nhật phiên bản chính mà không ảnh hưởng đến các tệp tin cấu hình riêng của họ.
*   **Triết lý "Extensible by Default" (Mở rộng là mặc định):** Flarum được thiết kế theo hướng module hóa cực cao. Trong tệp `composer.json`, bạn có thể thấy ngay cả những tính năng cơ bản như "thẻ tag" (`flarum/tags`), "thích" (`flarum/likes`) cũng là các extension.
*   **Mẫu thiết kế Extender (Extender Pattern):** Thông qua tệp `extend.php`, lập trình viên có thể can thiệp vào logic của hệ thống một cách an toàn mà không cần sửa mã nguồn lõi (core). Đây là cách Flarum giải quyết bài toán tùy biến nhưng vẫn dễ dàng cập nhật.
*   **Tách biệt thư mục công khai (Public Folder Isolation):** Chỉ thư mục `public/` được tiếp xúc với internet. Các mã nguồn quan trọng, tệp cấu hình và dữ liệu lưu trữ nằm ở cấp cao hơn, giúp ngăn chặn việc truy cập trái phép vào các tệp tin nhạy cảm.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Single Entry Point (Điểm nhập duy nhất):** Mọi yêu cầu web đều đi qua `public/index.php` và mọi lệnh CLI đều qua tệp `flarum`. Cả hai đều khởi tạo thông qua `site.php` để đảm bảo môi trường đồng nhất.
*   **H5BP Caching & Optimization:** Các tệp cấu hình máy chủ áp dụng các tiêu chuẩn từ *HTML5 Boilerplate*, thiết lập thời gian sống của cache rất dài cho tài sản tĩnh (CSS/JS là 1 năm, hình ảnh là 1 tháng) và bật nén cho mọi loại nội dung văn bản.
*   **Thực thi đồng bộ/bất đồng bộ:** Sử dụng hệ thống Queue (hàng đợi) để xử lý các tác vụ nặng (như gửi email, cập nhật thống kê) giúp phản hồi người dùng ngay lập tức.
*   **Security Protection:** Nginx và Apache được cấu hình sẵn để chặn truy cập vào các tệp tin nhạy cảm như `.git`, `composer.json`, `storage`, và `vendor`.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

**Luồng xử lý yêu cầu Web:**
1.  **Tiếp nhận:** Yêu cầu từ người dùng được máy chủ (Nginx/Apache) chuyển hướng về `public/index.php`.
2.  **Khởi tạo (Bootstrapping):** `index.php` yêu cầu `site.php`. Tại đây, hệ thống nạp trình tự nạp gói tự động (Composer Autoloader) và thiết lập các đường dẫn thư mục (base, public, storage).
3.  **Đăng ký mở rộng:** Hệ thống đọc tệp `extend.php` để nạp các tùy chỉnh của người dùng và các extension đã bật.
4.  **Xử lý:** `Flarum\Http\Server` tiếp nhận đối tượng site, phân tích route (đường dẫn), gọi Controller tương ứng và tương tác với Database.
5.  **Phản hồi:** Trả về dữ liệu JSON (cho API) hoặc render giao diện thông qua Mithril.js cho người dùng.

**Luồng CLI (Command Line):**
1.  Người dùng chạy lệnh `php flarum [command]`.
2.  Tệp `flarum` khởi tạo tương tự như Web nhưng thông qua `Flarum\Console\Server`.
3.  Thực hiện các tác vụ quản trị như di cư dữ liệu (migrate), xóa cache hoặc cài đặt extension.

**Kết luận:** Flarum là một ví dụ điển hình về việc xây dựng phần mềm hiện đại: **Cốt lõi tinh gọn - Tính năng nằm ở Extension - Hiệu suất nằm ở Frontend và Caching.**