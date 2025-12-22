
## @crxjs/vite-plugin
### 1. CRXJS là gì?
**CRXJS** là một bộ công cụ giúp phát triển tiện ích mở rộng trình duyệt (Chrome Extension) bằng Vite. Nó cho phép bạn sử dụng các công nghệ hiện đại như React, HMR (tải lại nhanh) để làm Extension.

### 2. Nó giải quyết vấn đề gì?
Phát triển Extension truyền thống rất khó khăn vì cấu hình file `manifest.json` phức tạp và không hỗ trợ các tính năng như tự động làm mới trang (Live Reload) một cách mượt mà. CRXJS giúp bạn code Extension giống hệt như code một trang web thông thường.

### 3. Các tính năng chính:
*   **HMR cho Extension:** Tự động cập nhật các thay đổi trong Content Scripts và Background Scripts mà không cần cài đặt lại Extension.
*   **Tự động quản lý Manifest:** Nó tự hiểu và đóng gói các file được khai báo trong `manifest.json`.

### 4. Cách sử dụng phổ biến:
*   Được khai báo làm plugin trong file `vite.config.ts`.

**Tóm lại:** Đây là "chìa khóa" để biến quy trình phát triển Chrome Extension trở nên nhanh chóng và hiện đại.

---
