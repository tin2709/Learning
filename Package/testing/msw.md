
## MSW (Mock Service Worker)
### 1. MSW là gì?
**MSW** là một thư viện dùng để giả lập API (API Mocking) bằng cách chặn các yêu cầu mạng ở tầng Service Worker của trình duyệt hoặc tầng mạng của Node.js.

### 2. Nó giải quyết vấn đề gì?
Khi làm Frontend, bạn thường bị phụ thuộc vào API Backend. Nếu Backend lỗi hoặc chưa xong, bạn sẽ bị đình trệ. MSW giúp bạn tạo ra một server ảo "thật" đến mức trình duyệt không hề biết đó là giả, giúp bạn code và test mà không cần server thật.

### 3. Các tính năng chính:
*   **Chặn ở tầng mạng:** Không giống như việc thay đổi hàm `fetch`, MSW chặn trực tiếp yêu cầu gửi đi, giúp môi trường test giống hệt thực tế.
*   **Dùng chung code:** Một bộ code giả lập có thể dùng cho cả lúc lập trình (Browser) và lúc chạy test (Vitest/Jest).
*   **Hỗ trợ đa dạng:** Làm việc tốt với cả REST API và GraphQL.

### 4. Cách sử dụng phổ biến
*   Sử dụng trong file cấu hình test để giả lập các phản hồi từ máy chủ khi chạy Unit Test.

**Tóm lại:** Đây là tiêu chuẩn vàng hiện nay cho việc giả lập dữ liệu, giúp quy trình phát triển Frontend không bao giờ bị gián đoạn.

