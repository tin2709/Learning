Dựa trên mã nguồn và cấu trúc thư mục của **Automad v2.0.0-beta.9** mà bạn cung cấp, dưới đây là phân tích chi tiết về công nghệ cốt lõi, tư duy kiến trúc và luồng hoạt động của dự án này.

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

Automad là một **Flat-file CMS** (CMS không cơ sở dữ liệu), kết hợp giữa sức mạnh xử lý phía server của PHP và giao diện hiện đại dựa trên Web Components.

*   **Backend:** PHP 8.2+. Không sử dụng database (MySQL/PostgreSQL), dữ liệu lưu trữ dưới dạng file `.json` và file văn bản.
*   **Frontend (Dashboard):** 
    *   **Ngôn ngữ:** TypeScript.
    *   **Kiến trúc UI:** Native Web Components (Custom Elements). Không dùng Framework lớn như React/Vue mà kế thừa từ một `BaseComponent` tùy chỉnh.
    *   **Bundler:** `esbuild` (được cấu hình rất chi tiết trong `esbuild.js`).
*   **Template Engine:** Một engine tùy chỉnh cho phép viết logic trực tiếp trong HTML với cú pháp riêng (snippets, pagelists).
*   **Styling:** Less (được biên dịch qua PostCSS và Autoprefixer).
*   **Editor:** Sử dụng **Editor.js** cho trải nghiệm soạn thảo dựa trên khối (block-based editing).

---

### 2. Tư duy kiến trúc và Kỹ thuật nổi bật

#### A. Kiến trúc Flat-file & Tính di động (Portability)
Automad được thiết kế để "chạy ngay" (Plug-and-play). Toàn bộ cấu hình hệ thống và nội dung trang web nằm trong các thư mục `/config`, `/pages`, và `/shared`. File `.htaccess` được cấu hình thông minh để tự động nhận diện URL cơ sở, giúp người dùng có thể di chuyển toàn bộ thư mục dự án sang một server khác hoặc vào thư mục con mà không cần sửa code.

#### B. Hệ thống Web Components tùy chỉnh
Thay vì sử dụng các thư viện UI cồng kềnh, Automad xây dựng các thẻ HTML riêng (ví dụ: `<am-alert>`, `<am-form>`, `<am-editor-js>`). 
*   **`BaseComponent.ts`**: Lớp cơ sở quản lý vòng đời (lifecycle), quản lý sự kiện (event listeners) và thuộc tính (attributes).
*   **Tư duy:** Giảm phụ thuộc vào framework, tận dụng tối đa tiêu chuẩn trình duyệt hiện đại, giúp Dashboard cực nhẹ và nhanh.

#### C. Build Pipeline phức tạp với `esbuild.js`
File `esbuild.js` của dự án không chỉ đơn thuần là đóng gói code. Nó thực hiện các kỹ thuật nâng cao:
*   **Automatic Code Splitting:** Tự động tách các module vendor và các lớp "Block" của EditorJS thành các file riêng biệt.
*   **Hashing System:** Một plugin tùy chỉnh (`hash-imports`) được viết để tạo mã hash cho file JS và cập nhật các đường dẫn import động. Điều này giúp trình duyệt cache file hiệu quả nhưng vẫn cập nhật ngay khi có code mới.
*   **Minification tùy chỉnh:** Ngoài trình thu gọn của esbuild, tác giả còn viết một bộ thu gọn TS/JS thủ công để tối ưu hóa template strings.

#### D. Bảo mật theo kiến trúc "ReadOnly"
Như đã nêu trong `SECURITY.md`, dự án phân loại rõ rệt giữa Admin (có quyền sửa file) và Visitor (chỉ xem). Vì không có database, rủi ro về SQL Injection bằng 0. XSS cũng được hạn chế tối đa vì đầu vào của khách truy cập gần như không bao giờ được lưu trữ trực tiếp vào hệ thống.

---

### 3. Các kỹ thuật kỹ thuật đáng chú ý

1.  **Undo/Redo System (`undo.ts`):** Một hệ thống quản lý trạng thái (State Management) thủ công cho Dashboard, cho phép người dùng quay lại các thao tác sửa đổi dữ liệu trước khi lưu chính thức.
2.  **Binding System (`bindings.ts`):** Một cơ chế phản ứng (reactive) đơn giản. Khi dữ liệu trong một input thay đổi, các thành phần UI khác đang "bind" với dữ liệu đó sẽ tự động cập nhật mà không cần load lại trang.
3.  **Hệ thống xử lý ảnh:** Sử dụng **Filerobot Image Editor** tích hợp sâu vào hệ thống file, cho phép chỉnh sửa ảnh trực tiếp trên trình duyệt và lưu lại server qua API PHP.
4.  **Internationalization (i18n):** Hệ thống dịch thuật dựa trên các file JSON (`english.json`). Dashboard có khả năng thay đổi ngôn ngữ động mà không cần tải lại toàn bộ tài nguyên.

---

### 4. Tóm tắt luồng hoạt động (Workflow)

#### Luồng xử lý yêu cầu (Request Flow):
1.  **Client gửi Request:** Truy cập vào một URL (ví dụ: `/blog/my-post`).
2.  **Server tiếp nhận:** `index.php` gọi `automad/init.php`.
3.  **Routing (PHP):** Lớp `Router.php` xác định xem đây là yêu cầu trang web (View) hay yêu cầu API (Controller).
    *   *Nếu là trang web:* Engine sẽ đọc file `.dat` trong thư mục `/pages`, kết hợp với Template (file `.php` trong theme) để render ra HTML.
    *   *Nếu là Dashboard:* Tải file `automad/dist/build/admin/index.js` để khởi chạy ứng dụng Single Page Application (SPA) bằng TypeScript.

#### Luồng làm việc của Dashboard (Frontend Workflow):
1.  **`am-root` khởi tạo:** Thành phần gốc `<am-root>` kiểm tra trạng thái đăng nhập.
2.  **Routing Dashboard:** Dựa trên URL (ví dụ: `/dashboard?view=Page&url=...`), nó sẽ khởi tạo thành phần tương ứng (ví dụ: `<am-pages>`).
3.  **Data Fetching:** Thành phần gọi API qua `request.ts` (Fetch API).
4.  **Render:** Các Web Components con nhận dữ liệu, render giao diện và gắn các sự kiện.
5.  **Save:** Khi nhấn Save, `am-form` thu thập dữ liệu từ các block, gửi POST request về PHP Controller để ghi đè vào file hệ thống.

### Kết luận
Automad là một dự án có độ tinh xảo kỹ thuật cao (High Engineering Quality). Nó chứng minh rằng không cần các framework nặng nề như React hay cơ sở dữ liệu phức tạp vẫn có thể xây dựng được một hệ thống quản trị nội dung mạnh mẽ, nhanh và cực kỳ linh hoạt nhờ vào việc làm chủ các công nghệ web cốt lõi.