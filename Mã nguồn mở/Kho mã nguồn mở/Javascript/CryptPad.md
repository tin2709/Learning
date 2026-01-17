Dựa trên cấu trúc thư mục và nội dung các file mã nguồn của dự án **CryptPad**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống này.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

CryptPad là một bộ ứng dụng văn phòng cộng tác trực tiếp (Real-time collaboration) nhưng ưu tiên tối đa quyền riêng tư thông qua **mã hóa đầu cuối (E2EE)**.

*   **Ngôn ngữ chính:** JavaScript (Node.js ở Backend và Browser-side JS). Có sự xuất hiện của TypeScript trong các module Worker mới.
*   **Thuật toán mã hóa:** Sử dụng thư viện `TweetNaCl` (triển khai thuật toán Salsa20 và Poly1305) để mã hóa dữ liệu ngay tại trình duyệt trước khi gửi lên server.
*   **Cơ chế đồng thuận (Sync Engine):** **ChainPad** – Đây là thuật toán độc quyền của dự án dựa trên log-based để đồng bộ hóa các thay đổi giữa nhiều người dùng mà không cần server phải hiểu nội dung (Zero-Knowledge).
*   **Giao thức truyền tải:** **Netflux** – Một API trừu tượng hóa việc truyền tin thời gian thực qua WebSockets.
*   **Lưu trữ:** Dữ liệu được lưu dưới dạng các file `.ndjson` (Newline Delimited JSON) trong `datastore/` thay vì dùng cơ sở dữ liệu quan hệ truyền thống, giúp tối ưu cho việc ghi log các thay đổi (encapsulated logs).

---

### 2. Tư duy Kỹ thuật & Kiến trúc (Engineering & Architectural Thinking)

Kiến trúc của CryptPad được xây dựng quanh nguyên tắc **"Zero-Knowledge Server"**:

*   **Sandboxing (Cô lập bảo mật):** Dự án sử dụng hai domain khác nhau (Main domain và Sandbox domain). 
    *   *Main Domain:* Quản lý khóa mật mã, tài khoản.
    *   *Sandbox Domain:* Chạy giao diện người dùng (UI) và các script chỉnh sửa. Nếu UI bị tấn công XSS, kẻ tấn công cũng không thể lấy được khóa mã hóa từ Main Domain nhờ chính sách Same-Origin Policy.
*   **Client-side Heavy:** Hầu hết logic xử lý văn bản, render và mã hóa nằm ở phía Client (`www/`). Server (`server.js`) đóng vai trò là một "bưu tá" thông minh: nhận các gói tin đã mã hóa, lưu trữ và chuyển tiếp (broadcast) tới các client khác.
*   **Kiến trúc phân tán (Worker):** Sử dụng `Web Workers` và `Service Workers` (`src/worker/`) để xử lý các tác vụ nặng về mã hóa và lưu trữ cục bộ (IndexDB) nhằm tránh làm treo giao diện người dùng.

---

### 3. Các kỹ thuật chính nổi bật

*   **Xác thực không mật khẩu (Zero-Knowledge Auth):** Server không bao giờ biết mật khẩu của bạn. Mật khẩu được dùng để tạo ra các cặp khóa (public/private key) thông qua `scrypt-async`. Khóa này dùng để định danh và giải mã dữ liệu.
*   **Mã hóa phân tầng:**
    *   *Metadata:* Mã hóa riêng thông tin về file (tên, ngày tạo).
    *   *Content:* Mã hóa nội dung văn bản.
    *   *Blobs:* Mã hóa các file đính kèm (hình ảnh, PDF).
*   **Khả năng tùy chỉnh (White-labeling):** Thư mục `customize.dist/` cho phép các quản trị viên thay đổi giao diện, logo, màu sắc mà không cần can thiệp sâu vào code lõi.
*   **Tối ưu hóa tài nguyên:** Sử dụng `Cluster` module trong Node.js (`server.js`) để tận dụng đa nhân CPU, giúp xử lý hàng ngàn kết nối WebSocket cùng lúc.

---

### 4. Tóm tắt luồng hoạt động (File Flow)

Quy trình từ lúc người dùng truy cập đến khi dữ liệu được lưu:

1.  **Khởi tạo (Entry Point):** `server.js` khởi chạy, nạp cấu hình từ `config/config.js`, mở cổng HTTP (3000) và WebSocket (3003).
2.  **Tải ứng dụng:** Người dùng truy cập trang web -> Trình duyệt tải các file tĩnh từ `www/`. File `www/common/boot.js` sẽ kích hoạt quá trình khởi động ứng dụng.
3.  **Xác thực/Mã hóa:** 
    *   Người dùng nhập mật khẩu -> JS tại trình duyệt tính toán ra khóa mã hóa.
    *   Khóa này thường nằm ở phần **Hash (#)** của URL (ví dụ: `cryptpad.fr/code/#xyz...`). Phần Hash này không bao giờ được gửi lên Server.
4.  **Kết nối Real-time:** Trình duyệt thiết lập kết nối WebSocket thông qua `lib/api.js` và giao thức Netflux.
5.  **Chỉnh sửa & Đồng bộ:**
    *   Người dùng gõ văn bản -> `ChainPad` tạo ra một "patch" (thay đổi).
    *   Patch được mã hóa bằng khóa bí mật tại máy người dùng.
    *   Gửi patch qua WebSocket -> Server nhận và lưu vào `datastore/` (dưới dạng log).
    *   Server đẩy patch này tới tất cả các user khác đang mở cùng một file.
6.  **Giải mã:** Các user khác nhận được patch mã hóa -> Dùng khóa trong URL Hash để giải mã và cập nhật lên màn hình.

---

### 5. Cấu trúc thư mục quan trọng

*   **`config/`**: Chứa file cấu hình hệ thống (Domain, cổng, quyền admin).
*   **`lib/`**: Logic xử lý phía Server (quản lý lưu trữ, quota, rpc).
*   **`www/`**: Chứa mã nguồn của từng ứng dụng con (Pad, Code, Sheet, Kanban...).
*   **`datastore/` (tạo khi chạy)**: Nơi lưu giữ các bản ghi (logs) đã mã hóa của tất cả tài liệu.
*   **`blob/` (tạo khi chạy)**: Lưu trữ các file hình ảnh/đính kèm đã mã hóa.
*   **`customize.dist/`**: Các bản mẫu để thay đổi giao diện (CSS, HTML, Logo).

**Kết luận:** CryptPad là một dự án có độ phức tạp kỹ thuật cao, kết hợp nhuần nhuyễn giữa bảo mật mật mã học và lập trình hướng sự kiện (event-driven). Điểm mạnh nhất của nó là việc biến Server thành một nơi lưu trữ mù (blind storage), đảm bảo quyền riêng tư tuyệt đối cho người dùng.