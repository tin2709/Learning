## Rimraf

### 1. Rimraf là gì?
**Rimraf** đơn giản là một gói thư viện cung cấp lệnh xóa thư mục theo kiểu `rm -rf` (xóa sạch bất chấp thư mục có chứa gì).

### 2. Nó giải quyết vấn đề gì?
Lệnh xóa thư mục trên Windows (`rd /s /q`) và Linux/macOS (`rm -rf`) là hoàn toàn khác nhau. Nếu bạn viết lệnh xóa trong `package.json` theo kiểu Linux, nó sẽ lỗi trên Windows. Rimraf giúp lệnh xóa chạy được trên **mọi hệ điều hành**.

### 3. Các tính năng chính:
*   **Cross-platform:** Viết một lần, chạy được cả trên Windows, macOS và Linux.
*   **Deep Deletion:** Xóa sạch mọi thư mục con và file bên trong một cách nhanh chóng.

### 4. Cách sử dụng phổ biến
Thường dùng trong các lệnh "clean" trước khi build lại dự án:
*   `rimraf dist`: Xóa sạch thư mục `dist` cũ để chuẩn bị cho lần build mới.

**Tóm lại:** Đây là công cụ "xóa rác" đa nền tảng cho lập trình viên.
