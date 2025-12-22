
## Knip
### 1. Knip là gì?
**Knip** là một công cụ phân tích mã nguồn chuyên tìm kiếm các file, biến, hàm và thư viện (dependencies) không còn được sử dụng.

### 2. Nó giải quyết vấn đề gì?
Sau thời gian dài phát triển, dự án thường tích tụ rất nhiều "code rác" (dead code) – những thứ đã viết nhưng không ai dùng tới. Knip giúp bạn dọn dẹp dự án, giúp code nhẹ hơn và dễ bảo trì hơn.

### 3. Các tính năng chính:
*   **Phát hiện Code chết:** Tìm ra các hàm/biến đã `export` nhưng chưa từng được `import` ở đâu.
*   **Kiểm tra Dependencies:** Chỉ ra các package đã cài trong `package.json` nhưng thực tế không dùng trong code.
*   **Hỗ trợ Monorepo:** Hoạt động hiệu quả trên quy mô dự án lớn có nhiều package.

### 4. Cách sử dụng phổ biến
*   `npx knip`: Chạy phân tích và liệt kê danh sách các phần tử cần xóa bỏ.

**Tóm lại:** Đây là "người dọn dẹp" tận tâm, giúp dự án của bạn luôn tinh gọn và sạch sẽ.

