Dựa trên mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về dự án **Maloja** - một hệ thống cơ sở dữ liệu scrobble nhạc tự lưu trữ (self-hosted).

---

### 1. Công Nghệ Cốt Lõi (Core Technology)

Dự án Maloja được xây dựng trên một ngăn xếp công nghệ tập trung vào sự nhẹ nhàng, hiệu quả và khả năng tương thích cao:

*   **Ngôn ngữ lập trình:** **Python (3.12+)** là ngôn ngữ chính cho backend.
*   **Web Framework:** Sử dụng **Bottle**, một WSGI micro-framework cực kỳ nhẹ, kết hợp với **Waitress** làm server sản phẩm.
*   **Cơ sở dữ liệu:** **SQLAlchemy 2.0** (ORM) kết hợp với **SQLite**. SQLite được chọn vì tính di động, dễ sao lưu và phù hợp với mô hình cá nhân (self-hosted).
*   **Template Engine:** **Jinja2** được dùng để tạo giao diện người dùng động từ phía server.
*   **Containerization:** Sử dụng **Docker** (dựa trên image của **linuxserver/alpine**) và **S6-overlay** để quản lý tiến trình bên trong container.
*   **Thư viện xử lý ảnh:** **pyvips** (tùy chọn) để xử lý thumbnail và **python-magic** để xác định loại tệp.
*   **Giao diện:** Sử dụng thuần **JavaScript (Vanilla JS)** cho các tính năng tương tác như tìm kiếm, upload ảnh và thông báo, tránh phụ thuộc vào các framework nặng như React hay Vue.

---

### 2. Tư Duy Kiến Trúc (Architectural Thinking)

Kiến trúc của Maloja được định hướng theo các nguyên tắc:

*   **Quyền sở hữu dữ liệu (Data Ownership):** Thiết kế để chạy cục bộ. Dữ liệu không bị phụ thuộc vào các dịch vụ cloud lớn. Thư viện nhạc được quản lý theo quy tắc riêng của người dùng.
*   **Tính tương thích ngược (Backward Compatibility):** Maloja không chỉ có API riêng mà còn giả lập các API tiêu chuẩn cũ như **GNU FM, ListenBrainz, và Audioscrobbler (v1.2 & v2.0)**. Điều này cho phép các ứng dụng nghe nhạc cũ vẫn có thể scrobble vào Maloja mà không cần thay đổi code.
*   **Kiến trúc hướng Module:**
    *   **APIs:** Chia tách giữa Native API (v1) và các Third-party Proxy APIs.
    *   **Cleanup Logic:** Tách biệt phần xử lý chuẩn hóa dữ liệu (Artist, Title) ra khỏi logic lưu trữ.
    *   **Third-party Services:** Các dịch vụ lấy metadata (Spotify, Last.fm, MusicBrainz) được thiết kế theo dạng Interface, dễ dàng mở rộng hoặc thay thế.
*   **Ưu tiên hiệu năng (Performance-First):** Sử dụng cơ chế cache nhiều lớp (Global Cache, Request-local Cache, Image Cache) để giảm tải cho SQLite khi xử lý các thống kê phức tạp.

---

### 3. Các Kỹ Thuật Chính (Key Techniques)

*   **CleanerAgent (Bộ lọc dữ liệu):** Sử dụng các tệp `.tsv` để định nghĩa quy tắc (rules). Kỹ thuật này cho phép người dùng cấu hình cách gộp artist (ví dụ: "Artist A feat. B" -> "Artist A"), sửa lỗi chính tả, hoặc gộp các dự án solo vào nhóm chính mà không làm hỏng dữ liệu gốc.
*   **Cơ chế Caching thông minh:**
    *   **LRU Cache (Least Recently Used):** Lưu trữ các kết quả truy vấn thống kê nặng trong RAM.
    *   **Image Proxying:** Thay vì link trực tiếp đến Spotify/Last.fm, Maloja tải ảnh về, lưu trữ cục bộ và phục vụ (serve) lại để bảo vệ quyền riêng tư và tăng tốc độ tải trang.
*   **Xử lý bất đồng bộ (Asynchronous logic):** Việc lấy ảnh (resolve images) từ bên thứ ba được thực hiện thông qua `ThreadPoolExecutor` để không chặn luồng xử lý chính của người dùng.
*   **Search Debouncing:** Kỹ thuật trì hoãn việc gửi yêu cầu tìm kiếm khi người dùng đang nhập liệu (trong file `search.js`), giúp giảm tải cho server.

---

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

Luồng đi của một bản ghi scrobble diễn ra như sau:

1.  **Tiếp nhận (Ingestion):** Một Client (Extension trình duyệt hoặc ứng dụng nghe nhạc) gửi yêu cầu POST đến một trong các endpoint API (`/newscrobble`, `/apis/listenbrainz`, v.v.).
2.  **Định danh (Identification):** Hệ thống kiểm tra `API Key` trong `apikeys.yml`. Nếu hợp lệ, nó sẽ xác định client nào đang gửi.
3.  **Chuẩn hóa (Cleanup):** Dữ liệu thô đi qua `cleanup.py`. Tại đây:
    *   Tách nghệ sĩ (ví dụ: dấu `;`, `/`, `feat`).
    *   Áp dụng các quy tắc tùy chỉnh từ người dùng (Custom Rules).
    *   Sửa lỗi tiêu đề bài hát (ví dụ: xóa "(Remastered)").
4.  **Lưu trữ (Persistence):** `sqldb.py` thực hiện ghi dữ liệu vào SQLite. Nếu bài hát hoặc nghệ sĩ chưa tồn tại, nó sẽ tạo mới. Hệ thống cũng tự động liên kết bài hát với Album dựa trên logic "Majority" hoặc "First-seen".
5.  **Phát tán (Proxying - Tùy chọn):** Nếu cấu hình, Maloja sẽ forward bản ghi này lên Last.fm.
6.  **Cập nhật Cache:** Các cache liên quan đến thời gian của scrobble đó bị xóa (invalidate) để đảm bảo thống kê mới nhất.
7.  **Hiển thị (Visualization):** Khi người dùng truy cập trang chủ, Jinja2 sẽ lấy dữ liệu từ DB (hoặc Cache), render ra HTML và hiển thị các biểu đồ Pulse, Charts, Performance.

---

### Tóm tắt đặc điểm nổi bật
Maloja không chỉ là một ứng dụng lưu trữ, nó là một **Data Aggregator**. Nó có khả năng "biến dữ liệu bẩn thành dữ liệu sạch" thông qua hệ thống quy tắc mạnh mẽ, điều mà các dịch vụ như Last.fm làm rất kém. Việc hỗ trợ cả Docker và quản lý quyền người dùng (PUID/PGID) cho thấy dự án rất chú trọng đến trải nghiệm triển khai thực tế trên các hệ thống NAS hoặc Homelab.