Dưới đây là nội dung bài viết về **Claude Code** đã được chuyển đổi thành định dạng **README.md** chuyên nghiệp, giúp bạn và team dễ dàng tra cứu các lệnh nâng cao để tối ưu hiệu suất lập trình.

---

# 🚀 Claude Code: 10 Hidden Commands for Supercharged Productivity

![Version](https://img.shields.io/badge/Update-March_2026-blue)
![Category](https://img.shields.io/badge/Tool-AI_Coding_CLI-orange)
![Efficiency](https://img.shields.io/badge/Productivity-10x-brightgreen)

Tài liệu này tổng hợp các lệnh ẩn và tính năng mới nhất (cập nhật đến tháng 3/2026) của **Claude Code** giúp bạn làm chủ CLI, tối ưu hóa Context và quản lý quy trình lập trình AI chuyên nghiệp.

---

## 📑 Danh sách lệnh nâng cao

### 1. `/btw` (By The Way)
Cho phép đặt câu hỏi phụ trong khi Claude đang thực hiện một task chính mà không làm gián đoạn luồng xử lý hoặc làm bẩn lịch sử hội thoại (Context Pollution).
*   **Cơ chế:** Chạy song song, không lưu vào history chính sau khi đóng.
*   **Sử dụng:** `[Task đang chạy] -> gõ /btw [Câu hỏi]`.

### 2. `/rewind` (Phím tắt: `Esc` x2)
Tính năng Undo mạnh mẽ, cho phép quay lại trạng thái trước đó của code hoặc hội thoại.
*   **Các tùy chọn:**
    *   Khôi phục cả code & hội thoại.
    *   Chỉ khôi phục hội thoại (giữ code hiện tại).
    *   Chỉ khôi phục code (giữ hội thoại).
    *   Tóm tắt hội thoại từ điểm này để giải phóng Context.

### 3. `/insights`
Tạo báo cáo HTML phân tích thói quen sử dụng của bạn trong 30 ngày qua.
*   **Nội dung:** Thống kê lệnh hay dùng, gợi ý tạo Custom Commands và các Skills phù hợp dựa trên pattern làm việc của bạn.

### 4. `/model opusplan` (Dành cho gói Pro)
Kích hoạt chế độ Hybrid thông minh để tối ưu quota.
*   **Cơ chế:** Sử dụng **Claude Opus 4.6** để lập kế hoạch (Planning) và tự động chuyển sang **Sonnet 4.6** để thực hiện viết code.
*   **Lợi ích:** Tận dụng khả năng tư duy sâu của Opus mà không lo cạn kiệt rate limit sớm.

### 5. `/simplify` (Built-in Skill)
Thay thế lệnh `/review` cũ, sử dụng 3 Agent chạy song song để tối ưu hóa code.
*   **Góc độ phân tích:**
    1. Khả năng tái sử dụng (Reusability).
    2. Chất lượng code (Clean code).
    3. Hiệu suất thực thi (Performance).

### 6. `/branch` (Hoặc `/fork`)
Tách nhánh hội thoại hiện tại thành một Session mới.
*   **Sử dụng:** Khi bạn muốn thử nghiệm một hướng triển khai khác (vũ trụ song song) mà không muốn mất đi tiến trình hiện tại.

### 7. `/loop`
Thiết lập task chạy định kỳ.
*   **Ví dụ:** `/loop 5m kiểm tra trạng thái deploy`.
*   **Lưu ý:** Kết quả mỗi lần loop được lưu vào context để Claude theo dõi biến động theo thời gian. Tự động hết hạn sau 3 ngày.

### 8. `/remote-control` (Hoặc `/rc`)
Đồng bộ terminal với thiết bị di động.
*   **Cơ chế:** Tạo URL để bạn điều khiển session Claude Code từ điện thoại. Mọi thao tác đồng bộ real-time với máy local.

### 9. `/export`
Xuất toàn bộ lịch sử hội thoại hiện tại ra định dạng **Markdown**.
*   **Ứng dụng:** Lưu trữ insight kiến trúc hoặc dùng làm dữ liệu đầu vào (Context) cho các model khác hoặc session sau.

---

## ⌨️ Phím tắt (Shortcuts) "Bất ly thân"

| Phím tắt | Chức năng |
| :--- | :--- |
| **Ctrl + V** | Paste trực tiếp Screenshot vào terminal (Claude sẽ tự phân tích ảnh). |
| **Ctrl + J** (hoặc `Opt + Enter`) | Xuống dòng trong terminal mà không gửi lệnh. |
| **Ctrl + R** | Tìm kiếm lại các prompt đã nhập trong lịch sử. |
| **Ctrl + U** | Xóa nhanh toàn bộ dòng lệnh đang nhập. |
| **Esc + Esc** | Gọi nhanh menu `/rewind`. |

---

## 💡 Lời khuyên tối ưu
*   **Theo dõi X (Twitter):** Nhiều tính năng của Claude Code được release "ngầm" hoặc qua các bài đăng của Dev team trước khi vào Changelog.
*   **Prompt Caching:** Sử dụng `/btw` thường xuyên vì nó hầu như không tốn thêm token nhờ tận dụng cache của task chính đang chạy.
*   **Cleanup:** Sử dụng tùy chọn `Summarize from here` trong lệnh `/rewind` để làm sạch context khi session quá dài, giúp Claude phản hồi chính xác hơn.

---
*© 2026 - Claude Code Power User Guide. Happy Coding!*