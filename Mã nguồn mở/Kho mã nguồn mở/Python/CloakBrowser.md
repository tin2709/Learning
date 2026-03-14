Dựa trên mã nguồn và tài liệu của dự án **CloakBrowser**, đây là bản phân tích chi tiết về công nghệ, kiến trúc và cách thức hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án này là một công cụ trình duyệt ẩn danh (stealth browser) được thiết kế để vượt qua các hệ thống chống bot (antibot).

*   **Lõi Trình duyệt (Engine):** **Patched Chromium (C++)**. Điểm khác biệt lớn nhất là dự án không chỉ thay đổi cấu hình mà can thiệp trực tiếp vào mã nguồn C++ của Chromium để sửa đổi vân tay trình duyệt (fingerprint) ở mức thấp nhất.
*   **Ngôn ngữ bao bọc (Wrapper):**
    *   **Python (3.9+):** Sử dụng `playwright` và `patchright` làm backend điều khiển.
    *   **TypeScript/JavaScript (Node.js 18+):** Hỗ trợ cả Playwright và Puppeteer.
*   **Hạ tầng và Thư viện:**
    *   **Playwright/Puppeteer:** Framework chính để điều khiển trình duyệt.
    *   **GeoLite2 (MaxMind):** Để tự động khớp múi giờ và ngôn ngữ dựa trên IP của Proxy.
    *   **Docker:** Cung cấp môi trường chạy sẵn (multi-arch: x86_64, arm64).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của CloakBrowser tập trung vào việc **"tàng hình hóa"** từ gốc thay vì chỉ đánh lừa bề mặt:

*   **Source-level Stealth (Ẩn danh mức mã nguồn):** Thay vì tiêm JavaScript (JS injection) — thứ dễ bị các hệ thống như Cloudflare hay Akamai phát hiện qua thứ tự thực thi — CloakBrowser sửa đổi các giá trị trả về của trình duyệt (Canvas, WebGL, Audio, GPU, Screen) ngay trong file thực thi (`.exe` hoặc binary).
*   **Kiến trúc "Drop-in Replacement":** Dự án được thiết kế để thay thế hoàn toàn thư viện Playwright/Puppeteer gốc. Người dùng chỉ cần đổi 1 dòng import mà không cần thay đổi logic code cũ.
*   **Coherent Identity (Danh tính nhất quán):** Hệ thống tạo ra một "Seed" (hạt giống) ngẫu nhiên mỗi lần khởi chạy. Mọi thông số từ phần cứng, độ phân giải màn hình đến font chữ đều được tính toán từ Seed này để đảm bảo tính logic (ví dụ: máy chạy card NVIDIA thì không thể có các thuộc tính của Apple GPU).
*   **Behavioral Simulation (Mô phỏng hành vi):** Không chỉ ẩn danh về thông số, dự án còn mô phỏng cách người dùng thật tương tác (di chuột, gõ phím).

### 3. Các kỹ thuật chính (Key Techniques)

*   **33 C++ Patches:** Can thiệp vào các thành phần nhạy cảm như `navigator.webdriver`, các tín hiệu CDP (Chrome DevTools Protocol), và các hàm báo cáo phần cứng.
*   **Hệ thống Humanize (Hành vi người):**
    *   **Mouse:** Sử dụng đường cong **Bézier** để di chuyển chuột, có hiện tượng rung (wobble) và di chuyển quá mục tiêu (overshoot) nhẹ giống tay người.
    *   **Keyboard:** Tốc độ gõ phím biến thiên, có khoảng dừng suy nghĩ và thỉnh thoảng gõ sai rồi tự sửa (mistyping & self-correction).
    *   **Scroll:** Cuộn trang có gia tốc và giảm tốc thay vì nhảy cóc.
*   **Normalization Quota:** Kỹ thuật chuẩn hóa dung lượng lưu trữ (Storage Quota) để vượt qua các bài kiểm tra chế độ ẩn danh (Incognito detection).
*   **GeoIP Sync:** Tự động phát hiện vị trí IP của Proxy để cấu hình múi giờ (Timezone) và ngôn ngữ (Locale) tương ứng, tránh sự lệch lạc giữa vân tay trình duyệt và vị trí mạng.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng hoạt động của CloakBrowser diễn ra như sau:

1.  **Khởi tạo:** Người dùng gọi hàm `launch(humanize=True, proxy=...)`.
2.  **Quản lý Binary:** Wrapper (Python/JS) kiểm tra xem phiên bản Chromium đã được chỉnh sửa đã có trong máy chưa. Nếu chưa, nó sẽ tải bản binary tương ứng với hệ điều hành (Windows, macOS, Linux) từ server của CloakHQ và xác thực SHA-256.
3.  **Cấu hình Stealth:**
    *   Wrapper tạo một Seed ngẫu nhiên.
    *   Nếu có Proxy, hệ thống tra cứu GeoIP để lấy Múi giờ/Ngôn ngữ.
    *   Các thông số này được truyền vào binary qua các cờ (flags) dòng lệnh đặc biệt.
4.  **Kích hoạt Trình duyệt:** Binary Chromium khởi chạy. Các bản vá C++ bên trong sẽ tự động tạo ra một bộ thông số phần cứng duy nhất dựa trên Seed.
5.  **Áp dụng lớp Hành vi:** Nếu chế độ `humanize` được bật, Wrapper sẽ "ghi đè" (patch) lên các hàm của Playwright như `page.click()` hay `page.fill()`. Khi code gọi lệnh click, thay vì click ngay lập tức, hệ thống sẽ tính toán đường cong di chuyển chuột và thực hiện các bước di chuyển trung gian trước khi nhấn.
6.  **Tương tác:** Người dùng điều khiển trình duyệt bằng API tiêu chuẩn.
7.  **Đóng:** Kết thúc phiên, xóa các cache tạm nếu cần hoặc lưu lại Profile (nếu dùng Persistent Context).

**Kết luận:** CloakBrowser là một giải pháp ẩn danh trình duyệt rất sâu sắc, kết hợp giữa việc chỉnh sửa phần mềm mức thấp (C++) và mô phỏng trí tuệ nhân tạo mức cao (Behavioral) để biến trình duyệt tự động thành một trình duyệt "người thật" trong mắt các hệ thống bảo mật.