Dựa trên mã nguồn của dự án **Browser Harness**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc, kỹ thuật và luồng hoạt động:

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ chính:** **Python 3.11+**, được chọn vì khả năng tương thích tốt với các công cụ AI (Codex, Claude Code).
*   **Giao thức điều khiển:** **CDP (Chrome DevTools Protocol)**. Thay vì dùng các framework nặng như Selenium hay Playwright, dự án giao tiếp trực tiếp với Chrome thông qua WebSocket.
*   **Thư viện quan trọng:**
    *   `cdp-use`: Thư viện thấp cấp để gửi các lệnh raw CDP.
    *   `websockets`: Duy trì kết nối thời gian thực với trình duyệt.
    *   `pillow (PIL)`: Xử lý hình ảnh (screenshots), vẽ điểm debug để AI xác nhận vị trí click.
    *   `fetch-use`: Proxy HTTP chuyên dụng để vượt qua các rào cản bot (CAPTCHA, residential proxy).
*   **Giao thức kết nối nội bộ:** **Unix Domain Sockets** (`/tmp/bu-default.sock`). Đây là cơ chế giao tiếp cực nhanh giữa tiến trình chạy code của AI và tiến trình Daemon giữ kết nối trình duyệt.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Browser Harness được thiết kế theo triết lý **"Agent-Centric"** (Lấy AI làm trung tâm):

*   **Self-healing (Tự chữa lành):** Đây là điểm độc đáo nhất. Hệ thống không cố gắng xây dựng mọi tính năng. Nếu AI thấy thiếu một hàm (ví dụ `upload_file`), nó được khuyến khích **tự chỉnh sửa file `helpers.py`** để thêm tính năng đó ngay trong quá trình thực hiện tác vụ.
*   **Thin & Transparent (Mỏng và Minh bạch):** Dự án chỉ khoảng 600 dòng code. Nó loại bỏ các lớp trừu tượng dư thừa (rails/recipes) để AI có "quyền tự do tuyệt đối" thao tác với trình duyệt như một người dùng thực thụ.
*   **Mô hình Client-Daemon:**
    *   **Daemon (`daemon.py`):** Đóng vai trò là "linh hồn", duy trì kết nối WebSocket bền vững với Chrome và lắng nghe lệnh từ Socket.
    *   **Client (`run.py` / `helpers.py`):** Là các công cụ AI sử dụng. Khi AI chạy một lệnh, Client gửi JSON qua Socket tới Daemon.
*   **Skill-based Learning (Học dựa trên kỹ năng):** Thay vì code cứng logic cho từng trang web, dự án sử dụng các file Markdown (`domain-skills/`) để truyền đạt "kinh nghiệm" (selectors, API ẩn) cho AI.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **JSON-RPC over Unix Sockets:** Daemon và Client trao đổi với nhau bằng các dòng JSON đơn lẻ kết thúc bằng ký tự newline (`\n`), giúp việc phân tích cú pháp cực kỳ đơn giản và nhanh chóng.
*   **Compositor-level Actions:** Ưu tiên tương tác ở cấp độ luồng hiển thị (Click theo tọa độ X, Y lấy từ screenshot) thay vì chỉ dựa vào DOM. Điều này giúp vượt qua các trang web phức tạp dùng Iframe, Shadow DOM hoặc Cross-origin.
*   **IIFE Wrapping cho JavaScript:** Hàm `js()` tự động bọc code JavaScript của AI vào một IIFE (Immediately Invoked Function Expression) nếu có từ khóa `return`, giúp AI lấy dữ liệu từ trình duyệt một cách an toàn.
*   **Coordinate-based Debugging:** Khi bật chế độ debug, hệ thống sử dụng Pillow để vẽ một vòng tròn đỏ tại tọa độ click lên screenshot, giúp AI tự kiểm tra xem mình có click nhầm chỗ hay không.

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Khởi tạo (Bootstrap):** AI chạy `browser-harness`. Hàm `ensure_daemon()` kiểm tra xem có Daemon nào đang sống không. Nếu không, nó tự động khởi chạy `daemon.py`.
2.  **Kết nối trình duyệt:** Daemon tìm kiếm file `DevToolsActivePort` của Chrome trên máy người dùng (hoặc kết nối tới Cloud Browser qua API Key) để lấy URL WebSocket.
3.  **Thực thi lệnh:**
    *   AI viết code Python sử dụng các hàm trong `helpers.py` (ví dụ: `goto_url`, `click_at_xy`).
    *   `run.py` nhận code này và thực thi bằng hàm `exec()`.
    *   Các hàm helper đóng gói yêu cầu thành JSON (ví dụ: `{"method": "Page.navigate", "params": {"url": "..."}}`) và gửi vào Unix Socket.
4.  **Chuyển tiếp CDP:** Daemon nhận gói tin từ Socket, chuyển tiếp nguyên văn tới trình duyệt qua WebSocket của CDP.
5.  **Phản hồi & Tự học:**
    *   Trình duyệt thực hiện lệnh và trả về kết quả.
    *   AI chụp ảnh màn hình (`capture_screenshot`) để xác nhận kết quả.
    *   Nếu gặp rào cản, AI tra cứu `domain-skills/` hoặc tự viết thêm helper mới vào `helpers.py`.

### Tổng kết
Browser Harness không phải là một thư viện Automation truyền thống mà là một **"Môi trường sinh tồn"** cho AI trong trình duyệt. Nó cung cấp các nguyên tử (atoms) lệnh thấp cấp và cho phép AI tự xây dựng các phân tử (molecules) logic phức tạp hơn.