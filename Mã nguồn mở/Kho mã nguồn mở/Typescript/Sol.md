Dựa trên mã nguồn của dự án **Sol**, đây là một ứng dụng launcher cho macOS cực kỳ tối ưu, kết hợp giữa sự linh hoạt của React Native và sức mạnh can thiệp sâu của hệ thống macOS thông qua Swift/C++.

Dưới đây là phân tích chi tiết:

### 1. Phân tích Công nghệ Cốt lõi (Core Technology Stack)

*   **Framework chính:** `react-native-macos` (một bản fork của React Native dành riêng cho desktop). Đây là lựa chọn hiếm thấy, giúp tận dụng hệ sinh thái React nhưng vẫn render ra các component native của macOS.
*   **Ngôn ngữ:** TypeScript (chiếm 63%) cho logic UI và Swift/Objective-C++ (chiếm hơn 30%) cho các tính năng hệ thống.
*   **Giao diện:** `NativeWind` (Tailwind CSS cho React Native). Sol là một trong những dự án desktop hiếm hoi áp dụng Tailwind một cách triệt để để tạo ra UI hiện đại và nhất quán.
*   **Giao tiếp JS-Native (JSI):** Thay vì dùng Bridge cũ chậm chạp, Sol sử dụng **JSI (JavaScript Interface)** để gọi trực tiếp các hàm C++/Swift từ JavaScript. Điều này cực kỳ quan trọng cho một launcher cần độ trễ bằng 0.
*   **Lưu trữ & Indexing:** 
    *   `react-native-mmkv`: Lưu trữ key-value siêu nhanh cho cấu hình.
    *   `SQLite3`: Được dùng ở lớp Native để lập chỉ mục (index) file hệ thống, giúp tìm kiếm file tức thì.

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **NSPanel thay vì NSWindow:** Sol sử dụng `NSPanel` (được cấu hình trong `Panel.swift`). Đặc điểm của Panel là có thể hiển thị trên cùng (floating), không chiếm focus của ứng dụng khác một cách thô bạo (non-activating), giúp nó hoạt động giống hệt Spotlight hay Raycast.
*   **Widget-based Architecture:** Các tính năng như Calendar, Clipboard, Translator, Emoji Picker được tách thành các "Widget" độc lập trong `src/widgets`. Mỗi widget có một `Store` riêng (MobX) để quản lý trạng thái.
*   **Kiến trúc Đẩy (Event-Driven):** Lớp Native (Swift) liên tục lắng nghe các sự kiện hệ thống (phím tắt, thay đổi clipboard, ứng dụng mới cài đặt) thông qua `FSEvents` và `Apple Events`, sau đó đẩy dữ liệu lên JS thông qua `SolEmitter`.
*   **Hybrid Storage:** Logic tìm kiếm ứng dụng và bookmark nằm ở JS để linh hoạt, nhưng logic quét hàng triệu file nằm ở Native (SQLite) để đảm bảo hiệu năng.

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Techniques)

*   **Can thiệp sâu vào Accessibility API:** Sol sử dụng `AccessibilityElement.swift` để điều khiển cửa sổ của các ứng dụng khác. Nó có thể thay đổi kích thước, di chuyển cửa sổ giữa các màn hình (Window Manager) - một tính năng mà các ứng dụng React Native thông thường không thể làm được.
*   **Hyper Key & Caps Lock Remapping:** Trong `HotKeyManager.swift`, Sol can thiệp vào tầng HID (Human Interface Device) để biến phím Caps Lock thành phím "Hyper" (kết hợp Cmd+Ctrl+Opt+Shift). Đây là kỹ thuật lập trình hệ thống cấp thấp (Low-level system programming).
*   **Folder Watcher JSI:** Sol triển khai `FolderWatcherJSI.mm` bằng C++ để lắng nghe thay đổi file. Khi bạn thêm một file vào thư mục được theo dõi, nó cập nhật UI gần như ngay lập tức mà không cần polling (quét định kỳ).
*   **Custom AppleScript Runner:** Sol cho phép thực thi các đoạn mã AppleScript để điều khiển Spotify, Apple Music hoặc thay đổi giao diện hệ thống (Dark mode) một cách mượt mà thông qua `AppleScriptHelper.swift`.

### 4. Luồng Hoạt động Hệ thống (System Workflow)

1.  **Khởi tạo:** `AppDelegate.swift` khởi động. Nó đăng ký phím tắt toàn cầu (Global Shortcut) thông qua thư viện `HotKey`.
2.  **Kích hoạt:** Khi người dùng nhấn phím tắt (thường là Cmd+Space):
    *   Native bắt được sự kiện -> gọi `PanelManager.shared.showWindow()`.
    *   JS nhận sự kiện `onShow` -> focus vào `MainInput.tsx`.
3.  **Xử lý Nhập liệu:** Khi người dùng gõ phím:
    *   JS Store lọc danh sách ứng dụng/lệnh có sẵn.
    *   Nếu query bắt đầu bằng dấu `/`, nó chuyển sang chế độ File Search, gọi xuống SQLite ở lớp Native để tìm kiếm file.
4.  **Thực thi hành động:**
    *   Nếu chọn một ứng dụng: Native dùng `NSWorkspace.shared.open`.
    *   Nếu chọn một lệnh hệ thống (ví dụ: "Lock Screen"): Native thực thi AppleScript hoặc Shell script tương ứng.
5.  **Clipboard Monitoring:** Một luồng chạy ngầm ở Native liên tục theo dõi phím Cmd+C. Mỗi khi có dữ liệu mới (văn bản hoặc hình ảnh), nó sẽ lưu vào cache và đẩy vào `clipboard.store.tsx` để người dùng có thể xem lại lịch sử.

### Tổng kết
Sol là một ví dụ mẫu mực về việc **"Native hóa" một Web Framework**. Nó không cố gắng ép React Native làm mọi thứ, mà dùng React Native để làm UI cực nhanh, còn tất cả những gì liên quan đến hiệu năng và hệ thống đều được đẩy xuống lớp C++ và Swift thông qua JSI.