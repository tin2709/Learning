Chào bạn, dựa trên tệp tin bạn cung cấp, đây là bản phân tích chi tiết về dự án **MedAssist** - một ứng dụng quản lý thuốc tự lưu trữ (self-hosted).

---

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

Hệ thống được xây dựng trên mô hình **Monolithic** đơn giản nhưng hiệu quả cho việc sử dụng cá nhân:

*   **Backend (Runtime):** Node.js với framework Express.js. Đây là lựa chọn tối ưu cho ứng dụng nhẹ, xử lý các yêu cầu I/O không đồng bộ tốt.
*   **Database:** SQLite3.
    *   *Ưu điểm:* Không cần cài đặt server database phức tạp, dữ liệu lưu trong 1 file `.db` duy nhất, cực kỳ phù hợp cho môi trường Docker và backup cá nhân.
*   **Frontend:** Vanilla HTML, CSS và JavaScript (không dùng framework như React/Vue). Sử dụng thư viện **Flatpickr** để xử lý lịch và thời gian.
*   **Tiện ích phụ trợ:**
    *   `node-cron`: Lập lịch gửi email tự động.
    *   `nodemailer`: Xử lý gửi email thông qua SMTP.
*   **Containerization:** Docker & Docker Compose giúp triển khai nhanh trên các hệ thống NAS (như Synology) hoặc Raspberry Pi.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án thể hiện tư duy thiết kế tập trung vào **sự đơn giản và tính riêng tư**:

*   **Dữ liệu tập trung (Flat Schema):** Thay vì sử dụng các bảng quan hệ phức tạp (Many-to-Many) cho thuốc và lịch trình, tác giả chọn cách lưu trữ lịch trình (schedules) dưới dạng chuỗi JSON (`JSON.stringify`) ngay trong bảng `list`. Điều này giúp giảm độ phức tạp của các câu lệnh SQL nhưng vẫn cho phép một loại thuốc có nhiều khung giờ uống khác nhau.
*   **Tính toán động (On-the-fly Calculation):** Ứng dụng không lưu số lượng thuốc còn lại cố định mà tính toán dựa trên: `Số lượng ban đầu` - `Tổng lượng đã tiêu thụ từ ngày cập nhật cuối`. Cách tiếp cận này đảm bảo tính chính xác theo thời gian thực mà không cần chạy các tiến trình trừ tồn kho mỗi giây.
*   **Thiết kế Mobile-First:** Mặc dù là giao diện web, nhưng CSS được tối ưu hóa cho hiển thị trên điện thoại (qua các screenshot và media queries), phù hợp với việc kiểm tra thuốc ngay tại tủ thuốc.

---

### 3. Các Kỹ thuật Chính (Key Techniques)

*   **Thuật toán tính toán tồn kho:** Trong hàm `fetchAndCalculateMedications`, tác giả sử dụng một vòng lặp `while` mô phỏng thời gian tương lai (tối đa 365 ngày) để dự báo chính xác ngày nào thuốc sẽ hết (`orderBefore`). Vòng lặp này tính đến cả chu kỳ uống thuốc (`every X days`).
*   **Xử lý Múi giờ (Timezone Management):** Sử dụng các hàm `getUTCDate` và chuyển đổi ISO string để đảm bảo thời gian uống thuốc không bị sai lệch khi người dùng chạy container ở các múi giờ khác nhau.
*   **Hệ thống Nhắc nhở thông minh:** 
    *   Kết hợp giữa `node-cron` và logic kiểm tra `min_days_left`. 
    *   Có cơ chế `email_delay_days` để tránh việc gửi quá nhiều email rác (spam) nếu người dùng chưa kịp bổ sung thuốc.
*   **Planner Logic:** Hàm `fetchAndCalculateMedicationsByDateRange` tính toán tổng lượng thuốc cần thiết cho một khoảng thời gian cụ thể (đi du lịch), giúp người dùng chuẩn bị đủ số lượng thuốc mang theo.

---

### 4. Luồng Hoạt động của Hệ thống (System Workflow)

1.  **Thiết lập (Configuration):**
    *   Người dùng cấu hình SMTP (Gmail/Outlook...) trong trang Settings.
    *   Thêm thuốc: Nhập tên, tổng số lượng hiện có, và tạo các lịch trình (ví dụ: sáng 1 viên, tối 1 viên).
2.  **Xử lý dữ liệu:**
    *   Backend lưu dữ liệu vào SQLite.
    *   Mỗi khi truy cập Dashboard, server sẽ tính toán: `Meds Left` (còn lại bao nhiêu) và `Days Left` (còn dùng được bao nhiêu ngày).
3.  **Hiển thị & Cảnh báo:**
    *   **Dashboard:** Hiển thị danh sách thuốc sắp hết lên đầu và danh sách các lần uống thuốc trong 2-3 ngày tới.
    *   **Background Task:** Mỗi ngày (theo giờ cấu hình), Cron Job sẽ quét database. Nếu có thuốc dưới ngưỡng `min_days_left`, email thông báo sẽ được gửi đi.
4.  **Lập kế hoạch (Planner):**
    *   Người dùng chọn ngày đi và ngày về. Hệ thống tính tổng lượng thuốc cần dùng và so sánh với kho hiện tại để cảnh báo nếu không đủ cho chuyến đi.

---

### 5. Đánh giá Tổng quan

**Ưu điểm:**
*   Cực kỳ nhẹ, dễ cài đặt.
*   Giải quyết đúng nỗi đau (pain point) của người phải dùng nhiều loại thuốc: "Khi nào cần mua thêm?" và "Đi du lịch mang bao nhiêu là đủ?".
*   Mã nguồn mở, hoàn toàn riêng tư (không gửi dữ liệu y tế lên cloud bên thứ ba).

**Hạn chế (Cần lưu ý):**
*   **Bảo mật:** Hiện tại ứng dụng chưa có lớp Login (đang nằm trong lộ trình phát triển). Nếu mở port ra internet mà không qua Proxy có xác thực, ai cũng có thể xem/sửa dữ liệu.
*   **Sự tin cậy:** Như tác giả đã cảnh báo, đây là dự án sở thích, không nên dùng cho các loại thuốc sinh tử (life-critical) mà không có phương án dự phòng thủ công.

Đây là một ví dụ tuyệt vời về việc áp dụng lập trình để giải quyết vấn đề thực tế trong cuộc sống hàng ngày!