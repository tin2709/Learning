Dựa trên mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về **Fathom Lite** dưới các góc độ công nghệ, kiến trúc và kỹ thuật lập trình:

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Fathom Lite được xây dựng với tiêu chí: **Nhẹ (Lite), Tốc độ và Quyền riêng tư.**

*   **Backend:**
    *   **Ngôn ngữ:** **Go (Golang)**. Lựa chọn này tối ưu cho việc xử lý đồng thời (concurrency) các request thu thập dữ liệu và tạo ra một file thực thi duy nhất (single binary).
    *   **Database Drivers:** Hỗ trợ linh hoạt **SQLite** (mặc định), **MySQL**, và **PostgreSQL** thông qua `sqlx` và các driver chuẩn của Go.
    *   **Router:** `gorilla/mux` để điều hướng API và `gorilla/sessions` để quản lý phiên làm việc.
    *   **Asset Embedding:** Sử dụng `packr/v2` để đóng gói toàn bộ file HTML/JS/CSS vào trong file binary Go.
*   **Frontend:**
    *   **Thư viện UI:** **Preact** (phiên bản siêu nhẹ của React, chỉ ~3KB). Phù hợp với triết lý "Lite" của dự án.
    *   **Data Visualization:** **D3.js** để vẽ biểu đồ tương tác.
    *   **Build Tool:** **Gulp**, **Browserify** và **Babel** để biên dịch mã nguồn Javascript hiện đại và tối ưu hóa tài nguyên.
*   **Tracking:**
    *   **Tracker Script:** Một đoạn mã Javascript (`tracker.js`) viết thuần (Vanilla JS) để tối thiểu hóa độ trễ khi tải trang của khách hàng.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Fathom Lite tuân thủ nguyên tắc **"Simple is better"**:

*   **Monolithic Single Binary:** Toàn bộ hệ thống (Server, API, Frontend) được đóng gói trong một file duy nhất. Điều này cực kỳ thuận tiện cho việc triển khai (self-hosted) trên các VPS yếu hoặc Docker.
*   **Kiến trúc API-First:** Backend đóng vai trò là một RESTful API Server. Frontend là một Single Page Application (SPA) giao tiếp hoàn toàn qua JSON API.
*   **Privacy-Focused Tracking:**
    *   Không thu thập thông tin cá nhân (PII).
    *   Cơ chế xác định "Unique Visitor" dựa trên một chuỗi ngẫu nhiên lưu trong cookie (`_fathom`), tự động hết hạn và xóa sau 30 phút hoặc khi phiên kết thúc.
    *   Tôn trọng thiết lập "Do Not Track" (DNT) của trình duyệt.
*   **Database Agnostic:** Sử dụng các interface chuẩn của Go giúp hệ thống không bị phụ thuộc vào một loại DB cụ thể, cho phép người dùng từ cá nhân (dùng SQLite) đến doanh nghiệp (dùng Postgres/MySQL).

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Go (Backend):**
    *   **CLI-driven:** Sử dụng `urfave/cli` để quản lý các lệnh điều khiển (ví dụ: `fathom server`, `fathom user add`).
    *   **Middleware Pattern:** Áp dụng middleware để xử lý Gzip compression, Logging, và Authentication (xác thực phiên qua session).
    *   **Environment-based Config:** Sử dụng `envconfig` và `.env` để quản lý cấu hình (12-Factor App).
    *   **Migration:** Tích hợp `sql-migrate` để tự động cập nhật cấu trúc database khi nâng cấp phiên bản.

*   **Javascript (Frontend):**
    *   **Functional Components (Preact):** Sử dụng các component nhỏ gọn, dễ tái sử dụng như `CountWidget`, `Table`, `Chart`.
    *   **State Management:** Quản lý trạng thái tại component cha (`dashboard.js`) và truyền qua props, không cần các thư viện phức tạp như Redux.
    *   **D3 Integration:** Kỹ thuật vẽ biểu đồ thủ công trên DOM của Preact giúp đạt hiệu năng cao hơn các thư viện chart bọc sẵn.

*   **Tracker Kỹ thuật:**
    *   **Pixel Tracking:** Thay vì gửi XHR/Fetch phức tạp, tracker tạo một đối tượng `<img>` ẩn và gán dữ liệu vào Query String của thuộc tính `src`. Đây là kỹ thuật kinh điển giúp tránh vấn đề Cross-Origin (CORS) và hoạt động tốt trên mọi trình duyệt cũ.
    *   **Queueing:** Sử dụng một mảng `q` để lưu lại các sự kiện tracking nếu script chưa tải xong, đảm bảo không mất dữ liệu.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Hệ thống hoạt động qua hai luồng chính:

#### A. Luồng thu thập dữ liệu (Collection Flow):
1.  Người dùng truy cập vào trang web có gắn mã Fathom.
2.  `tracker.js` kiểm tra quyền (DNT) và đọc/ghi cookie `_fathom`.
3.  `tracker.js` thu thập thông tin: Pathname, Referrer, Hostname, IsNewVisitor, IsNewSession.
4.  Gửi yêu cầu GET đến `/collect` (endpoint của Fathom server) dưới dạng một request ảnh.
5.  Backend Go tiếp nhận request, trích xuất dữ liệu từ query string và lưu vào database (async/background processing để không làm chậm request).

#### B. Luồng hiển thị Dashboard (Presentation Flow):
1.  Admin đăng nhập vào Dashboard (SPA).
2.  Frontend gửi yêu cầu API đến `/api/sites` để lấy danh sách các site.
3.  Khi chọn một site và khoảng thời gian (DatePicker), Frontend gọi các endpoint:
    *   `/stats/site/agg`: Lấy tổng chỉ số (Visitors, Pageviews, Bounce Rate...).
    *   `/stats/pages/agg`: Lấy danh sách trang xem nhiều nhất.
    *   `/stats/referrers/agg`: Lấy nguồn truy cập.
    *   `/stats/site/realtime`: Lấy số người đang truy cập trực tuyến (long polling/interval).
4.  Backend thực hiện các câu lệnh SQL `GROUP BY` và `COUNT` để trả về dữ liệu tổng hợp.
5.  Frontend nhận JSON và dùng D3.js để render biểu đồ.

### Tổng kết
Fathom Lite là một ví dụ điển hình về việc xây dựng sản phẩm **"tinh gọn"**. Nó không sử dụng những framework nặng nề mà tập trung vào những thư viện tối giản nhất để đạt được mục tiêu: cài đặt trong 1 phút, chạy mượt mà trên phần cứng thấp, và bảo mật dữ liệu tuyệt đối cho người dùng.