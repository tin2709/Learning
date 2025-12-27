Dưới đây là bản phân tích chi tiết về dự án **drawDB** dựa trên mã nguồn và tài liệu bạn đã cung cấp:

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

*   **Frontend Framework:** **React** (sử dụng **Vite** để build). Đây là lựa chọn tối ưu cho hiệu năng và tốc độ phát triển.
*   **Quản lý trạng thái (State Management):** Sử dụng **React Context API** một cách triệt để (thư mục `src/context/`). Thay vì dùng Redux, dự án chia nhỏ trạng thái thành các Context riêng biệt (Areas, Canvas, Diagram, UndoRedo, v.v.) để giảm việc render lại không cần thiết.
*   **Giao diện & UI:**
    *   **Semi UI (của ByteDance):** Bộ thư viện component chính (`@douyinfe/semi-ui`).
    *   **Tailwind CSS v4:** Dùng cho styling linh hoạt và hiện đại.
    *   **Framer Motion:** Xử lý các hiệu ứng chuyển động (FadeIn, SlideIn).
*   **Xử lý đồ họa:** **SVG (Scalable Vector Graphics)**. Toàn bộ sơ đồ được vẽ bằng SVG thay vì Canvas, điều này giúp các đối tượng (Table, Relationship) dễ dàng tương tác (DOM events) và xuất ảnh chất lượng cao.
*   **Lưu trữ dữ liệu:** **Dexie.js** (một wrapper của IndexedDB). Điều này cho phép ứng dụng hoạt động "Local-first" - lưu dữ liệu ngay trên trình duyệt người dùng mà không cần tài khoản.
*   **Xử lý ngôn ngữ SQL/DBML:** 
    *   `@dbml/core`: Để chuyển đổi định dạng DBML.
    *   `node-sql-parser` & `oracle-sql-parser`: Để phân tích cú pháp SQL khi import.
*   **Môi trường:** Hỗ trợ **Docker** để triển khai nhanh chóng.

---

### 2. Tư duy kiến trúc (Architectural Thinking)

*   **Component-Based Architecture:** Chia nhỏ UI thành các phần chức năng độc lập: `EditorHeader`, `EditorSidePanel`, `EditorCanvas`.
*   **Local-First & Privacy:** Kiến trúc được thiết kế để ưu tiên quyền riêng tư. Dữ liệu mặc định nằm ở IndexedDB của trình duyệt. Chỉ khi người dùng muốn "Share", dữ liệu mới được đẩy lên server (thông qua GitHub Gists).
*   **Action-Based Undo/Redo:** Sử dụng một ngăn xếp (stack) để lưu trữ các "Action". Mỗi khi người dùng thay đổi (di chuyển bảng, đổi tên trường), một bản ghi "Undo" được tạo ra, cho phép quay lại trạng thái trước đó một cách chính xác.
*   **Schema-Driven UI:** Giao diện bên thanh Sidebar (TableTab, RelationshipTab) được đồng bộ hóa trực tiếp với Schema hiện tại của Diagram. Khi bạn sửa ở Sidebar, Canvas cập nhật và ngược lại.
*   **Internationalization (i18n):** Kiến trúc hỗ trợ đa ngôn ngữ ngay từ đầu với `i18next`, cho phép mở rộng sang hàng chục ngôn ngữ khác nhau (có cả tiếng Việt).

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **SVG Manipulation:** Sử dụng `<foreignObject>` trong SVG để nhúng các thẻ HTML (như Input, Div) vào trong sơ đồ. Đây là kỹ thuật khó giúp kết hợp sức mạnh vẽ của SVG và khả năng tương tác của HTML.
*   **Dynamic Relationship Path:** Kỹ thuật tính toán đường cong (Path) giữa các bảng dựa trên vị trí X, Y. Khi bảng di chuyển, các đường nối (Relationship) tự động tính toán lại tọa độ để không bị chồng chéo (thông qua `calcPath.js`).
*   **Snapping to Grid:** Kỹ thuật căn chỉnh bảng vào lưới (grid) khi di chuyển để sơ đồ trông gọn gàng hơn.
*   **SQL Generation Logic:** Một hệ thống các hàm tiện ích (`src/utils/exportSQL/`) giúp chuyển đổi đối tượng JSON (đại diện cho bảng/cột) thành mã SQL chuẩn cho từng hệ quản trị (MySQL, PostgreSQL, SQLite, v.v.).
*   **Phân tích SQL ngược (Reverse Engineering):** Kỹ thuật phân tích một đoạn mã SQL có sẵn để tự động vẽ thành sơ đồ (Import SQL).

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Dựa trên file `README.md` và cấu trúc mã nguồn, đây là quy trình hoạt động của dự án:

1.  **Khởi tạo:** Người dùng truy cập trang web, ứng dụng sẽ kiểm tra IndexedDB để tải sơ đồ gần nhất. Nếu chưa có, người dùng chọn hệ quản trị cơ sở dữ liệu mục tiêu (MySQL, Postgres...).
2.  **Thiết kế (Editing):**
    *   Người dùng thêm **Table** (Bảng) trực tiếp trên **Canvas**.
    *   Thêm các **Field** (Trường), định nghĩa kiểu dữ liệu, khóa chính, khóa ngoại.
    *   Tạo **Relationship** bằng cách kéo thả giữa các trường của các bảng.
    *   Có thể thêm **Note** (Ghi chú) hoặc **Subject Area** (Nhóm vùng) để phân loại bảng.
3.  **Quản lý:** Mọi thay đổi được lưu tự động vào bộ nhớ trình duyệt. Người dùng có thể xem lịch sử hoạt động (Timeline) hoặc quản lý danh sách công việc (Todo) ngay trong trình chỉnh sửa.
4.  **Xuất bản (Export):**
    *   **SQL:** Generator sẽ tạo ra script SQL để người dùng chạy trực tiếp trên database thật.
    *   **Image/PDF:** Chuyển đổi SVG thành định dạng ảnh/tài liệu để đưa vào báo cáo.
    *   **Share:** Tạo một link chia sẻ (lưu nội dung sơ đồ lên server/Gist).
5.  **Hợp tác:** Người dùng khác có thể mở link chia sẻ, ứng dụng sẽ tải dữ liệu từ server về và cho phép họ chỉnh sửa hoặc lưu thành bản sao mới.

**Kết luận:** drawDB là một công cụ thiết kế database rất chuyên nghiệp, tập trung vào trải nghiệm người dùng mượt mà (React) và khả năng làm việc độc lập cao (IndexedDB/SVG).

![alt text](image.png)