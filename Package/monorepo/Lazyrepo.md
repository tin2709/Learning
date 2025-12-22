

## Lazyrepo
### 1. Lazyrepo là gì?
**Lazyrepo** là một công cụ quản lý tác vụ (task runner) hiệu suất cao dành cho các dự án monorepo. Nó đóng vai trò tương tự như Turborepo hay Nx nhưng tập trung vào sự đơn giản và cấu hình tối thiểu.

### 2. Nó giải quyết vấn đề gì?
Trong các dự án lớn, việc chạy đi chạy lại các lệnh build hay test cho những phần code không hề thay đổi là một sự lãng phí thời gian. Lazyrepo giúp tránh việc này bằng cách ghi nhớ những gì đã làm và chỉ chạy lại những gì thực sự cần thiết.

### 3. Các tính năng chính:
*   **Zero-config:** Có thể hoạt động ngay lập tức mà không cần các file cấu hình phức tạp.
*   **Caching thông minh:** Lưu trữ kết quả của các lần chạy trước (như build artifacts) và tái sử dụng chúng nếu mã nguồn không đổi.
*   **Chạy song song:** Tự động thực thi nhiều tác vụ cùng lúc để tận dụng tối đa sức mạnh của CPU.
*   **Lọc tác vụ (Filtering):** Cho phép chỉ chạy lệnh cho một hoặc một vài project cụ thể trong monorepo.

### 4. Cách sử dụng phổ biến
*   `lazy run build`: Chạy lệnh build cho các package có sự thay đổi.
*   `lazy run test --filter='packages/core'`: Chỉ chạy test cho một package cụ thể.

**Tóm lại:** Đây là một công cụ "lười biếng" theo nghĩa tích cực, giúp tăng tốc độ làm việc bằng cách bỏ qua các công việc dư thừa trong monorepo.


