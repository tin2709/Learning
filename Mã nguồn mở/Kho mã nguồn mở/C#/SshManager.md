Dựa trên mã nguồn và tài liệu của dự án **SshManager**, dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của ứng dụng này:

### 1. Công nghệ cốt lõi (Core Technology Stack)
SshManager được xây dựng trên nền tảng .NET hiện đại, tối ưu cho Windows:
*   **Framework:** **.NET 8/9** kết hợp với **WPF (Windows Presentation Foundation)**.
*   **Giao diện (UI):** Sử dụng thư viện **WPF-UI**, mang lại ngôn ngữ thiết kế Fluent Design giống Windows 11.
*   **Terminal Engine:** Sử dụng **xterm.js** chạy bên trong **WebView2 (Microsoft Edge)**. Đây là kỹ thuật giúp ứng dụng có hiệu năng render terminal cực cao, hỗ trợ đầy đủ các tính năng hiện đại như WebGL và Unicode.
*   **Giao thức kết nối:**
    *   **SSH:** Sử dụng thư viện `SSH.NET`.
    *   **Serial:** Kết hợp `System.IO.Ports` và `RJCP.SerialPortStream` để tối đa hóa khả năng tương thích với các thiết bị phần cứng.
*   **Cơ sở dữ liệu:** **SQLite** thông qua **Entity Framework Core (EF Core)**.
*   **Bảo mật:** Sử dụng **Windows DPAPI** (Data Protection API) để mã hóa mật khẩu theo từng user Windows cụ thể.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Dự án áp dụng kiến trúc đa tầng (Layered Architecture) với sự phân tách trách nhiệm (Separation of Concerns) rõ ràng:
*   **SshManager.Core:** Chứa các Domain Models thuần túy, Enums và các định nghĩa ngoại lệ. Đây là tầng không phụ thuộc vào bất kỳ thư viện bên thứ ba nào.
*   **SshManager.Security:** Tập trung vào mã hóa (DPAPI), quản lý bộ nhớ đệm thông tin xác thực (Credential Cache) và xử lý chuyển đổi định dạng khóa SSH (PPK sang OpenSSH).
*   **SshManager.Data:** Quản lý persistence dữ liệu. Sử dụng **Repository Pattern** để trừu tượng hóa các truy vấn EF Core, giúp tầng ứng dụng không cần biết về chi tiết của SQL.
*   **SshManager.Terminal:** Đây là tầng logic nghiệp vụ phức tạp nhất. Nó sử dụng **Bridge Pattern** để kết nối luồng dữ liệu thô (raw stream) từ SSH/Serial vào giao diện xterm.js thông qua một lớp trung gian (`SshTerminalBridge`, `SerialTerminalBridge`).
*   **SshManager.App:** Tầng Presentation thực hiện theo mô hình **MVVM (Model-View-ViewModel)**. Sử dụng `CommunityToolkit.Mvvm` với Source Generators để giảm thiểu mã lặp (boilerplate code).

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)
*   **Dependency Injection (DI):** Sử dụng `Microsoft.Extensions.Hosting` để quản lý vòng đời của hơn 60 ViewModels và 50 Services.
*   **Xử lý bất đồng bộ (Async/Await):** Áp dụng triệt để `ConfigureAwait(false)` và lan truyền `CancellationToken` xuống tận các tầng sâu nhất để đảm bảo giao diện không bị treo (non-blocking UI).
*   **Quản lý tài nguyên hiệu quả:** Sử dụng `ArrayPool<byte>` trong các vòng lặp đọc dữ liệu từ Terminal để giảm áp lực cho Garbage Collector (GC), tránh tình trạng lag khi nhận lượng lớn log từ server.
*   **Security Hardening:** 
    *   Tự động xóa thông tin xác thực trong bộ nhớ khi máy tính bị khóa (Session Lock).
    *   Cấp quyền truy cập tệp tin (ACL) nghiêm ngặt cho các tệp khóa tạm thời (ví dụ khi tích hợp với 1Password).
*   **Fuzzy Matching:** Sử dụng thuật toán so khớp mờ cho tính năng "Quick Connect" (Ctrl+K), giúp tìm kiếm server nhanh chóng dựa trên tên hoặc IP.

### 4. Luồng hoạt động hệ thống (System Operation Flows)

#### A. Luồng kết nối Terminal:
1.  **Người dùng** kích hoạt lệnh kết nối từ `HostListPanel`.
2.  `SessionViewModel` yêu cầu `SshConnectionService` tạo một kết nối mới.
3.  Hệ thống kiểm tra **Credential Cache**; nếu không có, nó sẽ gọi `DpapiSecretProtector` để giải mã mật khẩu từ DB.
4.  Kết nối SSH được thiết lập. `SshTerminalBridge` bắt đầu đọc luồng dữ liệu từ server.
5.  Dữ liệu được đẩy vào `WebTerminalBridge` (thông qua JavaScript interop) để xterm.js hiển thị trên WebView2.

#### B. Luồng ghi lại phiên (Session Recording):
1.  Khi tính năng ghi được bật, `SessionRecorder` sẽ được gắn vào Bridge.
2.  Mọi dữ liệu đi qua Bridge sẽ được nhân bản sang `AsciinemaWriter`.
3.  Dữ liệu được định dạng theo chuẩn **ASCIINEMA v2** (bao gồm timestamp và nội dung) và lưu xuống đĩa theo thời gian thực.

#### C. Luồng đồng bộ hóa đám mây (Cloud Sync):
1.  Ứng dụng sử dụng `OneDrivePathDetector` để tìm thư mục đồng bộ.
2.  Dữ liệu được đóng gói thành `SyncData`, mã hóa bằng một Passphrase riêng biệt của người dùng (không phải mật khẩu Windows).
3.  Hệ thống xử lý xung đột (`SyncConflictResolver`) dựa trên dấu thời gian (Timestamp) để quyết định ghi đè hay gộp dữ liệu từ các thiết bị khác nhau.

### Tổng kết
SSHManager là một ứng dụng Windows Desktop chuẩn mực về kỹ thuật. Nó kết hợp sức mạnh của hệ sinh thái .NET cũ (WPF) với các công nghệ web hiện đại (xterm.js/WebView2) và các thực hành bảo mật tốt nhất của Windows (DPAPI), tạo ra một công cụ quản trị server mạnh mẽ và mượt mà.