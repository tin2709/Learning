Dựa trên mã nguồn và tài liệu bạn cung cấp, dưới đây là bản phân tích chi tiết về dự án **EcoPaste** bằng tiếng Việt, bao gồm công nghệ, kiến trúc và luồng hoạt động.

---

### 1. Tổng quan dự án (README)
**EcoPaste** là một công cụ quản lý Clipboard (bộ nhớ tạm) mã nguồn mở, đa nền tảng (Windows, macOS, Linux).
*   **Triết lý:** Nhẹ, hiệu quả, ưu tiên quyền riêng tư (dữ liệu lưu cục bộ) và trải nghiệm người dùng mượt mà.
*   **Tính năng chính:** Hỗ trợ nhiều định dạng (văn bản, hình ảnh, file, HTML, RTF), tìm kiếm nhanh, gắn ghi chú, phân loại yêu thích, và tùy biến phím tắt.

---

### 2. Công nghệ cốt lõi (Tech Stack)

Dự án sử dụng mô hình Hybrid giữa Web và Native:

#### Frontend (Giao diện người dùng):
*   **Framework:** React 18 với TypeScript.
*   **Build Tool:** Vite (đảm bảo tốc độ phản hồi cực nhanh khi phát triển).
*   **UI Library:** **Ant Design (antd)** kết hợp với **UnoCSS** (Atomic CSS giúp tối ưu dung lượng CSS).
*   **State Management:** **Valtio** (Sử dụng proxy-state, rất nhẹ và hiệu quả cho ứng dụng desktop).
*   **Hooks:** ahooks (bộ sưu tập các hook hữu ích cho React).
*   **Internationalization (i18n):** i18next (hỗ trợ đa ngôn ngữ: Trung, Anh, Nhật).

#### Backend & Core (Tầng hệ thống):
*   **Core Engine:** **Tauri v2** (viết bằng Rust). Thay vì dùng Electron nặng nề, Tauri sử dụng Webview có sẵn của hệ điều hành, giúp giảm kích thước file cài đặt và mức tiêu thụ RAM.
*   **Database:** SQLite (thông qua plugin `tauri-plugin-sql`) kết hợp với **Kysely** (một Query Builder cho TypeScript) để thao tác dữ liệu an toàn về kiểu dữ liệu.
*   **Ngôn ngữ hệ thống:** Rust (xử lý các tác vụ cấp thấp như lắng nghe Clipboard, quản lý cửa sổ, phím tắt toàn cục).

---

### 3. Tư duy kiến trúc (Architectural Thinking)

Dự án được tổ chức theo mô hình **Plugin-based Architecture** và **Local-first**:

1.  **Tách biệt Frontend/Backend:** Tầng giao diện (React) chỉ lo hiển thị. Các logic nặng hoặc tương tác OS (như tự động khởi chạy, dán nội dung, phím tắt) được đẩy xuống Rust thông qua các Custom Plugins (`eco-window`, `eco-paste`, `eco-autostart`).
2.  **Quản lý trạng thái phân tán:** Sử dụng `Valtio` để đồng bộ cấu hình giữa các cửa sổ (Main window và Preference window) thông qua sự kiện `STORE_CHANGED`.
3.  **Thiết kế hướng Module:** 
    *   `src/components`: Các UI component dùng chung.
    *   `src/hooks`: Chứa logic nghiệp vụ như `useClipboard` (lắng nghe thay đổi), `useHistoryList` (tương tác DB).
    *   `src/plugins`: Cầu nối (Bridge) gọi các hàm Rust từ JavaScript.
4.  **Kiến trúc Đa cửa sổ:** Phân chia rõ ràng giữa cửa sổ chính (Main - hiển thị danh sách clipboard) và cửa sổ cài đặt (Preference).

---

### 4. Các kỹ thuật chính (Key Techniques)

*   **Lắng nghe Clipboard thông minh:** Sử dụng plugin `tauri-plugin-clipboard-x` để theo dõi sự thay đổi của bộ nhớ tạm. Hệ thống tự động phân loại nội dung: Nếu là URL -> subtype là `url`, nếu là mã màu -> subtype là `color`, nếu là đường dẫn file -> subtype là `path`.
*   **Xử lý nội dung RTF/HTML:** Sử dụng thư viện `rtf.js` để render định dạng Rich Text ngay trong Webview. Dùng `DOMPurify` để làm sạch HTML trước khi hiển thị để tránh lỗi bảo mật XSS.
*   **Ảo hóa danh sách (Virtual List):** Sử dụng `react-virtuoso` để hiển thị hàng nghìn mục clipboard mà không làm chậm ứng dụng (chỉ render những mục đang hiển thị trên màn hình).
*   **Native Window Injection:** Trên macOS, ứng dụng sử dụng `tauri-nspanel` để biến cửa sổ thành một "Panel", cho phép nó nổi trên các ứng dụng khác và xuất hiện trên mọi Workspace.
*   **Tối ưu hóa dữ liệu:** Hình ảnh không lưu trực tiếp vào DB mà được lưu vào thư mục `images` cục bộ, DB chỉ lưu đường dẫn.

---

### 5. Tóm tắt luồng hoạt động (Workflow Summary)

#### Luồng 1: Sao chép dữ liệu (Copy)
1.  Người dùng nhấn `Ctrl+C` trong bất kỳ ứng dụng nào.
2.  Tầng Rust (Backend) nhận tín hiệu thay đổi clipboard.
3.  Hook `useClipboard` trong React nhận dữ liệu mới.
4.  Dữ liệu được xử lý (trích xuất text, lưu ảnh nếu có, xác định loại dữ liệu).
5.  Lưu vào SQLite (bảng `history`).
6.  Cập nhật state `Valtio` -> Giao diện tự động thêm mục mới vào đầu danh sách.

#### Luồng 2: Sử dụng lại dữ liệu (Paste)
1.  Người dùng nhấn phím tắt (mặc định `Alt+C`) để mở EcoPaste.
2.  Người dùng chọn một mục trong danh sách (Click hoặc phím mũi tên).
3.  Khi người dùng nhấn `Enter` hoặc `Double Click`:
    *   Gọi hàm `writeToClipboard` để nạp lại dữ liệu đó vào clipboard hệ thống.
    *   Gọi hàm `paste` (thông qua Rust plugin) để thực hiện lệnh dán giả lập vào ứng dụng đang active trước đó.
4.  Cửa sổ EcoPaste tự động ẩn đi (tùy cài đặt).

---

**Đánh giá:** Đây là một dự án có mã nguồn rất sạch sẽ, áp dụng những công nghệ hiện đại nhất trong hệ sinh thái Tauri. Việc sử dụng Rust cho các tác vụ hệ thống giúp ứng dụng có hiệu năng cực cao mà vẫn giữ được sự linh hoạt trong phát triển giao diện bằng React.