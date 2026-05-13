Dựa trên mã nguồn của dự án **Ueli**, đây là một ứng dụng Electron được cấu trúc cực kỳ bài bản và hiện đại. Dưới đây là phân tích chi tiết:

### 1. Phân tích Công nghệ Cốt lõi (Core Technology Stack)

*   **Runtime & Framework:** Sử dụng **Electron** kết hợp với **Node.js 24**. Đây là bộ khung cho phép xây dựng ứng dụng desktop đa nền tảng bằng công nghệ web.
*   **Frontend:** Sử dụng **React 19** kết hợp với **Fluent UI React v9** (hệ thống design của Microsoft). Điều này giúp Ueli có giao diện đồng bộ với Windows 11 và macOS hiện đại.
*   **Ngôn ngữ:** Sử dụng **TypeScript 6** (phiên bản mới nhất) trên toàn bộ dự án (Main, Renderer, Preload). Việc sử dụng TypeScript giúp đảm bảo an toàn về kiểu dữ liệu (Type-safety) cho một hệ thống nhiều module phức tạp.
*   **Bundler:** Sử dụng **Vite** thay vì Webpack truyền thống, giúp tốc độ build và hot-reload cực nhanh trong quá trình phát triển.
*   **Tìm kiếm & Thuật toán:** Sử dụng các thư viện chuyên dụng như **Fuse.js** và **Fuzzysort** để xử lý tìm kiếm mờ (fuzzy search), đảm bảo kết quả hiện ra ngay lập tức khi người dùng gõ phím.
*   **Xử lý âm thanh/hình ảnh:** Sử dụng **Sharp** để xử lý hình ảnh (extract icon ứng dụng) hiệu năng cao.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Ueli tuân thủ chặt chẽ nguyên lý **Module-Based** và **Dependency Injection (DI)** thủ công:

*   **ModuleRegistry:** Đây là "trái tim" của Main Process. Thay vì sử dụng các biến toàn cục, Ueli khởi tạo một `ModuleRegistry` để đăng ký và quản lý instance của các dịch vụ như `Logger`, `FileSystem`, `SettingsManager`, `EventEmitter`.
*   **Kiến trúc Extension-Centric:** Mỗi tính năng (Máy tính, Tìm kiếm file, Google Search...) là một Extension độc lập tuân thủ theo một Interface chung (`src/main/Core/Extension/Contract/Extension.ts`). Điều này cho phép dễ dàng mở rộng hoặc tắt bớt tính năng mà không ảnh hưởng đến lõi (Core).
*   **Tách biệt quy trình (Process Separation):**
    *   `src/main`: Chứa logic nghiệp vụ nặng, tương tác OS (File system, Shell).
    *   `src/renderer`: Chỉ lo việc hiển thị UI.
    *   `src/common`: Chứa các Interface/Type chung, đảm bảo sự nhất quán khi giao tiếp IPC.
*   **Abstracting Operating System:** Ueli có các lớp trừu tượng để xử lý sự khác biệt giữa Windows, macOS và Linux (ví dụ: `AppIconFilePathResolver` hay `TerminalRegistry`).

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Techniques)

*   **Typed IPC Bridge:** Ueli không sử dụng IPC một cách lỏng lẻo. Toàn bộ các hàm giao tiếp giữa Renderer và Main được định nghĩa chặt chẽ trong `ContextBridge.ts`, giúp lập trình viên Frontend biết chính xác các hàm nào có sẵn và tham số của chúng.
*   **Action Handler Registry:** Khi một kết quả tìm kiếm được chọn, Ueli không chỉ "chạy" nó. Nó sử dụng một `ActionHandlerRegistry`. Mỗi kết quả tìm kiếm đi kèm với một `Action`. Handler tương ứng (như `OpenFileActionHandler` hay `UrlActionHandler`) sẽ chịu trách nhiệm thực thi. Điều này cho phép một item có thể có nhiều hành động (ví dụ: Mở file, Mở thư mục chứa file, Copy đường dẫn).
*   **Rescan Orchestrator:** Một hệ thống lập lịch (`TaskScheduler`) để tự động quét lại dữ liệu ứng dụng/file theo định kỳ mà không gây treo UI.
*   **Vibrancy & Background Material:** Tận dụng các tính năng cao cấp của Electron để tạo hiệu ứng kính mờ (Mica/Acrylic trên Windows, Vibrancy trên macOS) trực tiếp từ code Main process thông qua `VibrancyProvider`.

### 4. Luồng Hoạt động Hệ thống (System Workflow)

1.  **Giai đoạn Bootstrapping:**
    *   Main Process khởi động -> Khởi tạo `ModuleRegistry`.
    *   Đăng ký các lõi hệ thống (Settings, Logger, FileSystem).
    *   `ExtensionLoader` quét và kích hoạt các Extension được người dùng bật.
    *   Khởi tạo phím tắt toàn cầu (thường là `Alt + Space`).

2.  **Giai đoạn Indexing:**
    *   `RescanOrchestrator` kích hoạt các Extension.
    *   Mỗi Extension (ví dụ: `ApplicationSearch`) quét hệ điều hành và trả về danh sách `SearchResultItem`.
    *   Kết quả được lưu vào `SearchIndex` trong bộ nhớ.

3.  **Giai đoạn Tương tác người dùng:**
    *   Người dùng nhấn phím tắt -> `SearchWindow` hiện ra.
    *   Người dùng gõ phím -> Renderer gửi từ khóa qua IPC.
    *   Main Process nhận từ khóa -> Chạy qua thuật toán Fuzzy Search -> Kết hợp với "Instant Results" (kết quả tính toán trực tiếp) -> Trả về danh sách đã được sắp xếp độ ưu tiên.

4.  **Giai đoạn Thực thi:**
    *   Người dùng nhấn `Enter` -> Renderer gửi `SearchResultItemAction` về Main.
    *   Main tìm `ActionHandler` phù hợp -> Thực thi (ví dụ: `shell.openPath`) -> Ẩn cửa sổ.

### Tổng kết
Ueli là một dự án mẫu mực về cách tổ chức mã nguồn Electron. Nó giải quyết được bài toán hiệu năng (vốn là điểm yếu của Electron) bằng cách quản lý Index thông minh và tận dụng tối đa sức mạnh của TypeScript để duy trì một hệ thống plugin đồ sộ nhưng rất ổn định.