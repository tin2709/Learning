Dưới đây là phân tích chi tiết về dự án **cosmo-tui**, một ứng dụng Terminal Dashboard sử dụng dữ liệu mở từ NASA:

### 1. Công nghệ cốt lõi (Core Technologies)
Ứng dụng được xây dựng hoàn toàn bằng Python với các thư viện hiện đại tập trung vào giao diện dòng lệnh (TUI):

*   **Framework TUI:** **Textual**. Đây là một framework cực mạnh dựa trên mô hình lập trình hướng sự kiện (async), cho phép tạo ra các thành phần giao diện terminal có thể tương tác giống như trang web.
*   **Định dạng nội dung:** **Rich**. Thư viện này hỗ trợ hiển thị văn bản màu sắc, bảng biểu, thanh tiến trình và đặc biệt là các ký tự Unicode phức tạp trong terminal.
*   **Xử lý mạng:** **httpx**. Một thư viện HTTP client hỗ trợ bất đồng bộ (async), giúp ứng dụng gọi nhiều API của NASA cùng lúc mà không làm treo giao diện.
*   **Toán học không gian:** **sgp4**. Thư viện này được dùng để tính toán quỹ đạo thời gian thực của Trạm Vũ trụ Quốc tế (ISS) từ dữ liệu TLE (Two-Line Element).
*   **Hệ thống cấu hình:** **platformdirs**. Đảm bảo lưu trữ API Key và cài đặt đúng vị trí theo tiêu chuẩn của từng hệ điều hành (Windows, Linux, macOS).

### 2. Tư duy Kiến trúc (Architectural Thinking)
Dự án tuân thủ tư duy thiết kế mô-đun hóa cao, tách biệt rõ rệt giữa dữ liệu và hiển thị:

*   **Tách biệt API (Data Layer):** Thư mục `src/cosmo/api/` chia nhỏ mỗi dịch vụ của NASA thành một file riêng (eonet.py, neows.py, sentry.py...). Mỗi file định nghĩa các `dataclass` để cấu trúc hóa dữ liệu thô từ JSON trả về, giúp code dễ bảo trì và kiểm soát lỗi.
*   **Widget-based UI (Component Layer):** Giao diện được chia thành các Widget độc lập trong `src/cosmo/widgets/`. Ví dụ: `WorldMap` chỉ lo vẽ bản đồ, `AsteroidTable` chỉ lo hiển thị bảng thiên thạch. Tư duy này giúp việc tái sử dụng và kiểm thử từng thành phần trở nên đơn giản.
*   **Controller (App Layer):** File `app.py` đóng vai trò là "bộ não" điều phối, chịu trách nhiệm khởi tạo các tác vụ chạy nền (loop), gọi API và cập nhật dữ liệu vào các Widget thông qua cơ chế `reactive` của Textual.

### 3. Các kỹ thuật chính (Key Techniques)
*   **Sub-pixel Rasterization (Kỹ thuật vẽ bản đồ):** Đây là điểm ấn tượng nhất. Thay vì vẽ 1 ký tự cho 1 điểm tọa độ, ứng dụng sử dụng **Ký tự Braille (Unicode)**. Mỗi ô ký tự terminal được chia nhỏ thành một lưới 2x4 "điểm ảnh" (sub-pixel). Kỹ thuật này giúp tăng độ phân giải của bản đồ thế giới lên gấp 8 lần so với cách vẽ thông thường.
*   **Point-in-Polygon (Địa lý học):** Ứng dụng tích hợp dữ liệu đa giác (Natural Earth land polygons) từ file GeoJSON. Nó sử dụng thuật toán kiểm tra một điểm (tọa độ sự kiện) nằm trong đa giác nào để xác định vùng lục địa/đại dương một cách chính xác trên màn hình terminal.
*   **Async Concurrency:** Sử dụng `asyncio.gather` để thực hiện việc làm mới (refresh) dữ liệu từ 6-7 nguồn API khác nhau cùng lúc. Điều này giúp Dashboard luôn mượt mà, không bị trễ khi một API phản hồi chậm.
*   **TLE Propagation:** Thay vì hỏi API vị trí ISS liên tục (tốn băng thông), ứng dụng chỉ tải dữ liệu TLE (thông số quỹ đạo) một lần mỗi giờ và tự tính toán vị trí bằng toán học (SGP4) ngay tại máy cục bộ mỗi 30 giây.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)
1.  **Khởi động:** Kiểm tra cấu hình và API Key qua `config.py`. Nếu chưa có, ứng dụng sẽ thực hiện luồng `first_run_setup` để hướng dẫn người dùng đăng ký key từ NASA.
2.  **Khởi tạo UI:** `CosmoApp` xây dựng bố cục (Layout) gồm: Header, Bản đồ (WorldMap), Danh sách sự kiện (EventList) và các tab thông tin phụ.
3.  **Tải dữ liệu ban đầu:** Gọi đồng thời các API (EONET cho thiên tai, NeoWs cho thiên thạch, DONKI cho thời tiết không gian, APOD cho ảnh trong ngày).
4.  **Vẽ bản đồ:**
    *   Load dữ liệu địa lý từ GeoJSON.
    *   Chuyển đổi tọa độ Lat/Lon của các sự kiện (Cháy rừng, Bão, Động đất) thành tọa độ X/Y trên lưới terminal.
    *   Render các ký tự Braille để tạo hình dáng lục địa.
5.  **Vòng lặp (Event Loop):**
    *   Cập nhật đồng hồ hệ thống mỗi giây.
    *   Tính toán lại vị trí ISS mỗi 30 giây.
    *   Tự động gọi lại toàn bộ API để làm mới dữ liệu sau mỗi 300 giây (mặc định).
6.  **Tương tác:** Khi người dùng chọn một dòng trong `EventList`, một thông điệp (`Message`) được gửi đi để `WorldMap` thay đổi ký tự hiển thị (từ hình tròn sang chữ X) nhằm làm nổi bật vị trí đó trên bản đồ.