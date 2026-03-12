Dựa trên mã nguồn của dự án **Antigravity Telegram Bridge**, đây là phân tích chi tiết về kiến trúc và các kỹ thuật mà tác giả đã sử dụng để kết nối AI của Antigravity với Telegram:

### 1. Công nghệ cốt lõi (Core Tech Stack)
*   **VS Code Extension API:** Nền tảng chính để chạy extension. Sử dụng các lệnh đặc thù như `antigravity.sendPromptToAgentPanel` và `antigravity.getDiagnostics`.
*   **Node.js Runtime:** Chạy bên trong Extension Host của VS Code.
*   **Telegram Bot API (`node-telegram-bot-api`):** Thư viện để giao tiếp với máy chủ Telegram.
*   **Unix Domain Sockets (IPC):** Sử dụng thư viện `net` của Node.js để tạo giao tiếp giữa các cửa sổ VS Code khác nhau.
*   **File System Watching (`fs.watch`):** Theo dõi sự thay đổi file trong thư mục "brain" của AI để bắt kết quả trả về.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án giải quyết một vấn đề rất khó trong lập trình Extension: **Làm thế nào để điều khiển AI khi không có API phản hồi trực tiếp (Callback)?**

*   **Kiến trúc Master/Worker (IPC):** 
    *   Khi bạn mở nhiều cửa sổ VS Code, nếu mỗi cửa sổ đều chạy một Bot Telegram với cùng một Token, Telegram sẽ báo lỗi `409 Conflict`.
    *   **Giải pháp:** Extension sử dụng IPC để bầu ra một cửa sổ làm "Master". Chỉ Master mới kết nối với Telegram. Các cửa sổ khác là "Worker". Khi Master nhận được tin nhắn, nó sẽ điều hướng (route) lệnh đến Worker phù hợp (dựa trên project người dùng chọn trên Telegram).
*   **Cơ chế Phản hồi "Agentic" (File-based State):** 
    *   Thay vì đợi AI trả về chuỗi văn bản (vốn không có API chính thống trong bản hiện tại), Extension ép AI phải "viết" câu trả lời vào một file Markdown thông qua Prompt Injection.

### 3. Các kỹ thuật chính (Key Techniques)

#### A. Prompt Injection (Kỹ thuật then chốt)
Extension tự động thêm một đoạn chỉ dẫn (instruction) vào mỗi tin nhắn gửi đi:
> *"After your response, you MUST write your COMPLETE response to a file called 'telegram_response.md'..."*
Kỹ thuật này biến AI từ một công cụ chat thuần túy thành một "Agent" thực hiện hành động ghi file, giúp Extension có thể "bắt" được nội dung thông qua việc theo dõi file đó.

#### B. Brain Watcher (Trình theo dõi "Não bộ")
Extension theo dõi thư mục `~/.gemini/antigravity/brain/`. 
*   Nó lọc các file `.resolved` (artifact của AI).
*   Sử dụng **Debounce (3s)**: Chờ file ổn định trong 3 giây trước khi đọc để tránh gửi những đoạn mã đang viết dở dang lên Telegram.
*   **Cleaning:** Loại bỏ các thành phần rác như link `file:///`, các lệnh hệ thống `render_diffs()`, giúp nội dung hiển thị trên điện thoại sạch sẽ hơn.

#### C. Hệ thống Trạng thái (Status & Typing)
Dự án mô phỏng trải nghiệm chat thực tế:
*   Khi AI đang suy nghĩ, Extension gửi lệnh `sendChatAction('typing')` tới Telegram liên tục mỗi 4 giây.
*   Trạng thái được cập nhật theo thời gian thực: `Thinking` (đang xử lý), `Working` (đang viết artifact), `Idle` (đang rảnh).

#### D. Xử lý tin nhắn dài (Auto Splitting)
Telegram giới hạn mỗi tin nhắn 4096 ký tự. Extension có hàm `sendToTelegram` tự động cắt nhỏ nội dung tại các vị trí xuống dòng (`\n`) để gửi thành nhiều phần `(1/3)`, `(2/3)`, giúp không bị mất dữ liệu khi AI viết code dài.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Nhận lệnh:** Người dùng gửi tin nhắn từ điện thoại $\rightarrow$ **Master** nhận $\rightarrow$ Chuyển qua **IPC** tới **Worker** (cửa sổ project tương ứng).
2.  **Thực thi:** Worker dùng lệnh `antigravity.sendPromptToAgentPanel` kèm theo Instruction ẩn để "ra lệnh" cho AI.
3.  **Theo dõi:** AI bắt đầu làm việc và ghi các file artifact vào thư mục `brain`.
4.  **Bắt kết quả:** **Brain Watcher** phát hiện file `telegram_response.md` hoặc các file `.resolved` mới $\rightarrow$ Đọc nội dung $\rightarrow$ Dọn dẹp dữ liệu $\rightarrow$ Gửi ngược lại Telegram cho người dùng.
5.  **Fallback (Dự phòng):** Nếu cơ chế file thất bại, người dùng có thể dùng `/fetch` để Extension dùng AppleScript "giả lập" thao tác Copy-Paste từ giao diện chat của Antigravity (chỉ dành cho macOS).

### Đánh giá:
Dự án thể hiện tư duy **"Hacker"** rất tốt khi vượt qua các giới hạn đóng của phần mềm (Antigravity AI) bằng cách kết hợp giữa IPC, File Watching và Prompt Engineering. Đây là một giải pháp thông minh để biến một công cụ desktop thành một công cụ làm việc từ xa (Remote Work).