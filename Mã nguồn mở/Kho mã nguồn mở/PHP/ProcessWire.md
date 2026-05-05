Dựa trên cấu trúc thư mục và mã nguồn của **ProcessWire 3.x**, dưới đây là phân tích chuyên sâu về hệ quản trị nội dung (CMS) và khung làm việc (CMF) độc đáo này:

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Ngôn ngữ lập trình:** **PHP (>= 7.1)**. Mã nguồn được viết theo phong cách hướng đối tượng (OOP) kết hợp với các kỹ thuật lập trình hàm (Functional Programming) thông qua API.
*   **Cơ sở dữ liệu:** **MySQL/MariaDB**. ProcessWire sử dụng một cấu trúc DB cực kỳ linh hoạt (EAV - Entity-Attribute-Value cải tiến), trong đó mỗi trường dữ liệu (Field) thường có bảng riêng, giúp tối ưu hóa việc truy vấn và mở rộng không giới hạn.
*   **Web Server:** Phụ thuộc nhiều vào **Apache** với các quy tắc `mod_rewrite` phức tạp trong file `.htaccess`.
*   **Frontend-agnostic:** Không áp đặt bất kỳ Framework CSS hay thư viện JS nào cho phía người dùng (Client-side). Tuy nhiên, giao diện quản trị (Admin UI) sử dụng **jQuery**, **UIkit**, và **Font Awesome**.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của ProcessWire dựa trên một triết lý cốt lõi: **"Everything is a Page"** (Mọi thứ đều là một Trang).

*   **Cấu trúc cây phân cấp:** Khác với WordPress (phân chia Post, Page, Category), trong ProcessWire, tất cả thực thể (Người dùng, Quyền hạn, Vai trò, Cài đặt hệ thống, Nội dung) đều là các "Page" nằm trong một cây phân cấp duy nhất.
*   **Tách biệt Core và Site:**
    *   `/wire`: Chứa nhân hệ thống. Thư mục này không bao giờ được sửa đổi bởi người dùng để đảm bảo việc nâng cấp dễ dàng (chỉ cần ghi đè thư mục này).
    *   `/site`: Chứa toàn bộ logic ứng dụng, cấu hình, template và module của người dùng.
*   **Template & Field System:** Một Template không chỉ là file hiển thị (View), mà còn là định nghĩa dữ liệu (Model). Template liên kết với một nhóm các trường (Fieldgroup). Khi bạn tạo một Page, bạn chọn một Template, và Page đó sẽ có các trường dữ liệu tương ứng.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Hook System (AOP - Aspect Oriented Programming):** Đây là "vũ khí" mạnh nhất của ProcessWire. Bất kỳ phương thức nào trong lõi có tiền tố `___` (ví dụ: `___save`) đều có thể được "móc" vào (Hook) từ bên ngoài (trước, sau hoặc thay thế hoàn toàn) mà không cần sửa code lõi.
*   **Selectors (jQuery-style API):** Hệ thống truy vấn dữ liệu sử dụng cú pháp chuỗi tương tự jQuery selector.
    *   Ví dụ: `$pages->find("template=blog-post, tags=technology, limit=10, sort=-created");`
    *   Kỹ thuật này giúp lập trình viên thao tác với DB mà không cần viết SQL phức tạp.
*   **File Compiler:** Lớp `/wire/core/FileCompiler.php` cho phép ProcessWire tự động biên dịch các file PHP để hỗ trợ tính năng Namespace (vốn khó khăn trong các phiên bản cũ) và giúp code của các Module tương thích với nhau một cách minh bạch.
*   **Markup Regions:** Một kỹ thuật render phía server cho phép các file template con "ghi đè" hoặc "đẩy" nội dung vào các khu vực cụ thể trong file layout chính (`_main.php`) dựa trên ID của tag HTML.

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Giai đoạn Bootstrapping:
1.  **Request:** Client gửi yêu cầu đến `index.php`.
2.  **Bootstrap:** `index.php` nạp `/wire/core/ProcessWire.php`.
3.  **Config:** Hệ thống đọc `/site/config.php` để thiết lập môi trường, kết nối DB.
4.  **Autoload:** `WireClassLoader` nạp các lớp cần thiết.

#### B. Giai đoạn Initialization:
5.  **Init:** Các module có tính năng "Autoload" được khởi tạo. File `/site/init.php` được thực thi (nơi lý tưởng để đặt các Hook sớm).
6.  **Ready:** File `/site/ready.php` được gọi khi toàn bộ API hệ thống đã sẵn sàng và trang hiện tại đã được xác định.

#### C. Giai đoạn Dispatching & Rendering:
7.  **URL Mapping:** Module `ProcessPageView` phân tích URL và tìm Page tương ứng trong DB.
8.  **Access Control:** Kiểm tra quyền truy cập của người dùng đối với Page đó.
9.  **Template Execution:** PHP thực thi file template liên kết với Page (ví dụ: `basic-page.php`). 
    *   Nếu có `$config->prependTemplateFile`, nó sẽ nạp file init (`_init.php`) trước.
    *   Nội dung được xử lý và cuối cùng nạp file layout (`_main.php`) nếu sử dụng tính năng Append.
10. **Output:** Kết quả HTML được gửi về trình duyệt.

#### D. Giai đoạn Cleanup:
11. **Finished:** File `/site/finished.php` thực thi sau khi output đã được gửi, dùng cho các tác vụ dọn dẹp hoặc log.

### Tổng kết
ProcessWire không đi theo hướng "kéo và thả" cho người dùng phổ thông, mà tập trung vào **lập trình viên chuyên nghiệp**. Kiến trúc của nó cực kỳ nhất quán, linh hoạt và coi trọng sức mạnh của API, cho phép xây dựng từ blog đơn giản đến các hệ thống quản lý dữ liệu cực kỳ phức tạp.