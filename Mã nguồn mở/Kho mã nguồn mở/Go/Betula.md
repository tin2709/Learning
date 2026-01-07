Dựa trên mã nguồn và tài liệu bạn cung cấp cho dự án **Betula**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống:

---

### 1. Công nghệ cốt lõi (Core Technology)

*   **Ngôn ngữ lập trình:** **Go (Golang)** - chiếm hơn 90% dự án. Tận dụng tính hiệu quả của Go trong việc tạo ra một file thực thi duy nhất (single binary).
*   **Cơ sở dữ liệu:** **SQLite** - Toàn bộ dữ liệu được lưu trữ trong một file `.betula`. Sử dụng thư viện `mattn/go-sqlite3`.
*   **Giao thức Federation:** **ActivityPub** - Cho phép kết nối với mạng lưới Fediverse (như Mastodon). Dự án tự xây dựng (homebrew) các thực thi cho ActivityPub, WebFinger và chữ ký HTTP.
*   **Định dạng nội dung:** **Mycomarkup** - Một ngôn ngữ đánh dấu tùy chỉnh (thay vì Markdown) được phát triển bởi cùng tác giả để định dạng mô tả bookmark.
*   **Lưu trữ (Archiving):** **Obelisk-ng** - Thư viện dùng để tạo bản sao lưu cục bộ của các trang web (lưu dưới dạng blob trong SQLite).
*   **Frontend:** **Server-side Rendering (SSR)** - Sử dụng `html/template` của Go. Giao diện tối giản, ưu tiên hoạt động không cần JavaScript.

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Engineering & Architectural Thinking)

*   **Kiến trúc Ports & Adapters (Hexagonal Architecture):**
    *   Dự án phân tách rõ ràng giữa các interface (`ports/`) và các thực thi cụ thể (`gateways/`, `db/`).
    *   Ví dụ: `ports/liking` định nghĩa cách một hệ thống "Like" hoạt động, trong khi `db/repo_likes.go` thực thi việc lưu trữ vào DB.
*   **Quản lý Di chuyển Dữ liệu (Database Migrations):**
    *   Sử dụng một hệ thống migration tự chế (`db/migrations.go`) dựa trên các file SQL được đánh số trong thư mục `db/scripts/`. Điều này giúp nâng cấp database an toàn qua từng phiên bản phần mềm.
*   **Xử lý Bất đồng bộ (Job System):**
    *   Hệ thống `jobs/` sử dụng Go channels (`ListenAndWhisper`) để xử lý các tác vụ tốn tài nguyên như: gửi thông báo đến các server khác qua ActivityPub hoặc cào dữ liệu web để tạo bản lưu trữ (archive).
*   **Tư duy "Self-contained":**
    *   Dự án hướng tới việc cài đặt cực kỳ đơn giản: Một file binary + Một file DB. Mọi cấu hình (Port, Hostname, Title) đều lưu trong bảng `BetulaMeta` thay vì file cấu hình bên ngoài.

---

### 3. Các kỹ thuật chính nổi bật (Notable Key Techniques)

*   **HTTP Signatures:** Thực thi việc ký và xác thực các request giữa các instance trong Fediverse (dựa trên triển khai của dự án Honk) để đảm bảo bảo mật.
*   **Microformats (IndieWeb):** Gói `readpage` có khả năng phân tích HTML để tìm các class như `h-entry`, `u-bookmark-of`, giúp tự động trích xuất thông tin từ các trang web khác khi người dùng "repost".
*   **Tìm kiếm nâng cao (Hybrid Search):** Kết hợp truy vấn SQL và xử lý lọc bằng Go (gói `search/`). Hỗ trợ tìm kiếm theo từ khóa, bao gồm/loại trừ tag (`#tag`, `-#tag`) và lọc theo trạng thái repost.
*   **Xử lý Charset:** Sử dụng `golang.org/x/net/html/charset` để đảm bảo khi cào dữ liệu từ các trang web cũ (không phải UTF-8) vẫn hiển thị đúng tiêu đề và mô tả.
*   **Session Management:** Quản lý phiên đăng nhập thủ công bằng token ngẫu nhiên lưu trong SQLite, cho phép người dùng quản lý và thu hồi các phiên đăng nhập từ các thiết bị khác nhau.

---

### 4. Tóm tắt luồng hoạt động của Project (Workflow)

Dựa trên tài liệu `README.md` và mã nguồn, luồng hoạt động chính của Betula như sau:

1.  **Khởi tạo (Startup):**
    *   Chương trình kiểm tra file database SQLite. Nếu chưa có, nó sẽ khởi tạo schema từ bản migration mới nhất.
    *   Hệ thống `auth` kiểm tra xem người dùng admin đã được thiết lập chưa. Nếu chưa, ứng dụng sẽ chuyển hướng người dùng đến trang đăng ký lần đầu.
    *   Khởi động server HTTP và lắng nghe các tác vụ nền (Jobs).

2.  **Quản lý Bookmark (Local Workflow):**
    *   Người dùng lưu link (qua web UI hoặc Bookmarklet).
    *   Hệ thống tự động cào tiêu đề (nếu trống) và xử lý Mycomarkup cho mô tả.
    *   Nếu được kích hoạt, một Job sẽ được đẩy vào hàng chờ để tạo bản sao lưu (Archive) trang web đó.

3.  **Tương tác Fediverse (Federation Workflow):**
    *   **Outgoing:** Khi bạn đăng một bookmark công khai, Betula sẽ tìm danh sách những người theo dõi (Followers) và gửi hoạt động `Create{Note}` đến Inbox của họ.
    *   **Incoming:** Khi server khác gửi hoạt động (Like, Follow, Announce) đến `/inbox`, Betula sẽ xác thực chữ ký HTTP, sau đó xử lý tương ứng (ví dụ: thêm follower vào DB hoặc tạo thông báo mới).
    *   **Search:** Người dùng có thể tìm kiếm bookmark không chỉ trên máy mình mà còn yêu cầu kết quả từ các server "mutuals" (những người theo dõi lẫn nhau) thông qua giao thức Federated Search tùy chỉnh của tác giả.

4.  **Phân phối (Distribution):**
    *   Nội dung được cung cấp qua giao diện web (không cần JS), các luồng RSS feed (all posts hoặc daily digest) và tích hợp API cho các công cụ như Miniflux.

**Tóm lại:** Betula là một ví dụ điển hình về việc ứng dụng Go để xây dựng một phần mềm cá nhân bền vững, tôn trọng quyền riêng tư và có khả năng giao tiếp mạng xã hội phi tập trung mà vẫn giữ được sự đơn giản tối đa trong vận hành.