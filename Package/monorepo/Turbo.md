## Turborepo
### 1. Turborepo là gì?
**Turborepo** là một hệ thống build (build system) hiệu suất cao dành cho các dự án **JavaScript và TypeScript monorepo** (nhiều dự án/package nằm trong cùng một kho lưu trữ). Nó được phát triển bởi **Vercel** (công ty đứng sau Next.js).

### 2. Nó giải quyết vấn đề gì?
Khi bạn có một dự án lớn với nhiều folder con (ví dụ: `apps/web`, `apps/mobile`, `packages/ui`), việc chạy các lệnh như `build`, `test`, hoặc `lint` cho toàn bộ dự án sẽ rất chậm. Turborepo giúp tối ưu hóa việc này.

### 3. Các tính năng chính của phiên bản 2.x:
*   **Caching (Local & Remote):** Nếu bạn đã build một phần code và không thay đổi gì ở đó, Turborepo sẽ không build lại mà lấy kết quả từ cache ra ngay lập tức.
*   **Parallel Execution:** Nó tự động tính toán đồ thị phụ thuộc (dependency graph) và chạy các tác vụ song song một cách tối ưu nhất mà không gây xung đột.
*   **Task Pipelines:** Cho phép bạn định nghĩa thứ tự chạy các lệnh (ví dụ: phải build `packages/ui` xong rồi mới build `apps/web`).
*   **Terminal UI mới:** Ở phiên bản 2.x, Turborepo cung cấp một giao diện dòng lệnh trực quan và dễ theo dõi hơn khi chạy nhiều tác vụ cùng lúc.

### 4. Cách sử dụng phổ biến
Nếu bạn thấy package này trong project, thường bạn sẽ sử dụng nó thông qua các câu lệnh như:
*   `npx turbo run build`: Chạy lệnh build cho tất cả project con.
*   `npx turbo run dev`: Chạy chế độ phát triển cho các project.

### Lưu ý phân biệt:
Đôi khi có sự nhầm lẫn với **Turbo (Hotwire)** (một công cụ của Ruby on Rails để tăng tốc độ tải trang). Tuy nhiên, nếu nó nằm trong file `package.json` dưới tên `"turbo"` và có phiên bản `2.6.3`, thì chắc chắn đó là **Turborepo** của Vercel.

**Tóm lại:** Đây là công cụ giúp quản lý dự án monorepo, giúp việc build và phát triển code nhanh hơn rất nhiều nhờ vào việc lưu bộ nhớ đệm (caching) thông minh.