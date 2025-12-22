
## Bumpp
### 1. Bumpp là gì?
**Bumpp** là một công cụ dòng lệnh (CLI) cực kỳ mạnh mẽ và trực quan dùng để tăng số phiên bản (versioning) cho các dự án JavaScript/TypeScript.

### 2. Nó giải quyết vấn đề gì?
Việc cập nhật thủ công số phiên bản trong file `package.json`, sau đó tạo Git tag và commit thường rất dễ sai sót. Bumpp tự động hóa toàn bộ quy trình này chỉ với một lệnh duy nhất.

### 3. Các tính năng chính:
*   **Giao diện tương tác:** Cho phép bạn chọn loại cập nhật (Patch, Minor, Major) thông qua phím mũi tên.
*   **Hỗ trợ Monorepo:** Tự động phát hiện và cập nhật phiên bản cho tất cả các package con cùng lúc.
*   **Tích hợp Git:** Tự động tạo commit, gắn thẻ (tag) và đẩy code lên server nếu bạn yêu cầu.

### 4. Cách sử dụng phổ biến
*   `npx bumpp`: Chạy giao diện tương tác để chọn phiên bản mới.
*   `bumpp -r`: Cập nhật phiên bản theo kiểu đệ quy cho toàn bộ monorepo.

**Tóm lại:** Đây là công cụ giúp quy trình phát hành (release) phiên bản mới trở nên nhanh chóng và chuyên nghiệp hơn.
