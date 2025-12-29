Dựa trên cấu trúc thư mục và nội dung mã nguồn bạn cung cấp, đây là phân tích chi tiết về dự án **Pear Desktop** (một phiên bản mở rộng của YouTube Music Desktop App):

### 1. Phân tích Công nghệ cốt lõi (Core Technology Stack)

*   **Framework chính:** **Electron**. Đây là một ứng dụng desktop đa nền tảng chạy trên nền Chromium và Node.js.
*   **Ngôn ngữ:** **TypeScript** (chiếm đa số). Dự án sử dụng TypeScript cực kỳ chặt chẽ với việc định nghĩa các file `.d.ts` và cấu hình `tsconfig.json` tối ưu cho từng tiến trình (Main, Preload, Renderer).
*   **Build Tool:** **Vite** thông qua bộ công cụ `electron-vite`. Điều này giúp tốc độ phát triển (HMR) và đóng gói (bundle) nhanh hơn nhiều so với Webpack truyền thống.
*   **UI Framework (Renderer):** **SolidJS**. Thay vì React, dự án chọn SolidJS để đạt hiệu suất cực cao và dung lượng nhẹ, phù hợp với việc "tiêm" (inject) vào một trang web có sẵn như YouTube Music.
*   **Package Manager:** **pnpm**. Sử dụng tính năng `overrides` và `patchedDependencies` để sửa lỗi trực tiếp trong các thư viện node_modules mà không cần đợi tác giả gốc cập nhật.
*   **UI Components:** **MDUI 2** (Material Design 3). Một thư viện UI hiện đại dựa trên Web Components.

### 2. Kĩ thuật và Tư duy kiến trúc (Architecture & Thinking)

Dự án đi theo mô hình **Plugin-based Architecture** (Kiến trúc dựa trên plugin) rất mạnh mẽ:

*   **Tính mô-đun hóa cực cao:** Mọi tính năng (Lyrics, Adblocker, Discord RPC, Downloader...) đều được tách thành các plugin độc lập trong thư mục `src/plugins/`.
*   **Hệ thống Virtual Modules:** Sử dụng các Vite plugin tùy chỉnh (`plugin-importer.mts`, `i18n-importer.mts`) để tạo ra các module ảo (`virtual:plugins`, `virtual:i18n`). Kỹ thuật này cho phép ứng dụng tự động nhận diện và nạp các plugin/ngôn ngữ mới mà không cần đăng ký thủ công trong mã nguồn chính.
*   **Tư duy "Wrappers":** Thay vì xây dựng trình chơi nhạc từ đầu, dự án "bao bọc" trang web `music.youtube.com` và sử dụng tiến trình **Preload** để can thiệp vào DOM, API của YouTube Music.
*   **Cấu hình phân cấp:** Sử dụng `electron-store` để quản lý cấu hình người dùng, hỗ trợ xem thay đổi thời gian thực (watching) để cập nhật UI/Logic ngay lập tức mà không cần khởi động lại.

### 3. Các kỹ thuật chính (Key Techniques)

*   **In-App Menu Injection:** Thay vì dùng thanh menu chuẩn của hệ điều hành, Pear Desktop sử dụng kỹ thuật ẩn Frame gốc (`frame: false`) và tự vẽ thanh tiêu đề (TitleBar) bằng HTML/CSS để đạt được vẻ ngoài đồng nhất (Native look & feel).
*   **WebRequest Interception:** Sử dụng `@jellybrick/electron-better-web-request` để can thiệp vào các header HTTP, loại bỏ Content Security Policy (CSP) và sửa đổi các yêu cầu mạng để hỗ trợ chặn quảng cáo hoặc proxy.
*   **Trusted Types:** Áp dụng chính sách bảo mật `trusted-types` (trong `src/utils/trusted-types.ts`) để ngăn chặn các cuộc tấn công XSS khi chèn HTML động vào trang web.
*   **Monkey Patching:** Sửa đổi hành vi của các thư viện bên thứ ba thông qua thư mục `patches/`. Ví dụ: Sửa lỗi hiển thị của MDUI hoặc tối ưu hóa thư viện xử lý âm thanh `vudio`.
*   **I18n (Internationalization):** Hệ thống đa ngôn ngữ đồ sộ hỗ trợ hàng chục quốc gia, được tổ chức theo file JSON và nạp động thông qua i18next.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Khởi tạo (Main Process):**
    *   Đọc cấu hình từ người dùng.
    *   Đăng ký các giao thức tùy chỉnh (`peard://`).
    *   Nạp các plugin ở phía "Backend" (như Discord RPC, API Server).
    *   Tạo cửa sổ Electron, thiết lập User Agent giả lập trình duyệt để tránh bị Google chặn.

2.  **Giai đoạn Preload:**
    *   Trước khi trang web YouTube Music tải xong, tập tin `preload.ts` được thực thi.
    *   Thiết lập một "cầu nối" bảo mật (`ContextBridge`) để Renderer có thể giao tiếp với các tính năng của hệ thống (Node.js) một cách an toàn.

3.  **Giai đoạn Renderer (Tiêm mã):**
    *   Khi trang web đã sẵn sàng, tập tin `renderer.ts` sẽ chạy.
    *   Nó chèn các CSS tùy chỉnh để thay đổi giao diện (Theme).
    *   Nó quét tìm API trình phát nhạc của YouTube (`#movie_player`) để lấy thông tin bài hát, điều khiển âm lượng, chuyển bài.
    *   Các plugin phía Renderer sẽ được kích hoạt (ví dụ: hiển thị lời bài hát đồng bộ từ `LRClib`).

4.  **Tương tác người dùng:**
    *   Người dùng điều khiển thông qua phím tắt, khay hệ thống (Tray) hoặc thanh cảm ứng (Touchbar trên macOS).
    *   Mọi thay đổi cấu hình sẽ được lưu vào `config.json` và ứng dụng tự động phản hồi lại các thay đổi đó.

**Kết luận:** Pear Desktop là một dự án Electron mẫu mực về cách sử dụng **TypeScript** và **Vite**. Kiến trúc Plugin của nó cho phép cộng đồng dễ dàng mở rộng tính năng mà không làm ảnh hưởng đến lõi của ứng dụng.