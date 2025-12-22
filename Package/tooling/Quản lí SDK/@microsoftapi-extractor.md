
## @microsoft/api-extractor
### 1. @microsoft/api-extractor là gì?
Đây là một công cụ cao cấp của Microsoft dành cho các nhà phát triển **thư viện (SDK)** để quản lý và phân tích các tệp định nghĩa TypeScript (.d.ts).

### 2. Nó giải quyết vấn đề gì?
Khi bạn viết một thư viện lớn (như tldraw), các tệp định nghĩa kiểu dữ liệu thường bị phân tán khắp nơi. Công cụ này giúp gom chúng lại thành một file duy nhất, phát hiện các thay đổi làm hỏng API cũ, và tự động tạo tài liệu hướng dẫn.

### 3. Các tính năng chính:
*   **Bundling d.ts:** Gom hàng trăm file kiểu dữ liệu thành một file duy nhất cho người dùng cuối.
*   **Phát hiện rò rỉ API:** Cảnh báo nếu bạn vô tình để lộ các kiểu dữ liệu nội bộ ra ngoài.
*   **Tạo tài liệu:** Kết hợp với API Documenter để tạo trang hướng dẫn sử dụng tự động.

### 4. Cách sử dụng phổ biến
Thường dùng trong giai đoạn chuẩn bị phát hành (publish) một package để đảm bảo chất lượng của bộ SDK.

**Tóm lại:** Đây là công cụ chuyên nghiệp để đảm bảo các thư viện (thành phẩm) của bạn luôn gọn gàng, chính xác và dễ dùng đối với các lập trình viên khác.