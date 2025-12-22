
## SST (Serverless Stack)
### 1. SST là gì?
**SST** là một khung làm việc (framework) hiện đại giúp xây dựng và triển khai các ứng dụng full-stack (gồm cả frontend và backend) trên nền tảng đám mây AWS một cách dễ dàng nhất.

### 2. Nó giải quyết vấn đề gì?
Việc cấu hình các dịch vụ đám mây (như Lambda, API Gateway, S3) thường rất phức tạp và khó khăn. SST biến việc cấu hình hạ tầng thành việc viết code đơn giản (Infrastructure as Code) và hỗ trợ chế độ phát triển trực tiếp (Live Lambda Development).

### 3. Các tính năng chính:
*   **Live Lambda:** Cho phép bạn sửa code backend và thấy kết quả ngay lập tức trên đám mây mà không cần chờ deploy lại.
*   **Type-safe:** Kết nối giữa frontend và backend được kiểm tra kiểu dữ liệu chặt chẽ.
*   **Tích hợp sẵn:** Hỗ trợ tốt các framework như Next.js, Remix, Astro.

### 4. Cách sử dụng phổ biến
*   `sst dev`: Bắt đầu môi trường phát triển cục bộ nhưng kết nối trực tiếp với AWS.
*   `sst deploy`: Triển khai toàn bộ ứng dụng lên môi trường thực tế.

**Tóm lại:** Đây là công cụ giúp trải nghiệm làm việc với AWS trở nên dễ dàng như khi bạn đang code trên máy tính cá nhân.

