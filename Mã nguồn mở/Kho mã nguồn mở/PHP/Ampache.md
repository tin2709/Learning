Dưới đây là bản phân tích chi tiết về dự án **Ampache** dựa trên cấu trúc thư mục, mã nguồn và tài liệu đi kèm mà bạn cung cấp.

---

# Phân tích Dự án Ampache: Công nghệ, Kiến trúc và Quy trình

## 1. Công nghệ cốt lõi (Core Technology Stack)

Ampache là một ứng dụng web lâu đời nhưng đang được hiện đại hóa mạnh mẽ (phiên bản 7.x). Các công nghệ chính bao gồm:

*   **Backend:** **PHP 8.2+** (Yêu cầu PHP 8.4 cho Ampache 7.1). Sử dụng **Composer** để quản lý thư viện.
*   **Database:** **MySQL** hoặc **MariaDB**. Hệ thống sử dụng cơ chế Migration mạnh mẽ để quản lý phiên bản DB.
*   **Frontend:** Đang chuyển đổi từ jQuery/Legacy sang **Vite + Modern JavaScript (ES6+)**. Sử dụng **NPM** để quản lý package JS.
*   **Containerization:** Hỗ trợ **Docker** và **Docker-Compose** (với các Dockerfile riêng cho từng phiên bản PHP).
*   **Streaming & Media:** Dựa vào **FFmpeg**, **Lame**, **Flac**... để xử lý và chuyển mã (transcoding) âm thanh/video ngay lập tức.
*   **Giao thức hỗ trợ:** API (v3 đến v6), **Subsonic API**, **UPnP**, **WebDAV**, **DAAP**.

## 2. Kỹ thuật và Tư duy Kiến trúc (Architectural Thinking)

Dự án đang trong quá trình chuyển đổi từ mã nguồn "mì ăn liền" (Legacy) sang kiến trúc hiện đại, sạch sẽ hơn:

*   **Kiến trúc hướng Tên miền (Domain-Driven Design - DDD):** Trong thư mục `src/Module`, mã nguồn được chia theo các domain nghiệp vụ như `Album`, `Artist`, `Playback`, `Podcast`, `User`. Mỗi domain tự quản lý logic của riêng mình.
*   **Repository Pattern:** Toàn bộ việc tương tác với cơ sở dữ liệu được tách biệt vào `src/Repository`. Điều này giúp cô lập logic truy vấn và dễ dàng viết Unit Test (sử dụng PHPUnit và Mockery).
*   **Factory & Adapter Pattern:**
    *   **GuiFactory/TalFactory:** Dùng để tạo ra các giao diện người dùng khác nhau.
    *   **ViewAdapter:** Chuyển đổi dữ liệu model sang định dạng hiển thị.
    *   **Catalog_beets/dropbox/subsonic:** Các Adapter giúp Ampache kết nối với nhiều nguồn dữ liệu khác nhau.
*   **Dependency Injection (DI):** Sử dụng `php-di/php-di` để quản lý sự phụ thuộc giữa các lớp, giúp code linh hoạt và dễ bảo trì hơn.
*   **Hệ thống Plugin:** Ampache có kiến trúc plugin rất mạnh (thư mục `src/Plugin`). Các plugin có thể can thiệp vào quy trình tìm lời bài hát (lyrics), ảnh đại diện (avatar), hoặc thống kê (scrobbling).

## 3. Các kỹ thuật chính nổi bật (Key Techniques)

1.  **Chuyển mã linh hoạt (On-the-fly Transcoding):** Kỹ thuật quan trọng nhất của Ampache. Nó cho phép người dùng nghe nhạc ở định dạng FLAC trên các thiết bị chỉ hỗ trợ MP3 bằng cách chuyển đổi dữ liệu ngay khi đang stream.
2.  **Quản lý Metadata (getID3):** Tích hợp thư viện `getID3` để trích xuất thông tin từ file media (tag, bitrate, độ dài) và đồng bộ vào DB.
3.  **Hệ thống ACL (Access Control Lists):** Kiểm soát truy cập dựa trên IP và loại dịch vụ (Interface, Streaming, RPC, Local Network). Đây là kỹ thuật bảo mật cốt lõi để bảo vệ tài nguyên media.
4.  **Static Analysis & Quality Control:** Sử dụng **PHPStan** (Level 8), **Rector** (để tự động nâng cấp code lên PHP mới), và **PHP-CS-Fixer** để đảm bảo chất lượng code đồng nhất trên 126 contributors.
5.  **Smart Playlists:** Kỹ thuật tìm kiếm nâng cao (`advanced_search`) cho phép tạo playlist dựa trên các quy tắc logic (ví dụ: "Nhạc Rock, phát trên 10 lần, chưa nghe trong 1 tháng").

## 4. Tóm tắt luồng hoạt động của Project (Workflow)

Quy trình hoạt động của Ampache có thể tóm tắt qua 5 bước chính:

1.  **Khởi tạo & Quản lý Catalog:**
    *   Admin cấu hình đường dẫn thư mục media (Local, Dropbox, Subsonic...).
    *   Hệ thống quét (Scan) file, sử dụng `getID3` đọc tag và lưu vào database thông qua `Repository`.
    *   Quy trình này có thể chạy tự động qua Cron hoặc Inotify (theo dõi thay đổi file thời gian thực).

2.  **Xử lý Giao diện & Yêu cầu:**
    *   Khi người dùng truy cập, `src/Config/Init.php` sẽ kiểm tra môi trường, cấu hình và xác thực người dùng.
    *   `src/Gui` và hệ thống Template sẽ render dữ liệu dựa trên các `ViewAdapter`.

3.  **Xử lý API & Kết nối thiết bị:**
    *   Nếu người dùng sử dụng ứng dụng di động (như Power Ampache), yêu cầu sẽ đi qua `src/Module/Api`.
    *   Hệ thống xác thực qua `Handshake` (Token-based) và trả về dữ liệu JSON/XML.

4.  **Quy trình Streaming (Phát nhạc):**
    *   Khi người dùng nhấn "Play", Ampache kiểm tra quyền (ACL).
    *   Nếu cần chuyển mã, hệ thống gọi FFmpeg để stream dữ liệu qua bộ đệm.
    *   Media được trả về client thông qua `play/index.php`.

5.  **Ghi nhận & Thống kê:**
    *   Trong khi phát, hệ thống ghi nhận lượt nghe (Scrobbling) lên database nội bộ hoặc các dịch vụ ngoại vi như Last.fm thông qua hệ thống Plugin.

---

**Kết luận:** Ampache là một dự án có **chiều sâu kiến trúc kinh điển**. Dù có tuổi đời lâu nhưng cách họ quản lý migration (hàng trăm file migration từ v3 đến v7) và việc áp dụng Static Analysis cho thấy đây là một dự án cực kỳ ổn định và đáng tin cậy cho việc quản lý thư viện media cá nhân quy mô lớn.