Dưới đây là phân tích chi tiết về dự án **iSponsorBlockTV** dựa trên các tệp mã nguồn bạn đã cung cấp:

### 1. Công nghệ cốt lõi (Core Technology)

*   **Ngôn ngữ lập trình:** **Python (3.9+)** sử dụng mô hình lập trình **Bất đồng bộ (Asynchronous)** với `asyncio` làm nền tảng để quản lý nhiều kết nối thiết bị cùng lúc.
*   **Giao thức điều khiển:** Sử dụng **YouTube Lounge API** (thông qua thư viện `pyytlounge`). Đây là giao thức mà YouTube dùng để kết nối giữa ứng dụng điện thoại và TV (tương tự như tính năng "Watch on TV").
*   **Khám phá thiết bị:** Sử dụng giao thức **DIAL (Discovery and Launch)** và **SSDP** để tự động tìm kiếm các thiết bị chạy YouTube trong mạng nội bộ.
*   **Giao diện cấu hình (TUI):** Sử dụng framework **Textual** và **Rich** để xây dựng giao diện dòng lệnh (Terminal User Interface) cực kỳ chuyên nghiệp, có hỗ trợ CSS-like styling (`.tcss`).
*   **Quản lý kết nối HTTP:** Sử dụng `aiohttp` để thực hiện các yêu cầu mạng không chặn (non-blocking).
*   **API bên thứ ba:** Kết nối với **SponsorBlock API** để lấy dữ liệu về các đoạn quảng cáo trong video và **YouTube Data API v3** để truy xuất thông tin kênh/whitelist.

### 2. Tư duy kiến trúc (Architectural Thinking)

Dự án được xây dựng với kiến trúc hướng sự kiện (Event-driven) và module hóa rõ rệt:
*   **Cấu trúc Listener:** Lớp `DeviceListener` đóng vai trò là thực thể giám sát cho từng thiết bị. Mỗi thiết bị chạy trong một vòng lặp (`loop`) riêng biệt, không ảnh hưởng lẫn nhau.
*   **Cơ chế Cache thông minh:** Sử dụng `AsyncConditionalTTL` (Time-to-Live có điều kiện). Nếu một đoạn video đã được xác nhận (locked) bởi cộng đồng, nó sẽ được lưu cache lâu hơn để giảm tải cho API.
*   **Xử lý sai số (Offset Management):** Kiến trúc cho phép bù đắp độ trễ mạng (offset) giữa máy chủ iSponsorBlockTV và TV để việc nhảy đoạn (seek) diễn ra chính xác đến từng mili giây.
*   **Khả năng phục hồi (Resilience):** Có cơ chế `watchdog` (chó canh phòng) để kiểm tra trạng thái kết nối. Nếu không nhận được phản hồi từ YouTube trong 60 giây, hệ thống sẽ tự động khởi động lại tiến trình đăng ký thiết bị.

### 3. Các kỹ thuật chính nổi bật

*   **Ghép nối phân đoạn (Segment Merging):** Trong `api_helpers.py`, logic xử lý các phân đoạn quảng cáo rất thông minh: nếu hai đoạn quảng cáo nằm sát nhau (dưới 1 giây), chúng sẽ được gộp lại thành một để thực hiện lệnh "nhảy" duy nhất, tránh gây giật lag cho TV.
*   **Giả lập thiết bị di động:** Hệ thống giả lập các thông số của ứng dụng YouTube trên iOS (`ytios-phone`) để "đăng ký" quyền điều khiển với máy chủ Lounge của Google.
*   **Ghi đè âm lượng (Volume Overriding):** Kỹ thuật tự động Mute khi phát hiện quảng cáo và Unmute khi vào video chính bằng cách can thiệp trực tiếp vào trạng thái âm lượng của phiên kết nối.
*   **Hỗ trợ Proxy:** Tích hợp `trust_env` trong `aiohttp` cho phép người dùng chạy ứng dụng qua các mạng có kiểm soát (corporate networks/VPN).
*   **Containerization tối ưu:** Dockerfile sử dụng `python-alpine` và cơ chế Multi-stage build để giảm kích thước ảnh xuống mức tối thiểu và bảo mật hơn bằng cách biên dịch mã nguồn thành `.pyc` và xóa file `.py` gốc.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng hoạt động của dự án có thể tóm tắt qua các bước sau:

1.  **Khởi tạo & Cấu hình:** Người dùng sử dụng TUI (Setup Wizard) để ghép nối TV với ứng dụng thông qua mã (Pairing Code) hoặc tự động dò tìm (DIAL). Cấu hình được lưu vào `config.json`.
2.  **Kết nối & Đăng ký:** Ứng dụng khởi tạo một phiên kết nối bất đồng bộ đến máy chủ Lounge của YouTube cho từng TV. Nó đăng ký như một "Remote Control" (điều khiển từ xa).
3.  **Lắng nghe sự kiện (Monitoring):** Ứng dụng liên tục lắng nghe các sự kiện từ TV. Khi người dùng chọn một video:
    *   Hệ thống nhận được `videoId`.
    *   Ngay lập tức, nó gửi yêu cầu đến **SponsorBlock API** để lấy danh sách các đoạn cần bỏ qua (Sponsor, Intro, Outro...).
4.  **Xử lý thời gian thực:** Ứng dụng theo dõi thời gian phát hiện tại (`currentTime`) của video trên TV.
5.  **Thực thi Skip/Mute:**
    *   **Skip:** Khi thời gian phát chạm đến điểm bắt đầu của một đoạn quảng cáo, ứng dụng gửi lệnh `seekTo` đến TV để nhảy thẳng đến điểm kết thúc của đoạn đó.
    *   **Ad Muting:** Nếu phát hiện sự kiện quảng cáo gốc của YouTube (`onAdStateChange`), nó gửi lệnh `setVolume` với tham số `muted: true`. Khi quảng cáo hết, nó tự động trả lại âm lượng cũ.
6.  **Phản hồi cộng đồng:** Sau khi nhảy đoạn thành công, ứng dụng gửi thông báo về SponsorBlock API để tăng số lượt skip (giúp cộng đồng biết đoạn đó vẫn hoạt động tốt).