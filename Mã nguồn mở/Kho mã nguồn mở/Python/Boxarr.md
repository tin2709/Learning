
### 1. Phân tích Công Nghệ Cốt Lõi (Core Tech Stack)

Dự án được xây dựng trên nền tảng Python hiện đại với các thư viện chuyên biệt:

*   **Backend Framework:** `FastAPI` (Asynchronous). Tận dụng tính năng async/await để xử lý các yêu cầu I/O (gọi API Radarr, scraping) mà không làm tắc nghẽn hệ thống.
*   **Web Scraping:** `BeautifulSoup4` kết hợp với `httpx`. Dùng để bóc tách dữ liệu từ Box Office Mojo. Dự án sử dụng `httpx` thay vì `requests` để hỗ trợ async hoàn toàn.
*   **Automation/Scheduling:** `APScheduler`. Quản lý các công việc chạy ngầm (cron jobs) để tự động cập nhật dữ liệu hàng tuần.
*   **Data Validation & Settings:** `Pydantic` và `Pydantic-settings`. Toàn bộ cấu hình hệ thống và dữ liệu API được định nghĩa qua các Model, đảm bảo tính đúng đắn của dữ liệu.
*   **Template Engine:** `Jinja2`. Sử dụng cơ chế Server-Side Rendering (SSR) giúp trang web tải nhanh và thân thiện với các hệ thống tự lưu trữ (self-hosted).
*   **Containerization:** `Docker` với cơ chế multi-stage build, tối ưu hóa kích thước image và hỗ trợ đa kiến trúc (amd64, arm64).

---

### 2. Tư Duy Kiến Trúc (Architecture Thinking)

Boxarr tuân thủ kiến trúc phân lớp (Layered Architecture) rõ ràng, giúp tách biệt giữa logic nghiệp vụ và giao diện:

1.  **Core Layer (`src/core/`):** Chứa "trái tim" của ứng dụng.
    *   Tách biệt các Service: `RadarrService` chỉ lo việc giao tiếp API, `BoxOfficeService` chỉ lo việc cào dữ liệu, `MovieMatcher` lo việc so khớp.
    *   **Stateless logic:** Các service được thiết kế để có thể hoạt động độc lập (CLI mode hoặc API mode).
2.  **API/Interface Layer (`src/api/`):**
    *   Sử dụng `APIRouter` để phân tách các module: Admin, Config, Movies, Scheduler. Điều này giúp mã nguồn dễ bảo trì khi mở rộng.
3.  **Utility Layer (`src/utils/`):**
    *   Quản lý cấu hình tập trung (`config.py`) hỗ trợ cả tệp YAML và biến môi trường.
    *   Hệ thống Logging xoay vòng (Rotating log) giúp theo dõi lỗi mà không làm đầy bộ nhớ.
4.  **Data Persistence:**
    *   Không sử dụng Database nặng nề (như MySQL/Postgres). Thay vào đó, Boxarr sử dụng các tệp **JSON** để lưu trữ kết quả hàng tuần và **YAML** cho cấu hình. Cách tiếp cận này rất phù hợp với ứng dụng dạng "Sidecar" cho các phần mềm Media Lab.

---

### 3. Các Kỹ Thuật Key (Key Technical Implementations)

Dự án giải quyết các bài toán khó bằng các kỹ thuật thông minh:

*   **Thuật toán so khớp tiêu đề (Movie Matching):** Đây là phần phức tạp nhất. `matcher.py` xử lý:
    *   *Normalization:* Loại bỏ ký tự đặc biệt, đưa về chữ thường.
    *   *Number Handling:* Chuyển đổi số La Mã (IV -> 4) và chữ số thành chữ viết (4 -> four) để tăng tỷ lệ khớp chính xác.
    *   *Subtitle Stripping:* Loại bỏ các phần sau dấu hai chấm (:) hoặc gạch ngang (-) để tìm tên gốc của phim.
*   **Quản lý thư mục theo thể loại (Genre-based Root Folders):**
    *   Sử dụng hệ thống ưu tiên (Priority-based mapping). Nếu một phim có nhiều thể loại, hệ thống sẽ chọn thư mục có độ ưu tiên cao nhất (ví dụ: Phim "Animation + War" sẽ vào thư mục Animation nếu độ ưu tiên cao hơn).
*   **Làm giàu dữ liệu (Metadata Enrichment):**
    *   Dù phim chưa có trong Radarr, Boxarr vẫn thực hiện "TMDB Lookup" thông qua Radarr để lấy Poster và mô tả phim, giúp giao diện người dùng luôn đầy đủ thông tin.
*   **Hệ thống Dynamic Reload:**
    *   Thay đổi lịch trình Cron trong UI sẽ được áp dụng ngay lập tức vào `APScheduler` mà không cần khởi động lại Container (thông qua endpoint `/api/scheduler/reload`).

---

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

Luồng đi của dữ liệu từ internet đến thư viện người dùng:

1.  **Kích hoạt (Trigger):** Đến giờ hẹn (Cron) hoặc người dùng nhấn "Update Now".
2.  **Thu thập (Scraping):** `BoxOfficeService` truy cập Box Office Mojo, lấy danh sách Top 10 phim kèm doanh thu.
3.  **So khớp (Matching):** 
    *   Hệ thống lấy toàn bộ danh sách phim hiện có trong Radarr.
    *   `MovieMatcher` chạy qua các bộ lọc để xác định phim nào đã có, phim nào chưa.
4.  **Tự động thêm (Auto-Add):** Nếu tính năng này bật:
    *   Kiểm tra bộ lọc (Genre whitelist/blacklist, Age rating, Năm phát hành).
    *   Xác định thư mục lưu trữ (Root Folder Mapping).
    *   Gọi API Radarr để thêm phim vào danh sách theo dõi.
5.  **Lưu trữ (Persistence):** Tạo một tệp JSON (ví dụ: `2024W10.json`) chứa toàn bộ trạng thái của tuần đó.
6.  **Hiển thị (Rendering):** Người dùng truy cập Web UI, FastAPI đọc tệp JSON, kiểm tra trạng thái thực tế từ Radarr (Downloaded/Missing) và render ra giao diện qua Jinja2.

### Tổng kết
Boxarr là một ví dụ tốt về việc xây dựng công cụ hỗ trợ (utility tool) trong hệ sinh thái Media Server. Nó không cố gắng thay thế Radarr hay Overseerr mà đóng vai trò là một **người môi giới dữ liệu tự động** (Automated Data Broker), tập trung vào tính mainstream của thị trường điện ảnh.