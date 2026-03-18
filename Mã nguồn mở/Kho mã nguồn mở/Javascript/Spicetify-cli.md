Dựa trên mã nguồn và cấu trúc project **Spicetify-cli** mà bạn cung cấp, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và tư duy kỹ thuật của dự án này:

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án là một sự kết hợp giữa ngôn ngữ lập trình hệ thống và công nghệ web:

*   **Go (Golang):** Đóng vai trò là "trái tim" của công cụ CLI. Go được chọn nhờ khả năng biên dịch thành file thực thi duy nhất, xử lý file tốc độ cao, và hỗ trợ đa nền tảng (Windows, macOS, Linux) cực tốt.
*   **JavaScript (React):** Các ứng dụng tùy chỉnh (Custom Apps) và tiện ích mở rộng (Extensions) được viết bằng JavaScript, tận dụng thư viện React (vì UI gốc của Spotify cũng dựa trên React/Web-tech).
*   **CSS:** Sử dụng để can thiệp sâu vào giao diện người dùng thông qua kỹ thuật **CSS Injection**.
*   **Shell Script (Bash/PowerShell):** Sử dụng cho bộ cài đặt (installer) để tự động hóa việc cấu hình biến môi trường và tải về các bản build phù hợp với kiến trúc CPU (x64, ARM).

### 2. Kỹ thuật và Tư duy Kiến trúc (Engineering & Architectural Thinking)

Spicetify không cố gắng xây dựng lại Spotify, mà nó hoạt động theo tư duy **"Non-invasive Hooking"** (Can thiệp không xâm lấn quá sâu):

*   **Kiến trúc Command-based:** Dự án sử dụng cấu trúc lệnh phân cấp rõ ràng (như `apply`, `backup`, `restore`, `config`). Mỗi lệnh được đóng gói trong một file riêng trong thư mục `src/cmd/`, giúp dễ dàng bảo trì và mở rộng.
*   **Cơ chế Backup & Restore:** Tư duy kiến trúc cực kỳ an toàn. Trước khi sửa đổi bất kỳ file hệ thống nào của Spotify, Spicetify luôn tạo một bản sao lưu (Backup). Điều này đảm bảo người dùng có thể quay lại trạng thái gốc ngay lập tức nếu Spotify gặp lỗi hoặc cập nhật.
*   **Tính trừu tượng hệ điều hành (OS Abstraction):** Mã nguồn Go xử lý các đường dẫn file và quyền quản trị khác nhau giữa Windows và Unix thông qua các package con trong `src/utils/isAdmin/`, giúp mã nguồn chính luôn sạch sẽ.
*   **Kiến trúc Plugin (Extensions/Custom Apps):** Spicetify tạo ra một môi trường "sandbox" để các nhà phát triển bên thứ ba có thể viết JS/CSS và "nhúng" chúng vào Spotify mà không cần quan tâm đến cách Spicetify vận hành bên dưới.

### 3. Các kỹ thuật chính nổi bật (Key Techniques)

*   **Binary Patching & File Preprocessing:** Đây là kỹ thuật khó nhất. Spicetify tìm kiếm và thay đổi các file tài nguyên (thường là các file `.spa` hoặc bundle JS) của Spotify để chèn code tùy chỉnh vào.
*   **Chromium Embedded Framework (CEF) Interception:** Vì Spotify chạy trên nền CEF, Spicetify kích hoạt các tính năng ẩn như **Remote DevTools**, cho phép lập trình viên mở console (F12) để debug ngay trên app Spotify.
*   **CORS Proxying:** Trong code `lyrics-plus`, ta thấy kỹ thuật sử dụng Proxy để vượt qua rào cản CORS khi fetch lời bài hát từ các nguồn bên ngoài (Musixmatch, Netease) vào bên trong Spotify.
*   **Regex-based Injection:** Sử dụng Regular Expression mạnh mẽ để tìm các điểm mấu chốt trong code gốc của Spotify và "tiêm" (inject) logic mới vào đó (ví dụ: `src/preprocess/preprocess.go`).
*   **Update Blocking:** Kỹ thuật sửa đổi file thực thi hoặc cấu hình hệ thống để chặn Spotify tự động cập nhật, giúp duy trì các bản tùy biến không bị mất đi.

### 4. Tóm tắt luồng hoạt động (Project Workflow)

Luồng hoạt động của Spicetify có thể tóm tắt qua 5 bước chính:

1.  **Backup (Sao lưu):** Khi chạy lần đầu, Spicetify tìm thư mục cài đặt Spotify, copy các file quan trọng sang thư mục lưu trữ riêng.
2.  **Preprocess (Tiền xử lý):** Giải nén các file bundle của Spotify. Quét code để tìm các vị trí cần chèn API hoặc cho phép nạp CSS/JS bên ngoài.
3.  **Config Management (Quản lý cấu hình):** Đọc file `config.ini` để biết người dùng muốn dùng Theme nào, Extensions nào, App nào.
4.  **Injection & Apply (Thực thi):**
    *   Tạo ra các file tài nguyên đã được chỉnh sửa.
    *   Chèn các đoạn script khởi tạo của Spicetify vào luồng load của Spotify.
    *   Ghi đè hoặc trỏ Spotify tới các file đã chỉnh sửa này.
5.  **Execution (Khởi chạy):** Tắt các tiến trình Spotify đang chạy và khởi động lại Spotify với các tham số đặc biệt (như `--remote-debugging-port`) để áp dụng thay đổi.

### Kết luận
**Spicetify-cli** là một dự án **Reverse Engineering (Kỹ thuật ngược)** phần mềm cực kỳ thông minh. Thay vì bẻ khóa (crack), nó chọn cách "cải tạo" (modding) các ứng dụng Electron/CEF bằng cách tận dụng chính các công nghệ web mà ứng dụng đó đang sử dụng. Tư duy của dự án rất chặt chẽ về mặt an toàn dữ liệu và tính linh hoạt cho cộng đồng phát triển thêm (Extensibility).