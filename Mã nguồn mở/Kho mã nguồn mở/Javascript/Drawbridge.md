Dưới đây là phân tích chi tiết về dự án **Drawbridge** — một công cụ kết nối trải nghiệm duyệt web (UI Feedback) với quy trình lập trình bằng AI (Cursor/Claude Code).

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án này là một sự kết hợp giữa Browser Extension, Hệ thống tệp cục bộ và Prompt Engineering:

*   **Chrome Extension (Manifest V3):** Sử dụng các API hiện đại như `service_worker` để xử lý ngầm và `content_scripts` để tương tác trực tiếp với DOM của trang web.
*   **File System Access API:** Đây là công nghệ then chốt cho phép trình duyệt (Chrome Extension) xin quyền truy cập và ghi trực tiếp vào một thư mục dự án cục bộ trên máy tính người dùng. Điều này xóa bỏ rào cản giữa "Web" và "Local Disk".
*   **Native Screenshot API (`captureVisibleTab`):** Thay vì dùng các thư viện JS nặng nề như `html2canvas` (dễ gây treo trình duyệt), Drawbridge sử dụng API gốc của Chrome để chụp ảnh viewport nhanh chóng và mượt mà, ngay cả với nội dung đang streaming.
*   **IndexedDB:** Sử dụng để lưu trữ các "Directory Handles" (con trỏ thư mục) giúp duy trì kết nối giữa extension và folder dự án qua các phiên làm việc, giảm bớt việc phải cấp quyền lại liên tục.
*   **Node.js (CLI Tooling):** Cung cấp script `moat-watcher.js` để theo dõi thay đổi tệp theo thời gian thực và tự động đưa dữ liệu vào clipboard cho người dùng.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Drawbridge được thiết kế theo mô hình **"Communication through Files"** (Giao tiếp thông qua tệp tin):

*   **Thư mục `.moat` làm trung tâm:** Đây là "vùng đệm" (buffer) chứa mọi dữ liệu giao tiếp. Extension ghi vào đây, và AI (Cursor/Claude) đọc từ đây.
*   **Hệ thống tệp kép (Two-file System):**
    *   `moat-tasks.md`: Dành cho con người (Designer/Dev) để theo dõi danh sách công việc dưới dạng checklist trực quan.
    *   `moat-tasks-detail.json`: Dành cho máy (AI) chứa metadata kỹ thuật: CSS Selector chính xác, tọa độ hình chữ nhật (`boundingRect`), và đường dẫn ảnh chụp màn hình.
*   **Kiến trúc Agent-First:** Khác với các công cụ note thông thường, Drawbridge đi kèm với các tệp `.mdc` (Cursor Rules) hoặc `.md` (Claude Workflow). Đây là các "bản hướng dẫn vận hành" dạy cho AI cách đọc tệp JSON, cách xử lý các Task theo thứ tự phụ thuộc, và cách cập nhật trạng thái Task.
*   **Vòng đời trạng thái (Status Lifecycle):** Quy trình nghiêm ngặt `to do` $\rightarrow$ `doing` $\rightarrow$ `done` giúp AI không bao giờ bị lạc lối khi xử lý hàng loạt yêu cầu cùng lúc.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Atomic Batched Updates:** Trong tệp `drawbridge-workflow.md`, dự án ép buộc AI phải thực hiện cập nhật tệp theo lô (batch). Thay vì gọi 5-6 công cụ riêng lẻ, AI thực hiện cập nhật trạng thái và ghi log trong cùng một lần thao tác tệp để tránh xung đột dữ liệu (race conditions) và tăng 50% hiệu suất.
*   **Coordinate Scaling:** Xử lý chụp ảnh màn hình trên các màn hình DPI cao (Retina) bằng cách nhân tọa độ CSS với `window.devicePixelRatio` để đảm bảo ảnh cắt (crop) luôn chính xác đến từng pixel.
*   **Smart Selector Generation:** Kỹ thuật lấy CSS Selector thông minh, ưu tiên ID và Class đặc trưng để AI có thể tìm thấy chính xác đoạn code cần sửa trong hàng ngàn dòng mã nguồn.
*   **Deduplication (Chống trùng lặp):** `TaskStore.js` kiểm tra nội dung comment và selector. Nếu người dùng click lại cùng một chỗ với cùng một yêu cầu, nó sẽ cập nhật timestamp thay vì tạo task mới rác.
*   **Graceful Degradation:** Nếu việc chụp ảnh màn hình lỗi, hệ thống vẫn cho phép lưu Task với text và selector để không làm gián đoạn luồng làm việc.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Quy trình được chia thành 3 giai đoạn khép kín:

#### Giai đoạn 1: Chụp bối cảnh (Browser)
1.  Người dùng nhấn `C` (Comment) hoặc `R` (Rectangle).
2.  Extension bắt sự kiện Click hoặc Drag trên DOM.
3.  Hệ thống tính toán Selector hoặc Tọa độ.
4.  Gọi `captureVisibleTab` để lấy ảnh viewport.
5.  Ghi dữ liệu vào `.moat/moat-tasks-detail.json` và cập nhật `.moat/moat-tasks.md`.

#### Giai đoạn 2: Phân tích và Phân loại (AI Agent)
1.  Người dùng chạy lệnh `bridge` trong Cursor hoặc `/bridge` trong Claude Code.
2.  AI đọc tệp `.moat/drawbridge-workflow.md` để hiểu quy trình.
3.  AI phân tích tệp JSON để xác định các phụ thuộc (ví dụ: Task "Làm nút này màu xanh" phải làm trước Task "Thêm hiệu ứng đổ bóng cho cái nút xanh đó").
4.  AI chọn chế độ: **Step** (từng bước), **Batch** (theo nhóm) hoặc **YOLO** (tự động hoàn toàn).

#### Giai đoạn 3: Thực thi và Phản hồi (IDE)
1.  AI chuyển trạng thái Task sang `doing`.
2.  AI tìm tệp nguồn dựa trên Selector và thực hiện chỉnh sửa code (CSS/JSX/HTML).
3.  AI chuyển trạng thái Task sang `done` và đánh dấu `[x]` vào tệp Markdown.
4.  Người dùng quay lại trình duyệt, trang web tự động reload (Hot Reload) để kiểm tra kết quả.

### Tổng kết
Drawbridge không chỉ là một công cụ ghi chú, nó là một **Giao thức cộng tác Visual-to-Code**. Nó tận dụng khả năng đọc hiểu ngữ cảnh của AI để biến các "lời phàn nàn" trực quan của designer thành các "Pull Request" thực thụ trong code.