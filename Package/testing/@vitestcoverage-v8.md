
## @vitest/coverage-v8
### 1. @vitest/coverage-v8 là gì?
Đây là một plugin dành cho framework kiểm thử **Vitest**, sử dụng bộ máy V8 (của Chrome) để đo lường độ bao phủ của các bản test (code coverage).

### 2. Nó giải quyết vấn đề gì?
Bạn viết rất nhiều bản test nhưng không biết liệu chúng đã kiểm tra hết tất cả các dòng code trong dự án chưa. Thư viện này sẽ thống kê và báo cáo cho bạn những phần code nào đang bị "bỏ rơi" chưa được test.

### 3. Các tính năng chính:
*   **Tốc độ cực nhanh:** Sử dụng trực tiếp tính năng đo lường có sẵn trong nhân V8 của trình duyệt.
*   **Báo cáo đa dạng:** Xuất ra file HTML trực quan hoặc các định dạng khác cho các công cụ CI/CD.

### 4. Cách sử dụng phổ biến
*   `vitest run --coverage`: Chạy test và xuất báo cáo độ bao phủ.

**Tóm lại:** Đây là công cụ "đo lường chất lượng", giúp bạn biết được dự án của mình đã được kiểm thử kỹ lưỡng đến đâu.

