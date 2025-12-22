
## Husky (Git Hooks)

### 1. Husky là gì?
**Husky** là một công cụ giúp quản lý và tự động hóa các **Git Hooks**. Git Hooks là các kịch bản (scripts) mà Git sẽ tự động chạy mỗi khi có một sự kiện nhất định xảy ra, chẳng hạn như trước khi bạn `commit` code hoặc trước khi bạn `push` code lên server.

### 2. Nó giải quyết vấn đề gì?
Trong một nhóm làm việc, mỗi người có một kiểu viết code khác nhau, dễ dẫn đến việc code lỗi hoặc code "xấu" bị đẩy lên kho lưu trữ chung. Husky giúp ép buộc mọi thành viên phải tuân thủ các tiêu chuẩn (như chạy lint, chạy test) trước khi code được phép lưu lại.

### 3. Các tính năng chính:
*   **Chặn Code lỗi (Pre-commit):** Tự động chạy lệnh kiểm tra lỗi (Linting) hoặc Unit Test ngay khi bạn gõ `git commit`. Nếu có lỗi, lệnh commit sẽ bị hủy.
*   **Kiểm tra tin nhắn Commit (Commit-msg):** Đảm bảo tiêu đề commit phải đúng định dạng quy định của dự án.
*   **Dễ dàng chia sẻ:** Cấu hình Husky được lưu trong `package.json`, giúp tất cả mọi người trong team đều có chung một bộ quy tắc kiểm tra tự động.
*   **Tích hợp tốt với lint-staged:** Chỉ kiểm tra những file bạn vừa thay đổi, giúp tiết kiệm thời gian tối đa.

### 4. Cách sử dụng phổ biến
Sau khi cài đặt, Husky sẽ tự động can thiệp vào các lệnh Git:
*   `git commit -m "feat: add login"` -> Husky sẽ nhảy ra chạy lệnh `npm test` trước, nếu test pass thì mới cho commit.

### Lưu ý phân biệt:
Husky không phải là công cụ kiểm tra code, nó chỉ là **"người gác cổng"** gọi các công cụ khác (như ESLint, Prettier, Jest) ra làm việc vào đúng thời điểm bạn thao tác với Git.

**Tóm lại:** Đây là công cụ đảm bảo tính kỷ luật và chất lượng mã nguồn, ngăn chặn code lỗi bị đưa vào hệ thống một cách tự động.