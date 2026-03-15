Dựa trên mã nguồn và cấu trúc thư mục của dự án **gstack**, đây là phân tích chi tiết về các khía cạnh kỹ thuật và kiến trúc:

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Runtime & Package Manager:** `Bun` v1.0+. Dự án tận dụng tối đa Bun vì khả năng khởi động cực nhanh, hỗ trợ TypeScript mặc định, tích hợp sẵn SQLite và khả năng biên dịch thành file thực thi duy nhất (standalone binary).
*   **Trình duyệt tự động hóa:** `Playwright` (Chromium). Sử dụng Playwright để điều khiển trình duyệt headless, chụp ảnh màn hình và tương tác với DOM.
*   **Ngôn ngữ:** `TypeScript` (77.3%) là ngôn ngữ chủ đạo cho logic xử lý, kết hợp với `Shell` cho các script cài đặt và `Go Template` cho hệ thống tài liệu.
*   **AI Agent Integration:** Được tối ưu hóa riêng cho `Claude Code` (Anthropic). gstack đóng vai trò là một "hệ điều hành" mở rộng kỹ năng cho Claude thông qua các slash commands.
*   **Security:** Sử dụng `macOS Keychain` API để giải mã cookie trình duyệt (AES-128-CBC) và cơ chế Bearer Token (UUID) để bảo mật giao tiếp giữa CLI và Server cục bộ.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của gstack tập trung vào việc biến một AI Agent đa năng thành một "đội ngũ chuyên gia" có trạng thái (stateful) và có "mắt" (trình duyệt).

*   **Daemon Model (Mô hình tiến trình chạy ngầm):** Thay vì khởi động trình duyệt mới cho mỗi câu lệnh (mất 3-5 giây), gstack duy trì một Chromium daemon chạy liên tục. CLI chỉ là một "thin client" gửi HTTP request đến server cục bộ. Điều này giúp giảm độ trễ xuống mức sub-second (~100-200ms).
*   **Cognitive Gear Shifting (Chuyển đổi chế độ nhận thức):** Kiến trúc phân tách logic dựa trên "vai trò":
    *   `/plan-ceo-review`: Chế độ Founder (tầm nhìn sản phẩm).
    *   `/plan-eng-review`: Chế độ Tech Lead (kiến trúc hệ thống).
    *   `/review`: Chế độ Staff Engineer (soát lỗi logic/bảo mật).
    *   `/ship`: Chế độ Release Engineer (quy trình triển khai).
*   **Project-Local State:** Mỗi workspace (dự án) có một thư mục `.gstack/` riêng để lưu trữ file trạng thái (`browse.json`), logs và cookie, đảm bảo tính cô lập khi làm việc trên nhiều dự án song song qua Conductor.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Ref System (Semantic Locators):** Đây là kỹ thuật thông minh nhất của dự án. Lệnh `snapshot` tạo ra một cây sơ đồ các phần tử tương tác dựa trên thuộc tính ARIA (Accessibility), gán cho chúng các ID ngắn gọn như `@e1`, `@e2`. AI chỉ cần gọi `click @e1` thay vì phải đoán CSS Selector phức tạp.
*   **Cookie Decryption (Giải mã Cookie):** gstack có khả năng truy cập trực tiếp vào DB cookie của các trình duyệt phổ biến (Chrome, Arc, Edge), giải mã chúng qua Keychain để AI có thể thực hiện QA trên các trang web yêu cầu đăng nhập mà không cần user nhập mật khẩu thủ công.
*   **Template-to-Skill Workflow:** Hệ thống tài liệu (`SKILL.md.tmpl`) được tự động hóa. Khi lập trình viên thêm một tính năng mới trong code, script `gen-skill-docs.ts` sẽ cập nhật trực tiếp hướng dẫn sử dụng vào file Markdown mà Claude đọc, tránh tình trạng tài liệu bị lạc hậu so với code (documentation drift).
*   **3-Tier Testing Infrastructure:**
    1.  *Static:* Kiểm tra cú pháp và logic cơ bản (miễn phí).
    2.  *E2E (End-to-End):* Chạy thực tế qua `claude -p` để kiểm tra luồng hoạt động.
    3.  *LLM-as-judge:* Sử dụng một model Claude khác để chấm điểm chất lượng phản hồi của Agent dựa trên các tiêu chí Clarity, Completeness, Actionability.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Kích hoạt:** Người dùng gõ một lệnh (ví dụ `/browse google.com`) trong Claude Code.
2.  **Khởi động Daemon:** Binary `browse` kiểm tra nếu server chưa chạy thì khởi tạo một tiến trình Bun chạy ngầm, mở Chromium headless.
3.  **Thực thi lệnh:**
    *   Claude gửi lệnh `goto`. Server Playwright điều khiển trình duyệt truy cập URL.
    *   Claude gửi lệnh `snapshot -i`. Server phân tích cây Accessibility, gán `@ref` và trả về danh sách các phần tử cho Claude.
    *   Claude gửi lệnh `click @e3`. Server ánh xạ `@e3` ngược lại Locator của Playwright và thực hiện cú click.
4.  **Duy trì trạng thái:** Mọi thay đổi về cookie, localStorage hay tab trình duyệt được giữ nguyên cho đến khi server tự tắt sau 30 phút không hoạt động (idle timeout).
5.  **Tích hợp bên thứ ba:** Trong luồng `/ship`, gstack tự động gọi API của `Greptile` để lấy các nhận xét review tự động, sau đó trình bày cho người dùng để quyết định sửa lỗi hay bỏ qua trước khi tạo PR.

**Kết luận:** gstack không chỉ là một công cụ automation, mà là một cơ sở hạ tầng (infrastructure) giúp AI Agent tương tác với thế giới web và quy trình kỹ thuật một cách chuyên nghiệp, nhanh chóng và có tính kỷ luật cao.