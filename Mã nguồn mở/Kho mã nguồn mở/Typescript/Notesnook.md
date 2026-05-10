Đây là bản phân tích chuyên sâu về dự án **Notesnook** dựa trên cấu trúc thư mục và nội dung mã nguồn bạn đã cung cấp. Notesnook là một minh chứng điển hình cho việc xây dựng ứng dụng mã hóa đầu cuối (E2EE) quy mô lớn, đa nền tảng.

---

### 1. Phân tích Công nghệ Cốt lõi (Core Technology)

Notesnook sử dụng mô hình **Monorepo** để quản lý một hệ sinh thái phức tạp bao gồm Web, Desktop, Mobile và các Package dùng chung.

*   **Ngôn ngữ & Framework:** 
    *   **TypeScript:** Chiếm 85%, đảm bảo tính an toàn kiểu liệu cho các thuật toán mã hóa phức tạp.
    *   **UI Layer:** React (Web/Desktop) và React Native (Mobile).
    *   **Desktop Layer:** Electron kết hợp với **tRPC** (một lựa chọn hiện đại thay thế cho IPC truyền thống) để giao tiếp giữa Main và Renderer process.
*   **Lưu trữ & Dữ liệu:**
    *   **SQLite:** Sử dụng `better-sqlite3-multiple-ciphers` (Desktop) và `react-native-quick-sqlite` (Mobile).
    *   **Kysely:** Một "Type-safe SQL query builder" được sử dụng xuyên suốt để đảm bảo các câu lệnh truy vấn SQL đồng nhất và không có lỗi logic trên mọi nền tảng.
*   **Mã hóa (Cryptography):**
    *   Dựa trên **libsodium** (thông qua `@notesnook/sodium` và `@notesnook/crypto`).
    *   Thuật toán chính: **XChaCha20-Poly1305** để mã hóa dữ liệu và **Argon2** để băm mật khẩu (KDF). Đây là những tiêu chuẩn bảo mật cực cao, hiện đại hơn AES-CBC truyền thống.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Notesnook được thiết kế theo hướng **"Shared Core, Platform-Specific Shells"**:

*   **@notesnook/core:** Đây là "bộ não". Nó chứa logic về đồng bộ hóa (Sync), quản lý bộ sưu tập (Collections), định dạng tiêu đề, và các bộ lọc tìm kiếm. Mọi nền tảng (Web/Mobile/Desktop) đều import package này để đảm bảo logic nghiệp vụ là duy nhất.
*   **Kiến trúc Offline-First:** Dữ liệu luôn được ưu tiên ghi vào SQLite cục bộ trước. Quá trình đồng bộ (`packages/core/src/api/sync`) diễn ra ngầm định, xử lý các xung đột (merge conflicts) dựa trên timestamp và định danh thiết bị.
*   **Module hóa Editor:** `@notesnook/editor` được tách riêng, xây dựng dựa trên **Tiptap/ProseMirror**. Điều này cho phép họ duy trì một trình soạn thảo văn bản giàu tính năng (Rich-text) đồng nhất trên Web và Desktop, trong khi có một bản wrapper mỏng (`@notesnook/editor-mobile`) cho Mobile.

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Techniques)

*   **tRPC qua IPC (Desktop):** 
    Thay vì dùng `ipcMain.on` và `ipcRenderer.send` một cách rời rạc, Notesnook sử dụng `electron-trpc`. Kỹ thuật này giúp lập trình viên gọi các hàm ở Main process từ Renderer process như thể đang gọi một hàm local, có đầy đủ Autocomplete và Type-checking.
    *   *File tiêu biểu:* `apps/desktop/src/api/index.ts` định nghĩa toàn bộ Router cho app.

*   **Dynamic Event Bridge bằng Proxy (Desktop):**
    Trong `apps/desktop/src/api/bridge.ts`, họ sử dụng **JavaScript Proxy** để tạo ra một hệ thống Subscription tự động. Khi cần tạo một Item mới, Main process phát tín hiệu qua Proxy này, và Renderer (UI) sẽ nhận được thông qua `observable` của tRPC.

*   **Custom Protocol Serving (Desktop):**
    Trong `apps/desktop/src/utils/protocol.ts`, họ đăng ký một protocol riêng để chặn các request đến `app.notesnook.com` và phục vụ file từ thư mục local. Điều này giúp tránh các vấn đề về CORS và tăng tốc độ load ứng dụng đáng kể so với việc chạy một web server nội bộ.

*   **Native Optimization (Mobile):**
    Việc sử dụng **Nitro Modules** và **MMKV Storage** cho thấy sự đầu tư vào hiệu năng. MMKV là hệ thống lưu trữ key-value cực nhanh dựa trên memory-mapped files, giúp việc truy xuất cài đặt người dùng gần như không có độ trễ.

### 4. Luồng Hoạt động Hệ thống (System Workflows)

#### A. Luồng Khởi tạo (Startup Workflow - Desktop)
1.  **CLI Parsing:** `cli.ts` phân tích các tham số (ví dụ: `--hidden` để chạy ngầm).
2.  **Configuration Loading:** `config.ts` tải cài đặt người dùng từ `config.json`.
3.  **Security Check:** Kiểm tra `safe-storage` của hệ điều hành để bảo vệ các token nhạy cảm.
4.  **Protocol & Tray:** Đăng ký giao thức custom và tạo biểu tượng khay hệ thống (Tray).
5.  **Window Creation:** Khởi tạo BrowserWindow với cấu hình bảo mật (chặn Geolocation, ép buộc Spellchecker).

#### B. Luồng Mã hóa & Lưu trữ (Encryption & Storage)
1.  Người dùng nhập ghi chú.
2.  **Editor** chuyển nội dung thành cấu trúc JSON (ProseMirror schema).
3.  **Crypto Package** lấy khóa từ bộ nhớ (đã giải mã qua mật khẩu vault), thực hiện mã hóa nội dung bằng XChaCha20.
4.  **Kysely Layer** tạo câu lệnh `INSERT` vào SQLite local.
5.  **Sync Engine** đánh dấu bản ghi này là "cần đồng bộ" (`dirty`).

#### C. Luồng Đồng bộ hóa (Sync Workflow)
1.  **Collector:** Thu thập tất cả các thay đổi cục bộ chưa được đẩy lên server.
2.  **Merger:** Kiểm tra xem trên server có phiên bản mới hơn không. Nếu có xung đột, thực hiện logic hòa trộn.
3.  **Encryption:** Dữ liệu luôn được mã hóa tại máy khách trước khi gửi đi. Server chỉ nhận được các "blob" dữ liệu đã mã hóa và không bao giờ biết nội dung bên trong.

---

### Tổng kết
Notesnook là một dự án có độ hoàn thiện kỹ thuật rất cao. Điểm đáng học tập nhất ở đây chính là cách họ **abtraction (trừu tượng hóa)** các thành phần phức tạp như Database và Encryption thành các Package riêng biệt, giúp việc bảo trì code trên 3 nền tảng khác nhau trở nên khả thi với một đội ngũ nhỏ. Tư duy sử dụng **tRPC cho Electron** và **Kysely cho SQLite** là những hướng đi rất hiện đại, giúp giảm thiểu tối đa các lỗi runtime.