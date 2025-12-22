

## Lerna

### 1. Lerna là gì?
**Lerna** là một công cụ quản lý các dự án JavaScript có nhiều package (monorepo). Nó giúp điều phối việc kiểm thử, xây dựng và đặc biệt là quy trình **phát hành (publish)** các package lên NPM một cách đồng bộ.

### 2. Nó giải quyết vấn đề gì?
Khi bạn có hàng chục thư viện nằm trong một kho mã nguồn (như thư mục `packages/*` trong file của bạn), việc cập nhật phiên bản và đẩy từng cái lên NPM thủ công là cực hình. Lerna giúp tự động hóa việc tăng số phiên bản (versioning) và đảm bảo các package con phụ thuộc vào nhau luôn khớp phiên bản.

### 3. Các tính năng chính:
*   **Version Management:** Tự động xác định package nào đã thay đổi để tăng phiên bản (Patch/Minor/Major).
*   **Bulk Commands:** Chạy một lệnh (như `npm run build`) cho tất cả các package cùng một lúc chỉ với một câu lệnh duy nhất ở thư mục gốc.
*   **Publishing:** Tự động tạo git tag và đẩy các package lên NPM.
*   **Optimize Workspaces:** Kết hợp tốt với tính năng `workspaces` của npm/pnpm để tối ưu hóa việc cài đặt thư viện.

### 4. Cách sử dụng phổ biến
Trong dự án này, Lerna được dùng để chạy lệnh build và test cho toàn bộ các thư mục con:
*   `lerna run build`: Tìm tất cả lệnh build trong các package con và chạy chúng.
*   `lerna publish`: Cập nhật phiên bản và đẩy code lên hệ thống quản lý package.

**Tóm lại:** Lerna là "người điều phối" cho các dự án đa thư viện, tập trung mạnh vào quy trình phát hành và quản lý phiên bản.

