# 1. JS Perf #1: Chi phí ẩn của việc kiểm tra giao nhau trong Mảng

Phần này phân tích một vấn đề hiệu năng phổ biến khi kiểm tra sự giao nhau giữa hai mảng trong JavaScript.

### Vấn đề

Cách tiếp cận sử dụng `source.some(item => target.includes(item))` để kiểm tra xem hai mảng có phần tử chung hay không trông đơn giản nhưng **cực kỳ kém hiệu quả** với mảng lớn.

### Nguyên nhân gốc rễ

*   `Array.includes()` có độ phức tạp **O(n)** (tìm kiếm tuyến tính).
*   `Array.some()` có độ phức tạp **O(m)** (lặp qua mảng nguồn).
*   Kết hợp lại, độ phức tạp tổng thể là **O(m * n)**, rất chậm khi m và n lớn.

### Giải pháp: Sử dụng cấu trúc dữ liệu tối ưu

Chuyển đổi mảng cần tìm kiếm (`target`) thành cấu trúc dữ liệu cho phép tra cứu **O(1)** như `Set`, `Map`, hoặc `Object`.

**Ví dụ sử dụng Set (Tốt nhất):**

```javascript
function isIntersectOptimized() {
  // O(n) để tạo Set từ target
  const set = new Set(target);
  // O(m) để lặp qua source, với mỗi lần kiểm tra .has() là O(1)
  return source.some(item => set.has(item));
}
// Tổng độ phức tạp thời gian: O(n + m)
````
# Tối ưu hóa kiểm tra giao nhau giữa Mảng: `some`+`includes` vs `Set`

Tài liệu này so sánh hiệu năng và độ phức tạp của hai phương pháp kiểm tra sự giao nhau (tìm phần tử chung) giữa các mảng trong JavaScript: sử dụng `some` kết hợp `includes` và sử dụng `Set`.

## So sánh độ phức tạp

| Phương pháp             | Độ phức tạp Thời gian | Độ phức tạp Không gian (Bổ sung) | Ghi chú                                       |
| :---------------------- | :-------------------- | :------------------------------- | :-------------------------------------------- |
| **Gốc (`some`+`includes`)** | **O(m * n)**          | O(1)                             | Rất chậm với mảng lớn (m, n là kích thước) |
| **Tối ưu (dùng `Set`)**   | **O(n + m)**          | O(n) (để lưu trữ `Set`)         | Nhanh hơn đáng kể với mảng lớn             |

*(Trong đó: m = kích thước mảng nguồn (source), n = kích thước mảng đích (target))*

## Kết luận chính (Array Intersection)

*   **Tránh sử dụng `Array.includes` bên trong vòng lặp** (như `Array.some`, `for`, `forEach`) để kiểm tra sự tồn tại trong một mảng *khác* khi làm việc với dữ liệu lớn, do độ phức tạp **O(m * n)** của nó.
*   **Sử dụng `Set`** (hoặc `Map`/`Object` trong một số trường hợp) để chuyển đổi mảng cần tra cứu. Điều này giúp giảm độ phức tạp thời gian xuống **O(n + m)**, cải thiện đáng kể hiệu năng.
*   `Set` là lựa chọn **tự nhiên và hiệu quả nhất** cho bài toán kiểm tra sự tồn tại hoặc tìm các phần tử chung giữa các tập hợp trong JavaScript.
