
## Sherif
### 1. Sherif là gì?
**Sherif** là một trình kiểm tra lỗi (linter) dành riêng cho cấu trúc và sự đồng nhất của các dự án **Monorepo**.

### 2. Nó giải quyết vấn đề gì?
Trong dự án Monorepo, việc giữ cho các package con đồng nhất (ví dụ: cùng phiên bản thư viện, cùng cấu hình) là rất khó. Sherif giúp phát hiện các điểm sai lệch này để tránh các lỗi xung đột phiên bản kỳ lạ.

### 3. Các tính năng chính:
*   **Kiểm tra chéo:** Đảm bảo một thư viện chỉ dùng duy nhất một phiên bản trên toàn bộ monorepo.
*   **Cảnh báo cấu trúc:** Phát hiện các file `package.json` thiếu thông tin hoặc sai định dạng quy định.
*   **Hiệu suất cao:** Chạy cực nhanh ngay cả với những dự án có hàng trăm package con.

### 4. Cách sử dụng phổ biến
*   `npx sherif`: Kiểm tra toàn bộ monorepo để tìm các điểm thiếu đồng nhất.

**Tóm lại:** Đúng như tên gọi (Cảnh sát trưởng), Sherif đảm bảo trật tự và kỷ luật cho hạ tầng mã nguồn của các dự án monorepo phức tạp.