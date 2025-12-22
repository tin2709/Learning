
## Cross-env
### 1. Cross-env là gì?
**Cross-env** là một thư viện nhỏ giúp thiết lập các biến môi trường (environment variables) một cách nhất nhất quán trên mọi hệ điều hành (Windows, macOS, Linux).

### 2. Nó giải quyết vấn đề gì?
Mỗi hệ điều hành có cách thiết lập biến môi trường khác nhau trong terminal (ví dụ Windows dùng `SET`, Linux dùng `export`). Điều này khiến các lệnh trong `scripts` của `package.json` thường bị lỗi khi chuyển giữa các máy tính khác nhau.

### 3. Các tính năng chính:
*   **Đa nền tảng:** Đảm bảo một câu lệnh duy nhất chạy được ở bất cứ đâu.
*   **Dễ sử dụng:** Chỉ cần thêm từ khóa `cross-env` vào trước lệnh thiết lập biến.

### 4. Cách sử dụng phổ biến
*   `"start": "cross-env NODE_ENV=production node server.js"`: Đảm bảo biến `NODE_ENV` luôn được hiểu là `production` dù bạn đang dùng Windows hay Mac.

**Tóm lại:** Đây là giải pháp "phá bỏ rào cản" hệ điều hành khi quản lý các biến môi trường trong dự án.

