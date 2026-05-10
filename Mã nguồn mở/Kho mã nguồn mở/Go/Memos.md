Dựa trên cấu trúc thư mục và nội dung các file từ bản ghi chép của dự án **Memos** (phiên bản khoảng tháng 5/2026), dưới đây là bản phân tích chuyên sâu về công nghệ, kiến trúc và các kỹ thuật lập trình đặc sắc của hệ thống này.

---

### 1. Phân tích Công nghệ Cốt lõi (Core Technology)

Memos là một ứng dụng Full-stack hiện đại, đi theo hướng "Lightweight" (nhẹ) nhưng vẫn đảm bảo khả năng mở rộng mạnh mẽ.

*   **Backend (Go 1.26+):**
    *   **Framework:** Sử dụng **Echo v5** cho HTTP Server. Đây là một lựa chọn tối ưu về hiệu suất.
    *   **API Layer:** Kết hợp **Connect RPC** và **gRPC-Gateway**. Connect RPC giúp việc gọi API từ Frontend (TypeScript) trở nên cực kỳ đơn giản và type-safe mà không cần overhead phức tạp của gRPC truyền thống trên trình duyệt.
    *   **AI Integration:** Tích hợp sâu với các mô hình AI thông qua `internal/ai`, hỗ trợ cả STT (Speech-to-Text) của OpenAI và Audio-LLM của Google Gemini.
    *   **MCP (Model Context Protocol):** Một điểm đặc biệt là dự án đã tích hợp MCP server, cho phép các AI Assistant (như Claude Desktop) có thể đọc/ghi dữ liệu trực tiếp vào Memos của người dùng.
*   **Frontend (React 18 + TypeScript 6):**
    *   **Build Tool:** **Vite 7** và **pnpm** (version 11).
    *   **Styling:** **Tailwind CSS v4** (sử dụng `@tailwindcss/vite`). Hệ thống màu sắc sử dụng **OKLch** tokens, mang lại độ chính xác màu sắc cao trên các màn hình hiện đại.
    *   **State Management:** Dữ liệu Server được quản lý bởi **React Query v5**, trong khi Client state dùng **React Context**.
*   **Storage (Đa trình điều khiển):**
    *   Hỗ trợ đồng thời **SQLite** (mặc định), **MySQL**, và **PostgreSQL**. Sử dụng cơ chế migration phiên bản (incremental migration) cho cả 3 DB.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Memos phản ánh sự trưởng thành và tuân thủ các tiêu chuẩn công nghiệp (như Google API Improvement Proposals - AIP).

*   **Resource-Oriented Design (AIP-122):** Hệ thống định danh tài nguyên theo dạng chuỗi: `users/alice`, `memos/123`. Điều này giúp API mang tính nhất quán và dễ dự đoán.
*   **Tách biệt Cơ chế Định danh (Identity Linkage):** Theo tài liệu `2026-04-21-sso-user-identity-linkage`, Memos đã tách biệt bảng `user` (thông tin người dùng nội bộ) và bảng `user_identity` (thông tin liên kết với SSO/OAuth2). Điều này cho phép một tài khoản Memos liên kết với nhiều nhà cung cấp định danh khác nhau mà không làm hỏng logic username.
*   **Kiến trúc AI phân tách (STT vs. Audio-LLM):** Hệ thống tách biệt rõ ràng giữa **STT** (chuyển đổi giọng nói thành văn bản thuần túy) và **Audio-LLM** (sử dụng mô hình ngôn ngữ lớn để hiểu và phản hồi từ âm thanh). Cách tiếp cận này cho phép hệ thống linh hoạt chuyển đổi giữa Whisper (OpenAI) và Gemini tùy theo nhu cầu về độ chính xác hay tính năng.
*   **Plugin-based Markdown:** Sử dụng `Goldmark` (Go) và `Remark/Rehype` (TS). Dự án tự xây dựng các extension cho `Tag` (#tag) và `Mention` (@user), giúp việc xử lý nội dung giàu tính năng nhưng vẫn bảo đảm bảo mật (sanitization).

---

### 3. Kỹ thuật Lập trình Đặc sắc (Programming Highlights)

*   **Sử dụng CEL (Common Expression Language):** Trong `internal/filter`, Memos sử dụng thư viện **Google CEL** để thực hiện các bộ lọc tìm kiếm phức tạp. Thay vì viết SQL động thủ công, hệ thống biên dịch các biểu thức lọc thành mã CEL, giúp ngăn chặn SQL Injection và tăng khả năng tùy biến lọc dữ liệu.
*   **Cơ chế "Race Recovery" khi SSO Sign-in:** Trong `auth_service.go`, khi xử lý đăng nhập SSO lần đầu, hệ thống thực hiện một kỹ thuật thông minh: thử tạo user và liên kết định danh đồng thời; nếu xảy ra xung đột (Race condition), nó sẽ bắt lỗi Unique Constraint và thực hiện "re-read" để lấy thông tin từ luồng đã thắng.
*   **Xử lý Media phức tạp:** Hỗ trợ **Motion Photo** (ảnh động của Google/Apple). Hệ thống có khả năng bóc tách phần video từ file ảnh và stream nó qua `fileserver`, cho phép xem ảnh Live ngay trên trình duyệt web.
*   **Zero-Telemetry & Privacy-First:** Toàn bộ logic xử lý (ngay cả AI nếu dùng Local models) đều hướng tới việc tự vận hành. Các API được thiết kế để không rò rỉ ID tuần tự (như chuyển từ `users/1` sang `users/username`).

---

### 4. Luồng Hoạt động Hệ thống (System Workflow)

**Luồng tạo một Memo có nhắc tên (@mention):**

1.  **Frontend:** Người dùng gõ `@`. Component `MentionSuggestions` gọi RPC `SearchUsers` để gợi ý danh sách. Sau khi chọn, nội dung lưu dưới dạng Markdown: `Chào @alice!`.
2.  **Backend (Create):**
    *   Hàm `CreateMemo` nhận dữ liệu, lưu nội dung Markdown gốc vào DB.
    *   Hệ thống gọi `memopayload.RebuildMemoPayload`.
    *   **Markdown Parser:** `internal/markdown` quét nội dung, phát hiện `MentionNode`.
    *   **Resolution:** Hệ thống kiểm tra xem `alice` có tồn tại không. Nếu có, ID của Alice được đưa vào trường `mentions` trong JSON payload của Memo.
3.  **Side Effects:**
    *   Hệ thống kiểm tra quyền truy cập (nếu Memo là Private, Alice sẽ không nhận được thông báo).
    *   Nếu thỏa mãn, một bản ghi `MEMO_MENTION` được tạo trong bảng `inbox`.
4.  **Real-time:** `SSE Hub` (Server-Sent Events) gửi một tín hiệu đến trình duyệt của Alice để cập nhật biểu tượng thông báo ngay lập tức.

---

### 5. Nhận xét & Đánh giá

Memos không chỉ là một ứng dụng ghi chú đơn giản mà là một bài mẫu về **Modern Web Architecture**.

*   **Ưu điểm:** Khả năng xử lý đa nền tảng tốt, mã nguồn Go rất sạch (clean code), tuân thủ nghiêm ngặt việc xử lý lỗi (linters bắt buộc dùng `errors.Wrap`). Việc chuyển dịch sang định danh dựa trên Username thay vì ID số tăng tính chuyên nghiệp cho API.
*   **Sự đổi mới:** Việc tích hợp **MCP (Model Context Protocol)** cho thấy dự án đang đi đầu trong xu hướng "AI-Native App", nơi ứng dụng không chỉ phục vụ con người mà còn phục vụ cả các tác nhân AI (AI Agents).

Tóm lại, đây là một dự án có độ hoàn thiện kỹ thuật rất cao, phù hợp để nghiên cứu về cách xây dựng hệ thống tự vận hành (self-hosted) bền vững và an toàn.