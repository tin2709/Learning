Dựa trên cấu trúc thư mục và nội dung mã nguồn của dự án Joplin, đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

Joplin là một hệ sinh thái đa nền tảng sử dụng mô hình **Monorepo** (quản lý nhiều gói mã nguồn trong một kho lưu trữ duy nhất) với các công nghệ chính:

*   **Ngôn ngữ:** TypeScript (chiếm >73%) là chủ đạo, đảm bảo tính chặt chẽ cho mã nguồn lớn. Rust được sử dụng cho các module hiệu năng cao (như `onenote-converter`).
*   **Frameworks:**
    *   **Desktop:** Electron + React (trong `packages/app-desktop`).
    *   **Mobile:** React Native (trong `packages/app-mobile`).
    *   **CLI:** Node.js (trong `packages/app-cli`).
    *   **Server:** Node.js + Koa Framework + PostgreSQL/SQLite (trong `packages/server`).
*   **Lưu trữ:** SQLite được dùng cho các ứng dụng client (Desktop/Mobile/CLI). PostgreSQL dành cho hệ thống Server đồng bộ.
*   **Xử lý văn bản:** Markdown-it (để render Markdown), TinyMCE (cho trình soạn thảo Rich Text).
*   **AI/HTR (Mới):** Sử dụng `llama.cpp` và `transcribe` service để nhận dạng chữ viết tay.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Joplin được thiết kế theo hướng **"Library-Centric"** và **"Offline-First"**:

*   **Mô hình Library-Centric (`packages/lib`):** Đây là thành phần quan trọng nhất. Toàn bộ logic nghiệp vụ cốt lõi (Core logic) như: Thuật toán đồng bộ (Synchronizer), Mô hình dữ liệu (Models), Mã hóa (E2EE), và các hàm tiện ích đều nằm ở đây. Các ứng dụng (Desktop, Mobile, CLI) chỉ là lớp giao diện (UI Layer) gọi đến thư viện dùng chung này.
*   **Cơ chế Shim (Platform Abstraction):** Do chạy trên nhiều môi trường khác nhau (Node.js, Browser, React Native), Joplin sử dụng một lớp "Shim". Ví dụ: Khi cần ghi file, lớp `lib` sẽ gọi `shim.fs.writeFile()`, và mỗi nền tảng sẽ tự cài đặt hàm này theo API riêng của nó.
*   **Offline-First:** Mọi thay đổi được ghi trực tiếp vào cơ sở dữ liệu local (SQLite). Việc đồng bộ hóa là một tiến trình chạy ngầm tách biệt, đảm bảo ứng dụng vẫn hoạt động mượt mà khi không có mạng.
*   **Plugin System:** Cho phép mở rộng tính năng mà không làm ảnh hưởng đến core, sử dụng cơ chế IPC (Inter-Process Communication) để giao tiếp giữa plugin và ứng dụng chính.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Xử lý bất đồng bộ (Async Management):** Sử dụng `AsyncActionQueue` và `TaskQueue` để quản lý các tác vụ nặng như đồng bộ hàng nghìn file hoặc xử lý tài nguyên (resources) mà không làm treo giao diện.
*   **Mã hóa đầu cuối (E2EE):** Kỹ thuật sử dụng AES và RSA để mã hóa dữ liệu tại client trước khi đẩy lên server đồng bộ. Chìa khóa mã hóa (Master Keys) cũng được quản lý và đồng bộ một cách an toàn.
*   **Data Migration System:** Hệ thống quản lý phiên bản cơ sở dữ liệu (`packages/lib/migrations`) giúp tự động cập nhật cấu trúc bảng khi người dùng nâng cấp ứng dụng.
*   **Dependency Injection (DI) thủ công:** Thông qua lớp `Registry`, hệ thống quản lý các instance của dịch vụ (như `SyncTarget`, `EncryptionService`) để các thành phần khác có thể truy cập dễ dàng.

### 4. Luồng hoạt động hệ thống (System Operation Flows)

#### A. Luồng Soạn thảo và Render (Edit & Render Flow):
1.  Người dùng nhập Markdown trong `NoteEditor`.
2.  Dữ liệu được lưu vào SQLite thông qua `BaseModel`.
3.  `Renderer` (`packages/renderer`) tiếp nhận Markdown, chạy qua các plugin của `markdown-it`.
4.  Kết quả là HTML được làm sạch (Sanitized) và hiển thị trên `Webview`.

#### B. Luồng Đồng bộ hóa (Synchronization Flow):
1.  **Phát hiện thay đổi:** Dựa trên bảng `item_changes` trong SQLite để biết những gì đã thay đổi từ lần đồng bộ cuối.
2.  **Mã hóa:** Nếu E2EE được bật, dữ liệu được mã hóa trong `EncryptionService`.
3.  **Tải lên (Upload):** `ItemUploader` đẩy dữ liệu lên "Sync Target" (Dropbox, OneDrive, Joplin Server...).
4.  **Giải quyết xung đột (Conflict Resolution):** Nếu một ghi chú bị sửa đổi ở cả hai nơi, Joplin sẽ tạo ra một bản sao "Conflict" thay vì ghi đè mất dữ liệu của người dùng.

#### C. Luồng Xử lý Tài nguyên (Resource Flow):
1.  Khi thêm ảnh/file đính kèm, file được lưu vào thư mục `resources`.
2.  Một bản ghi được tạo trong bảng `resources` để quản lý metadata.
3.  Hệ thống `ResourceService` sẽ quản lý việc tải/đẩy các file này lên cloud theo từng phần (chunking) để tiết kiệm băng thông.

### Tóm lại
Kiến trúc của Joplin là một ví dụ điển hình về việc xây dựng ứng dụng đa nền tảng bền vững bằng cách **tối đa hóa việc dùng chung mã nguồn (shared code)** thông qua TypeScript và gói `lib`, đồng thời duy trì tính linh hoạt bằng cách sử dụng các lớp trừu tượng (Shims) cho từng nền tảng cụ thể.