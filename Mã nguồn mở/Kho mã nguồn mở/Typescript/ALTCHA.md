Dưới đây là phân tích chi tiết về dự án **ALTCHA** (phiên bản v3), một giải pháp bảo mật thay thế CAPTCHA truyền thống dựa trên cơ chế Proof-of-Work (PoW) và ưu tiên quyền riêng tư.

### 1. Công nghệ Cốt lõi (Core Technologies)

*   **Svelte 5 & Web Components:** Dự án sử dụng Svelte 5 (với các tính năng mới như `$state`, `$derived`, `$effect`) để xây dựng UI. Tuy nhiên, nó được đóng gói dưới dạng **Web Component** (`customElement: true`), cho phép tích hợp vào bất kỳ nền tảng nào (React, Vue, HTML thuần) mà không bị xung đột.
*   **Web Crypto API:** Sử dụng thư viện `crypto.subtle` có sẵn trong trình duyệt để thực hiện các phép tính băm (SHA-256/384/512) và HMAC, đảm bảo hiệu suất native và tính bảo mật cao mà không cần thư viện ngoài nặng nề.
*   **Web Workers:** Cơ chế PoW chạy trên các luồng riêng biệt (Web Workers) để tránh làm treo luồng chính (Main thread/UI thread), giữ cho trải nghiệm người dùng mượt mà trong khi máy tính đang giải đố.
*   **Memory-hard Algorithms (Argon2 & Scrypt):** Ngoài các thuật toán CPU-bound truyền thống, v3 tích hợp các thuật toán ràng buộc bộ nhớ thông qua **WASM** (`hash-wasm`). Điều này giúp chống lại sự tăng tốc phần cứng từ các trang trại bot sử dụng ASICs hoặc GPU.

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Mô hình Zero-Tracking & Privacy-First:** Kiến trúc không sử dụng cookies (mặc định), không theo dõi vân tay trình duyệt (fingerprinting) và không lưu trữ dữ liệu người dùng trên máy chủ trung tâm. Mọi xác thực đều diễn ra cục bộ hoặc trên máy chủ tự host của người dùng.
*   **Tính bất đối xứng (Asymmetric Complexity):** Thiết kế sao cho việc "giải đố" (solving) tiêu tốn nhiều tài nguyên của client (kẻ tấn công), nhưng việc "xác thực" (verification) lại cực kỳ nhanh và nhẹ nhàng cho server.
*   **Hệ thống Plugin & Registry toàn cục:** Sử dụng một đối tượng toàn cục `$altcha` để quản lý các cấu hình mặc định, đăng ký thuật toán mới và các bản dịch. Điều này cho phép mở rộng tính năng (như plugin ẩn danh dữ liệu - *Obfuscation*) mà không làm thay đổi logic lõi.
*   **Thiết kế thích ứng (Responsive & Adaptive):** Hỗ trợ nhiều chế độ hiển thị (`standard`, `floating`, `bar`, `overlay`, `invisible`) để phù hợp với mọi loại form và luồng trải nghiệm người dùng.

### 3. Kỹ thuật Lập trình Đặc sắc (Notable Techniques)

*   **Cơ chế Thử lại khi thiếu bộ nhớ (OOM Retry Logic):** Trong file `pow.ts`, hàm `solveChallengeWorkers` có logic tự động giảm số lượng Web Workers và thử lại nếu trình duyệt báo lỗi hết bộ nhớ (Out-Of-Memory) khi chạy Argon2/Scrypt. Đây là kỹ thuật xử lý lỗi rất thông minh cho các thiết bị di động cấu hình thấp.
*   **So sánh thời gian không đổi (Constant-time Equality):** Hàm `constantTimeEqual` trong `helpers.ts` được sử dụng để so sánh các chữ ký số. Kỹ thuật này ngăn chặn các cuộc tấn công dựa trên thời gian (timing attacks) bằng cách đảm bảo thời gian so sánh luôn như nhau bất kể dữ liệu khớp hay không.
*   **Thu thập dấu hiệu tương tác người dùng (HIS - Human Interaction Signature):** Trong `his.ts`, hệ thống thu thập các mẫu (samples) về sự tập trung (focus), di chuyển con trỏ, cuộn trang và chạm màn hình. Dữ liệu này được gửi kèm để server phân tích hành vi con người thực thay vì chỉ dựa vào PoW thuần túy.
*   **Xử lý i18n động:** Dự án hỗ trợ hơn 50 ngôn ngữ với cơ chế phát hiện ngôn ngữ tự động từ thẻ `<html lang>` hoặc `navigator.languages`, kết hợp với khả năng nạp các gói ngôn ngữ riêng lẻ để tối ưu kích thước bundle.

### 4. Luồng Hoạt động Hệ thống (System Workflow)

1.  **Challenge Request (Yêu cầu thử thách):** Khi widget tải lên, nó gửi một yêu cầu GET đến server (hoặc nhận challenge tĩnh). Server trả về một đối tượng chứa: `salt`, `algorithm`, `cost` (độ khó) và `signature` (HMAC).
2.  **Solving Phase (Giải đố):**
    *   Widget khởi tạo Web Workers dựa trên số lượng CPU cores.
    *   Workers thực hiện vòng lặp: `Băm(nonce + counter)`.
    *   Mục tiêu là tìm ra một giá trị `counter` sao cho kết quả băm có phần tiền tố (prefix) khớp với yêu cầu của server.
3.  **Payload Generation:** Sau khi tìm được `counter`, client đóng gói kết quả thành một chuỗi JSON mã hóa Base64 (gọi là *Payload*).
4.  **Verification (Xác thực):**
    *   **Client-side:** Kiểm tra sơ bộ tính hợp lệ của challenge và thời gian hết hạn.
    *   **Server-side:** Server nhận Payload, kiểm tra chữ ký HMAC để đảm bảo challenge không bị sửa đổi, sau đó chỉ cần thực hiện **duy nhất một phép tính băm** với `counter` do client gửi lên để xác nhận kết quả.
5.  **Form Submission:** Nếu xác thực thành công, widget kích hoạt trạng thái `verified`, cho phép người dùng gửi form đi cùng với mã xác thực ALTCHA.

**Tổng kết:** ALTCHA v3 là sự kết hợp nhuần nhuyễn giữa kỹ thuật mật mã học hiện đại và kiến trúc phần mềm linh hoạt. Nó giải quyết bài toán chống spam bằng cách "đánh thuế" tài nguyên tính toán của bot mà vẫn bảo vệ tối đa quyền riêng tư và khả năng truy cập (accessibility) của người dùng thực.