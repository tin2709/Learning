Dựa trên mã nguồn và tài liệu của dự án **komorebi**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và kỹ thuật của hệ thống quản lý cửa sổ (Tiling Window Manager) này trên Windows:

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ lập trình:** **Rust (97.6%)**. Tận dụng khả năng quản lý bộ nhớ an toàn, hiệu suất cao và xử lý đa luồng mạnh mẽ của Rust để điều khiển các tài nguyên hệ thống Windows.
*   **Giao tiếp Hệ điều hành (OS Interface):** Sử dụng crate `windows-rs` (v0.62) để gọi trực tiếp các **Win32 API** và **DWM (Desktop Window Manager) API**. Đây là cầu nối giúp phần mềm can thiệp vào cách Windows quản lý các cửa sổ (HWND).
*   **Giao tiếp liên tiến trình (IPC):** 
    *   **Unix Domain Sockets (UDS):** Phương thức chính để `komorebic` (CLI) gửi lệnh đến `komorebi` (Daemon).
    *   **Named Pipes:** Cho phép các ứng dụng bên thứ ba (như status bar) đăng ký nhận thông báo sự kiện theo thời gian thực.
*   **Giao diện đồ họa (GUI):** Sử dụng `egui` / `eframe` cho các công cụ bổ trợ (`komorebi-gui`, `komorebi-shortcuts`), giúp giữ cho dung lượng nhẹ và tốc độ phản hồi nhanh.
*   **Xử lý bất đồng bộ & Luồng:** Sử dụng `crossbeam-channel` và `parking_lot` để quản lý các luồng sự kiện từ hệ thống mà không gây treo ứng dụng (deadlock detection).

### 2. Tư duy Kiến trúc (Architectural Mindset)

Komorebi được thiết kế theo kiến trúc **Client-Server** tách biệt, mô phỏng theo triết lý của các Tiling WM nổi tiếng trên Linux/macOS như `bspwm` hay `yabai`:

*   **Daemon (`komorebi.exe`):** Là trung tâm xử lý, chạy ngầm. Nó giữ "Source of Truth" (trạng thái duy nhất) về vị trí cửa sổ, các màn hình (monitor) và không gian làm việc (workspace).
*   **Controller (`komorebic.exe`):** Một CLI thuần túy. Nó không chứa logic quản lý cửa sổ mà chỉ gửi các lệnh (Socket Messages) đến Daemon.
*   **Hotkey Independent:** Komorebi **không** tự xử lý phím tắt. Nó dựa vào các công cụ bên thứ ba (`whkd`, AutoHotKey) để ánh xạ phím tắt người dùng vào các lệnh CLI.
*   **Cấu trúc dữ liệu phân cấp:** 
    *   **Monitor:** Màn hình vật lý.
    *   **Workspace:** Desktop ảo trên mỗi màn hình.
    *   **Container:** Một ô trong bố cục (layout), có thể chứa một hoặc nhiều cửa sổ (Stacking).
    *   **Window:** Cửa sổ ứng dụng thực tế.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **WinEvent Hooking:** Daemon đăng ký các hook sự kiện từ Windows (như `EVENT_OBJECT_SHOW`, `EVENT_OBJECT_DESTROY`, `EVENT_SYSTEM_FOREGROUND`) để biết khi nào một ứng dụng được mở, đóng hoặc lấy tiêu điểm, từ đó tự động sắp xếp lại bố cục.
*   **Hiding vs Cloaking:** Một kỹ thuật đặc biệt là sử dụng **Cloaking** (một tính năng ít tài liệu của Win32) để ẩn cửa sổ ở các workspace không hoạt động. Cách này tốt hơn `ShowWindow(SW_HIDE)` vì không làm mất trạng thái của các ứng dụng Electron hoặc gây lag khi chuyển đổi.
*   **Layout Engine (Binary Space Partitioning - BSP):** Sử dụng thuật toán chia không gian nhị phân để tự động tính toán kích thước cửa sổ sao cho lấp đầy màn hình.
*   **Trait-based Design:** Sử dụng các Trait như `Lockable` và `LockableSequence` (trong `lockable_sequence.rs`) để đảm bảo các cửa sổ đã được người dùng "khóa" vị trí sẽ không bị dịch chuyển khi tính toán lại layout.
*   **Ring Buffer:** Triển khai cấu trúc dữ liệu `Ring` (vòng tròn) cho phép người dùng xoay vòng tiêu điểm (focus) qua các cửa sổ/màn hình một cách vô tận (liền mạch giữa cuối danh sách và đầu danh sách).

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng xử lý một hành động của người dùng (ví dụ: chuyển focus sang trái):

1.  **Input:** Người dùng nhấn `Alt + H`.
2.  **Hotkey Daemon:** `whkd` bắt được phím tắt -> thực thi lệnh `komorebic.exe focus left`.
3.  **Client IPC:** `komorebic` kết nối tới Unix Domain Socket của Daemon, gửi một gói tin JSON chứa lệnh `FocusWindow(Left)`.
4.  **Daemon Processing:** 
    *   `komorebi.exe` nhận lệnh qua socket.
    *   Truy vấn trạng thái hiện tại để tìm cửa sổ nằm bên trái cửa sổ đang được focus.
    *   Gọi Win32 API (`SetForegroundWindow`, `SetWindowPos`) để thực hiện việc chuyển đổi.
    *   Cập nhật lại sơ đồ quản lý trong bộ nhớ.
5.  **Event Broadcast:** Daemon gửi thông báo qua **Named Pipes** cho tất cả các subscriber (như `komorebi-bar`).
6.  **UI Update:** Thanh status bar nhận được JSON trạng thái mới, cập nhật hiển thị (ví dụ: đổi màu icon workspace đang hoạt động).

### Tổng kết
**komorebi** là một ví dụ điển hình về việc sử dụng **Rust** để "hiện đại hóa" trải nghiệm người dùng trên Windows. Nó không thay thế trình quản lý cửa sổ mặc định của Windows mà chạy song song như một lớp điều khiển (orchestration layer), mang lại khả năng tùy biến cực cao cho những người dùng chuyên nghiệp (power users) yêu thích dòng lệnh và phím tắt.