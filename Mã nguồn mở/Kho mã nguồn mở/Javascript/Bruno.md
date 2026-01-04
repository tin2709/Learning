Dựa trên nội dung các file mã nguồn và cấu trúc thư mục của dự án **Bruno**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của dự án này:

---

### 1. Công nghệ cốt lõi (Core Technology)

Bruno được xây dựng như một ứng dụng desktop hiện đại, sử dụng mô hình Monorepo để quản lý nhiều gói (packages) khác nhau:

*   **Framework chính:** 
    *   **Electron:** Dùng để đóng gói ứng dụng web thành ứng dụng desktop, cho phép truy cập vào hệ thống file cục bộ.
    *   **React (v19):** Thư viện UI chính để xây dựng giao diện người dùng.
*   **Quản lý trạng thái (State Management):** **Redux Toolkit**. Đây là "đầu não" điều khiển toàn bộ dữ liệu về collections, tabs, và các cài đặt của người dùng.
*   **Giao diện (Styling):**
    *   **Tailwind CSS:** Dùng cho bố cục (layout) và các tiện ích nhanh.
    *   **Styled Components:** Dùng để quản lý theme (Dark/Light) và đóng gói logic CSS vào các component.
*   **Trình soạn thảo mã (Code Editor):** **CodeMirror (v5)**. Được tùy biến mạnh mẽ để hỗ trợ syntax highlighting cho định dạng file `.bru` và các script JavaScript.
*   **Công cụ build:** **RSBuild (Rspack)**. Thay vì Webpack truyền thống, Bruno sử dụng RSBuild để đạt tốc độ build và hot-reload cực nhanh.
*   **Giao tiếp mạng:** **Axios** (cho REST/HTTP), hỗ trợ thêm **gRPC** và **WebSocket**.
*   **Định dạng dữ liệu đặc trưng:** **Bru** - một ngôn ngữ đánh dấu (markup language) dạng văn bản thuần túy do dự án tự phát triển để lưu trữ thông tin API.

---

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của Bruno tập trung vào triết lý **"Local-first"** và **"Git-friendly"**:

*   **Filesystem-Centric (Lấy hệ thống file làm trung tâm):** Khác với Postman lưu dữ liệu trong DB nội bộ hoặc Cloud, Bruno lưu trực tiếp các Collection thành các thư mục và file trên ổ cứng. Mỗi request là một file `.bru`.
*   **Offline-Only:** Không có đồng dịch chuyển (cloud sync). Bruno ưu tiên quyền riêng tư và tốc độ truy cập cục bộ.
*   **Kiến trúc Monorepo:** Chia nhỏ ứng dụng thành hơn 15 packages (ví dụ: `bruno-app` cho UI, `bruno-cli` cho dòng lệnh, `bruno-lang` để xử lý ngôn ngữ Bru). Điều này giúp dễ dàng bảo trì và tái sử dụng logic (ví dụ CLI và App dùng chung parser).
*   **Version Control Integration:** Vì mọi thứ là file văn bản thuần túy, người dùng có thể sử dụng Git, SVN để quản lý phiên bản API, làm việc nhóm (merge, branch) giống như quản lý mã nguồn phần mềm.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Custom DSL Parser (`bruno-lang`):** Kỹ thuật chuyển đổi từ UI (JSON) sang định dạng Bru và ngược lại. Parser này cho phép đọc/ghi dữ liệu API một cách có cấu trúc nhưng vẫn dễ đọc với con người.
*   **Sandbox Scripting:** Sử dụng các kỹ thuật sandbox (như `vm` trong Node.js hoặc QuickJS) để chạy các script `Pre-request` và `Post-response` của người dùng một cách an toàn mà không làm sập ứng dụng chính.
*   **Variable Interpolation:** Kỹ thuật xử lý biến (ví dụ: `{{host}}`). Bruno quét và thay thế các biến này dựa trên môi trường (Environment) đã chọn trước khi thực hiện request.
*   **IPC (Inter-Process Communication):** Kỹ thuật giao tiếp giữa tiến trình Electron (Main) và React (Renderer). Tiến trình Main xử lý việc đọc/ghi file và thực hiện request mạng thực tế để tránh các giới hạn về CORS trên trình duyệt.
*   **Polling & Watcher:** Sử dụng thư viện như `chokidar` để theo dõi sự thay đổi của file trên đĩa cứng. Nếu bạn sửa file `.bru` bằng VS Code, UI của Bruno sẽ tự động cập nhật ngay lập tức.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Dựa trên file `README.md` và code thực tế, luồng hoạt động của Bruno như sau:

1.  **Khởi tạo:** Người dùng chọn một thư mục trên máy tính để tạo **Collection**. Bruno sẽ tạo file `bruno.json` để định danh cấu trúc.
2.  **Soạn thảo:**
    *   Người dùng tạo Request trên giao diện React.
    *   Dữ liệu được lưu vào Redux Store.
    *   Thông qua `bruno-filestore`, dữ liệu JSON từ Redux được `bruno-lang` chuyển đổi thành định dạng văn bản Bru và ghi xuống file `.bru`.
3.  **Thực thi request:**
    *   Khi nhấn **"Send"**, ứng dụng thu thập dữ liệu từ file/Redux.
    *   Chạy script **Pre-request** (nếu có).
    *   Thay thế các biến môi trường vào URL/Header/Body.
    *   Gửi request thông qua tiến trình Electron (để bỏ qua rào cản trình duyệt).
4.  **Xử lý phản hồi:**
    *   Nhận kết quả từ Server.
    *   Chạy script **Tests** và các script hậu xử lý.
    *   Hiển thị dữ liệu (JSON, HTML, Image, v.v.) lên màn hình thông qua các Renderer tương ứng.
5.  **Cộng tác:** Người dùng sử dụng Git để `commit` thư mục chứa các file `.bru` lên GitHub/GitLab. Đồng nghiệp `pull` về và Bruno sẽ tự động load các request mới đó từ hệ thống file.

---

### Kết luận
Bruno là một dự án có kiến trúc **mạch lạc, hiện đại**, giải quyết tốt vấn đề "khóa dữ liệu" (vendor lock-in) của các công cụ API khác bằng cách trả quyền kiểm soát file về tay lập trình viên. Việc sử dụng **RSBuild/Rspack** cho thấy dự án rất chú trọng đến hiệu suất phát triển.