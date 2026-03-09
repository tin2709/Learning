Dựa trên mã nguồn và tài liệu bạn cung cấp, **Readest** là một dự án cực kỳ hiện đại, áp dụng những tư duy mới nhất trong phát triển ứng dụng đa nền tảng. Dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống này:

---

### 1. Công nghệ cốt lõi (Core Stack)
Readest sử dụng mô hình **Hybrid App** nhưng ở một đẳng cấp rất cao, kết hợp giữa Web và Hệ điều hành:

*   **Frontend (Giao diện):** **Next.js 16 + React 19**. Đây là sự lựa chọn táo bạo (Next.js thường dùng cho Web), nhưng Readest đã tinh chỉnh để nó chạy được trong môi trường Local (SSG - Static Site Generation).
*   **Backend & Native Bridge:** **Tauri v2 (Rust)**. Khác với Electron (ngốn RAM), Tauri sử dụng WebView có sẵn của hệ điều hành và dùng Rust để xử lý các tác vụ nặng (File I/O, SQLite, xử lý ảnh). Điều này giúp ứng dụng cực nhẹ và bảo mật.
*   **Mobile:** Tận dụng tính năng mới của **Tauri v2** để biên dịch sang iOS (Swift) và Android (Kotlin) từ cùng một codebase logic.
*   **Database:** **SQLite (qua LibSQL/Turso)**. Dự án sử dụng một lớp trừu tượng (Abstraction) để hỗ trợ cả lưu trữ cục bộ (Native) và đồng bộ hóa đám mây (WASM trên trình duyệt).
*   **Rendering Engine:** Dựa trên **foliate-js**. Đây là bộ thư viện xử lý hiển thị ebook (EPUB, MOBI) mạnh mẽ nhất hiện nay trên nền tảng Web.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

#### A. Kiến trúc Monorepo (Quản lý đa dự án)
Dự án sử dụng **pnpm workspaces** để quản lý:
*   `apps/readest-app`: Mã nguồn chính của ứng dụng.
*   `packages/foliate-js`: Thư viện lõi xử lý ebook.
*   `extensions/windows-thumbnail`: Một module viết bằng Rust để tích hợp sâu vào Windows Explorer (hiển thị bìa sách thay cho icon file).

#### B. Tư duy "Offline-First" nhưng "Cloud-Synced"
Hệ thống được thiết kế để hoạt động hoàn toàn không cần mạng. Dữ liệu (sách, ghi chú) lưu ở SQLite local. Khi có mạng, nó sử dụng **Supabase (Auth)** và **S3/R2 (Storage)** để đồng bộ.

#### C. Adapter Pattern (Mẫu thiết kế bộ chuyển đổi)
Readest áp dụng mẫu này rất rõ trong việc xử lý các dịch vụ bên thứ ba:
*   **TTS (Text-to-Speech):** Hỗ trợ nhiều "Provider" (Edge TTS, Web Speech API, Native Android/iOS TTS).
*   **Dịch thuật:** Hỗ trợ DeepL, Google, Azure, Yandex.
*   **AI:** Có khả năng chuyển đổi giữa Ollama (chạy AI local) và AI Gateway (chạy Cloud).

---

### 3. Kỹ thuật lập trình nổi bật (Programming Techniques)

#### A. Native Integration qua Rust
Thay vì dùng JavaScript cho mọi thứ, các tác vụ sau được đẩy xuống lớp Rust (`src-tauri`):
*   **Thumbnail Provider:** Xử lý COM DLL trên Windows để hiển thị bìa sách trong File Explorer.
*   **Discord RPC:** Tích hợp trạng thái "đang đọc sách" vào Discord.
*   **File Scanner:** Duyệt hàng nghìn file trong thư mục để nhập vào thư viện với tốc độ cực nhanh.

#### B. Xử lý UI/UX phức tạp
*   **Safe Area Insets:** Kỹ thuật xử lý "tai thỏ" (notch) trên iPhone/Android được viết rất kỹ trong `docs/safe-area-insets.md`, đảm bảo UI không bị che khuất.
*   **Theme Engine:** Sử dụng CSS Variables kết hợp với Tailwind CSS và DaisyUI, cho phép thay đổi toàn bộ màu sắc ứng dụng (Primary color, Background) theo thời gian thực.
*   **Web Workers:** Dùng để xử lý việc chuyển đổi định dạng sách (TXT converter) ở background, tránh làm đứng giao diện.

#### C. Type-Safety
Dự án áp dụng TypeScript cực kỳ nghiêm ngặt (Strict mode). Việc sử dụng `any` bị cấm hoàn toàn, thay vào đó là các interface phức tạp cho ghi chú (Annotations), vị trí sách (CFI - Canonical Fragment Identifier).

---

### 4. Luồng hoạt động hệ thống (System Flow)

#### Luồng 1: Nhập sách (Import Flow)
1.  Người dùng chọn file (hoặc kéo thả qua Tauri Bridge).
2.  Lớp Rust kiểm tra định dạng và trích xuất Metadata (tên sách, tác giả, ảnh bìa).
3.  Ảnh bìa được lưu vào bộ nhớ đệm (Cache), thông tin sách được lưu vào SQLite.
4.  Nếu bật Sync, một bản sao của file sẽ được đẩy lên Cloud (S3/R2).

#### Luồng 2: Hiển thị và Đọc (Reading Flow)
1.  `foliate-js` tải file từ hệ thống vào WebView.
2.  Next.js render các lớp phủ (Overlay) như: thanh tiến trình, thanh menu, ghi chú.
3.  Khi người dùng lật trang, vị trí (CFI) được lưu ngay lập tức vào database local (Debounced).
4.  Dịch thuật/Từ điển: Khi bôi đen văn bản, ứng dụng gọi API (DeepL/Wiki) hoặc gửi tin nhắn tới AI Assistant để giải thích nội dung.

#### Luồng 3: Đồng bộ hóa (Sync Flow)
1.  Sử dụng hệ thống **KOReader Sync Protocol** (một tiêu chuẩn phổ biến trong cộng đồng ebook) để cho phép Readest đồng bộ được với các thiết bị đọc sách chuyên dụng như Kindle/Kobo đã cài KOReader.
2.  Dùng cơ chế Checksum (MD5/Partial MD5) để kiểm tra sự thay đổi của file mà không cần tải lại toàn bộ dữ liệu.

---

### Tổng kết
Readest không chỉ là một ứng dụng đọc sách, nó là một bài mẫu về **Tauri + Next.js**. Điểm mạnh nhất của nó là việc **"Native hóa" sức mạnh của Web**, mang lại trải nghiệm mượt mà của ứng dụng gốc (Native App) nhưng vẫn giữ được sự linh hoạt và giao diện đẹp của công nghệ Web hiện đại.