Dưới đây là phân tích chi tiết về dự án **SearXNG**, một công cụ tìm kiếm siêu dữ liệu (metasearch engine) mã nguồn mở tập trung vào quyền riêng tư.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Backend:** **Python (80.2%)** là ngôn ngữ chủ đạo. Dự án sử dụng framework **Flask** làm nền tảng Web server.
*   **Frontend:** Đang trong quá trình hiện đại hóa. Sử dụng **TypeScript**, **Vite** để đóng gói (bundle) và **Less** để xử lý CSS. Dự án duy trì triết lý "HTML-first", JS là tùy chọn.
*   **Templating Engine:** **Jinja2** được dùng để render các trang HTML phía server.
*   **Database & Caching:** **Valkey** (một bản fork mã nguồn mở của Redis) được dùng làm cache và lưu trữ trạng thái phiên, kết hợp với **SQLite**.
*   **Networking:** Sử dụng thư viện **HTTPX** (thay cho requests) để thực hiện các yêu cầu HTTP đồng thời (concurrent) và hỗ trợ HTTP/2.
*   **Toolchain & DevOps:** 
    *   **Makefile** và **Bash script** (`manage`, `utils/`) để quản lý toàn bộ quy trình phát triển.
    *   **Go:** Được tích hợp để chạy một số công cụ phát triển nhanh (như `shfmt` để format script).
    *   **Mise/ASDF:** Để quản lý phiên bản ngôn ngữ (Python, Node, Go).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của SearXNG được xây dựng theo mô hình **Middleware Aggregator (Bộ điều phối trung gian)**:

*   **Tính mô-đun của Engine:** Mỗi nguồn tìm kiếm (Google, Bing, Wikipedia...) được coi là một "Engine" riêng biệt nằm trong `searx/engines/`. Tư duy này cho phép cộng đồng dễ dàng thêm mới hoặc sửa đổi một nguồn mà không ảnh hưởng đến lõi hệ thống.
*   **Kiến trúc không trạng thái (Stateless):** SearXNG cố gắng không lưu trữ dữ liệu người dùng. Mọi truy vấn là độc lập, giúp bảo vệ quyền riêng tư tối đa.
*   **Xử lý song song (Concurrency):** Khi người dùng tìm kiếm, SearXNG sẽ gửi yêu cầu đồng thời đến hàng chục engine khác nhau, sau đó tổng hợp kết quả lại. Điều này yêu cầu một kiến trúc xử lý bất đồng bộ hoặc đa luồng cực tốt để đảm bảo tốc độ.
*   **Ưu tiên quyền riêng tư (Privacy-by-design):** Hệ thống tự động xóa các thông tin định danh (cookies, IP người dùng) trước khi gửi yêu cầu tới các engine tìm kiếm bên thứ ba.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Scraping & Parsing:** Sử dụng **LXML** và **XPath** để bóc tách dữ liệu từ HTML của các trang tìm kiếm khác. Đây là kỹ thuật quan trọng nhất vì nhiều engine không cung cấp API miễn phí.
*   **Bot Detection & Rate Limiting:** Tích hợp bộ lọc (`limiter.py`) để ngăn chặn việc instance bị khai thác bởi các bot khác, bảo vệ địa chỉ IP của server không bị các engine lớn (như Google) đưa vào blacklist.
*   **Image Proxy:** Kỹ thuật proxy hình ảnh giúp người dùng xem kết quả ảnh mà không cần kết nối trực tiếp đến server nguồn, tránh bị theo dõi qua IP.
*   **Babel Localization:** Quy trình dịch thuật tự động hóa rất mạnh mẽ thông qua thư viện Babel và nền tảng Weblate, hỗ trợ hơn 58 ngôn ngữ.
*   **Static Build với Vite:** Sử dụng Vite trong thư mục `client/simple` để tối ưu hóa tài nguyên frontend, giúp trang web load cực nhanh và hỗ trợ cả hai chế độ LTR (trái sang phải) và RTL (phải sang trái).

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Tiếp nhận (Input):** Người dùng nhập truy vấn vào `webapp.py`. Truy vấn này có thể chứa các "bangs" (ví dụ: `!github`) để chỉ định engine.
2.  **Phân tích truy vấn (Parsing Query):** Hệ thống phân tích từ khóa, xác định các category (General, IT, Science...) và các engine cần gọi.
3.  **Phân tán yêu cầu (Dispatching):** Lõi tìm kiếm gửi yêu cầu HTTP đồng thời đến các engine đã chọn thông qua proxy hoặc kết nối trực tiếp (đã xóa định danh).
4.  **Tổng hợp & Xếp hạng (Aggregation & Ranking):** 
    *   Thu thập kết quả trả về từ tất cả engine.
    *   Loại bỏ các kết quả trùng lặp.
    *   Tính toán điểm số (scoring) dựa trên trọng số của engine và độ liên quan để sắp xếp thứ tự.
5.  **Render kết quả:** Dữ liệu sau khi tổng hợp được đưa qua Jinja2 template để trả về trang HTML hoàn chỉnh cho trình duyệt.
6.  **Dọn dẹp:** Không có lịch sử tìm kiếm hay hồ sơ người dùng nào được lưu lại sau khi phiên làm việc kết thúc.

### Đánh giá chung
SearXNG là một dự án có độ phức tạp cao về mặt **xử lý dữ liệu phân tán** và **tối ưu hóa mạng**. Nó là minh chứng cho việc sử dụng Python hiệu quả trong các tác vụ yêu cầu tính kết nối và xử lý văn bản quy mô lớn.