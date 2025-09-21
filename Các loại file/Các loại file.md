# 1 File HAR
Trong tab **"Mạng" (Network)** của Công cụ nhà phát triển (Developer Tools) trên trình duyệt, icon mũi tên hướng lên (biểu tượng "upload" hoặc "import") và dòng chữ **"Nhập tên HAR"** (hoặc đôi khi chỉ là "Import") có ý nghĩa là:

Nó cho phép bạn **tải lên (import) một file `.har` (HTTP Archive file)** đã được lưu trữ trước đó.

### HAR File (HTTP Archive file) là gì?

*   **HAR** là viết tắt của **HTTP Archive**. Đây là một định dạng file chuẩn (dựa trên JSON) được sử dụng để **ghi lại toàn bộ nhật ký hoạt động mạng** của một trang web trong một phiên làm việc cụ thể.
*   File HAR chứa thông tin rất chi tiết về mọi yêu cầu (request) và phản hồi (response) HTTP/HTTPS mà trình duyệt của bạn đã thực hiện khi tải và tương tác với một trang web.

### File HAR ghi lại những thông tin gì?

Một file HAR có thể chứa:
*   **URL** của tất cả các tài nguyên được yêu cầu (HTML, CSS, JavaScript, hình ảnh, API, v.v.).
*   **Thời gian tải** của từng tài nguyên (thời gian chờ đợi, thời gian nhận phản hồi, v.v.).
*   **Headers (tiêu đề)** của yêu cầu và phản hồi (ví dụ: `User-Agent`, `Content-Type`, `Cookie`, `Authorization`).
*   **Nội dung (content)** của yêu cầu và phản hồi (body của POST request, nội dung JSON từ API response).
*   **Trạng thái (status code)** của phản hồi (ví dụ: 200 OK, 404 Not Found, 500 Internal Server Error).
*   **Thông tin cookie**.
*   **CORS** (Cross-Origin Resource Sharing) headers và các chi tiết liên quan.
*   Và nhiều thông tin khác nữa...

### Tại sao lại cần "Nhập tên HAR" (Import HAR)?

Chức năng này cực kỳ hữu ích cho việc:

1.  **Phân tích lỗi (Debugging):**
    *   Nếu một nhà phát triển khác hoặc nhóm hỗ trợ gặp lỗi trên một trang web, họ có thể ghi lại một file HAR và gửi cho bạn. Bạn có thể nhập file HAR đó vào công cụ nhà phát triển của mình để xem chính xác các yêu cầu mạng đã xảy ra, từ đó dễ dàng xác định nguyên nhân gây lỗi (ví dụ: API trả về lỗi 500, một file JavaScript không tải được, lỗi CORS, v.v.).
2.  **Đánh giá hiệu suất (Performance Analysis):**
    *   So sánh hiệu suất tải trang giữa các phiên làm việc hoặc các môi trường khác nhau. Bạn có thể có một file HAR từ môi trường phát triển và một từ môi trường sản phẩm để so sánh.
3.  **Kiểm soát chất lượng (QA) và báo cáo lỗi:**
    *   Tester có thể ghi lại file HAR khi phát hiện một vấn đề và đính kèm nó vào báo cáo lỗi, giúp nhà phát triển tái tạo và khắc phục vấn đề nhanh chóng.
4.  **Hỗ trợ kỹ thuật:**
    *   Khi bạn gặp sự cố với một dịch vụ trực tuyến, đội ngũ hỗ trợ có thể yêu cầu bạn gửi file HAR để họ có thể xem xét luồng mạng chi tiết trên máy của bạn.

Tóm lại, nút "Nhập tên HAR" (Import HAR) cho phép bạn **tải một file chứa lịch sử hoạt động mạng đã được ghi lại trước đó vào công cụ nhà phát triển của bạn để xem và phân tích**, thay vì phải tự ghi lại từ đầu.