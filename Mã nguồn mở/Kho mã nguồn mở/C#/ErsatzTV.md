Dựa trên các tệp tin mã nguồn của **ErsatzTV**, đây là phân tích chuyên sâu về công nghệ, kiến trúc và phương thức vận hành của hệ thống biến thư viện media cá nhân thành các kênh truyền hình trực tiếp (Linear TV).

---

### 1. Công nghệ cốt lõi (Core Technologies)
ErsatzTV là một ứng dụng .NET hiện đại, sử dụng các công nghệ tiên tiến nhất trong hệ sinh thái Microsoft:
*   **Runtime:** **.NET 10** (phiên bản cực kỳ mới, cho thấy dự án luôn cập nhật hiệu năng và tính năng mới nhất).
*   **Web Framework:** **ASP.NET Core** kết hợp với **Blazor Server**. Blazor được dùng để xây dựng giao diện quản trị (Dashboard) động mà không cần viết quá nhiều JavaScript, sử dụng thư viện UI **MudBlazor**.
*   **Streaming & Transcoding:** **FFmpeg** (v7.1.1). Đây là "trái tim" xử lý video, hỗ trợ đa dạng tăng tốc phần cứng: **NVENC (Nvidia), QSV (Intel), VAAPI, AMF (AMD), VideoToolbox (macOS)**.
*   **Database:** Hỗ trợ song song **SQLite** (cho người dùng cá nhân, cấu hình nhẹ) và **MySQL** (cho nhu cầu lưu trữ lớn, mở rộng). Sử dụng **Entity Framework Core** làm ORM.
*   **Search Engine:** Sử dụng **Lucene.NET** để đánh chỉ mục nội dung media cục bộ và hỗ trợ cả **Elasticsearch** cho các hệ thống lớn.
*   **Communication Patterns:** Sử dụng **MediatR** để thực hiện mô hình **CQRS** (Command Query Responsibility Segregation), giúp tách biệt logic đọc và ghi dữ liệu.

---

### 2. Tư duy Kiến trúc (Architectural Mindset)
Kiến trúc của ErsatzTV đi theo hướng **Clean Architecture** (Kiến trúc sạch) và **Decoupled Workers**:

*   **Phân lớp (Layering):**
    *   `ErsatzTV.Core`: Chứa các Business Logic cốt lõi, Domain Models và các Interface (Abstractions).
    *   `ErsatzTV.Application`: Chứa các Use Case (Commands/Queries), ví dụ: "Tạo kênh", "Lấy lịch phát sóng".
    *   `ErsatzTV.Infrastructure`: Triển khai thực tế các Interface (Database Context, API Clients cho Plex/Jellyfin, logic tương tác File System).
    *   `ErsatzTV.Scanner`: Một thành phần riêng biệt chuyên trách việc quét thư viện, xử lý Metadata (NFO) mà không làm ảnh hưởng đến luồng streaming chính.
*   **Kiến trúc Playout (Linear TV):** Khác với các ứng dụng VOD (Video on Demand) như Plex phát từ đầu tệp, ErsatzTV tư duy theo "Trục thời gian thực". Hệ thống tính toán dựa trên thời điểm hiện tại để biết video nào đang phát ở giây thứ bao nhiêu, tạo cảm giác như xem TV truyền thống.
*   **Modular Pipeline:** Luồng xử lý FFmpeg được xây dựng dưới dạng Pipeline, cho phép chèn linh hoạt các Filter (Watermark - Logo đài, Subtitles, Normalize Loudness).

---

### 3. Các kỹ thuật chính (Key Techniques)
*   **FFmpeg Pipeline Building:** Hệ thống tự động tạo ra các câu lệnh FFmpeg cực kỳ phức tạp dựa trên cấu hình kênh (Resolution, Bitrate, Codec) và định dạng tệp nguồn.
*   **Hardware Acceleration Detection:** Tự động phát hiện khả năng của GPU để chọn bộ giải mã/mã hóa tối ưu nhất (ví dụ: ưu tiên giải mã H265 bằng phần cứng).
*   **IPTV/EPG Generation:** Triển khai các chuẩn truyền hình qua internet:
    *   Tạo danh sách kênh định dạng **M3U**.
    *   Tạo lịch chương trình điện tử **XMLTV**.
    *   Giả lập giao thức **HDHomeRun** để các ứng dụng như Plex hoặc Emby có thể nhận diện ErsatzTV như một đầu thu kỹ thuật số (Tuner).
*   **Media Server Integration:** Kỹ thuật đồng bộ hai chiều với **Plex, Jellyfin, và Emby** thông qua Webhook và API để lấy dữ liệu media mà không cần quét lại file vật lý.
*   **Scripted Schedules:** Cho phép người dùng viết kịch bản phát sóng phức tạp (ví dụ: phát ngẫu nhiên, phát theo mùa, chèn quảng cáo/filler giữa các tập phim).

---

### 4. Tóm tắt luồng hoạt động (Operational Workflow)

1.  **Giai đoạn Khám phá (Discovery):**
    *   Scanner quét các thư mục cục bộ hoặc kết nối API tới các Media Server (Plex/Jellyfin).
    *   Metadata được trích xuất (tên phim, tập, mùa, thời lượng) và lưu vào Database.
    *   Search Index (Lucene) được cập nhật để tìm kiếm nhanh.

2.  **Giai đoạn Lập lịch (Scheduling):**
    *   Người dùng tạo **Collections** (Bộ sưu tập) hoặc **Smart Collections**.
    *   Người dùng thiết lập **Schedules** (Lịch phát): Quy định thứ tự phát, các quy tắc lặp lại.
    *   **Playout Engine** tính toán một "Timeline" phát sóng dài hạn cho từng kênh.

3.  **Giai đoạn Phát sóng (Streaming):**
    *   **Client** (VLC, Tivimate, Plex) gửi yêu cầu tới `IptvController`.
    *   Hệ thống xác định tệp media cần phát ngay lúc đó.
    *   **FFmpegProcessService** dựng câu lệnh transcode -> Output luồng stream dưới dạng **HLS** hoặc **MPEG-TS**.
    *   Nếu có Logo hoặc Subtitle, chúng được "đốt" (burn-in) trực tiếp vào luồng video bằng FFmpeg Filters.

4.  **Duy trì trạng thái:** 
    *   Background Services liên tục chạy để cập nhật lịch EPG và dọn dẹp các tệp tạm (transcode cache).

**Kết luận:** ErsatzTV là một hệ thống trung gian (Middleware) cực kỳ phức tạp và tinh vi, kết hợp giữa quản lý cơ sở dữ liệu mạnh mẽ và xử lý tín hiệu video thời gian thực chuyên sâu.