Dựa trên mã nguồn và tài liệu của kho lưu trữ **Kando**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và cách thức hoạt động của dự án này:

### 1. Công nghệ cốt lõi (Core Tech Stack)
Kando là một ứng dụng desktop hiện đại được xây dựng trên nền tảng Web kết hợp với sức mạnh hệ thống:

*   **Framework chính:** [Electron](https://www.electronjs.org/) - Cho phép xây dựng ứng dụng cross-platform (Windows, macOS, Linux) bằng HTML, CSS và JS.
*   **Ngôn ngữ:** 
    *   **TypeScript (79.3%):** Toàn bộ logic nghiệp vụ được viết bằng TS để đảm bảo an toàn về kiểu dữ liệu.
    *   **C/C++/Objective-C++:** Sử dụng để viết các **Native Modules** (Add-ons) can thiệp sâu vào hệ điều hành (như bắt phím tắt toàn cục, quản lý cửa sổ trên Wayland/X11/Win32/macOS).
*   **Giao diện (UI/UX):**
    *   **React:** Được sử dụng cho cửa sổ Cấu hình (Settings Window) để quản lý trạng thái phức tạp.
    *   **Vanilla JS/TS & SCSS:** Được dùng cho trình kết xuất Menu (Menu Renderer) để tối ưu hóa hiệu năng, đảm bảo các hiệu ứng chuyển động (animation) mượt mà nhất có thể.
*   **Công cụ build:** Webpack (Bundling), Electron Forge (Packaging), và CMake.js (để biên dịch mã nguồn C++ sang Node.js Addons).

### 2. Tư duy kiến trúc (Architectural Patterns)
Kando áp dụng kiến trúc **Hybrid** giữa ứng dụng Web và Native:

*   **Backend Abstraction Layer (Lớp trừu tượng hóa hệ điều hành):** Đây là điểm hay nhất của Kando. Trong `src/main/backends`, tác giả tạo ra một interface chung cho mọi hệ điều hành. Dù bạn dùng Windows, macOS hay các môi trường Linux khác nhau (GNOME, KDE, Hyprland, Niri), ứng dụng sẽ gọi các hàm giống nhau, còn phần thực thi native bên dưới sẽ tự điều chỉnh.
*   **Phân tách Process (IPC):**
    *   **Main Process:** Quản lý vòng đời ứng dụng, đăng ký phím tắt hệ thống và thực thi các hành động (chạy lệnh, giả lập phím bấm).
    *   **Renderer Process:** Tách biệt hoàn toàn giữa cửa sổ hiển thị Menu (tối ưu tốc độ) và cửa sổ Cài đặt (tối ưu tính năng).
*   **Kiến trúc hướng Plugin/Registry:** Các loại mục menu (Command, URI, Hotkey, v.v.) và bộ giải mã icon được quản lý qua các `Registry`. Điều này cho phép dễ dàng mở rộng thêm các tính năng mới mà không làm xáo trộn mã nguồn cũ.

### 3. Các kỹ thuật chính (Key Techniques)
*   **Marking Menu & Gestures:** Kando triển khai thuật toán tính toán góc quay và khoảng cách để xác định lựa chọn của người dùng dựa trên chuyển động chuột thay vì chỉ là cú click.
*   **Turbo Mode:** Kỹ thuật phát hiện trạng thái phím bổ trợ (Ctrl/Shift/Alt) đang được giữ để cho phép người dùng lướt chuột qua các mục và chọn ngay lập tức mà không cần click.
*   **Native Interop (N-API):** Sử dụng `node-addon-api` để gọi mã nguồn C++ trực tiếp từ Node.js, xử lý các tác vụ như:
    *   Lấy tiêu đề cửa sổ đang active.
    *   Giả lập các tổ hợp phím hệ thống (Keyboard simulation).
    *   Tương tác với giao thức Wayland (Linux) hoặc Windows API.
*   **Theming Engine:** Sử dụng các biến CSS (CSS Variables) và JSON5 để người dùng có thể tùy biến hoàn toàn giao diện menu (màu sắc, kích thước, độ mờ).

### 4. Tóm tắt luồng hoạt động (Workflow Summary)
Luồng hoạt động của Kando diễn ra như sau:

1.  **Khởi tạo:** Khi ứng dụng chạy, **Main Process** sẽ tải cấu hình từ tệp JSON và đăng ký **Global Shortcut** (mặc định là `Ctrl + Space`) thông qua Backend native tương ứng.
2.  **Kích hoạt:** Khi người dùng nhấn phím tắt:
    *   Backend native báo cho Main Process.
    *   Main Process xác định vị trí con trỏ chuột hiện tại.
    *   Cửa sổ **Menu Renderer** được hiển thị ngay lập tức tại vị trí đó (thường là một cửa sổ xuyên thấu, tràn màn hình).
3.  **Tương tác:**
    *   Người dùng di chuyển chuột hoặc thực hiện cử chỉ (gesture). 
    *   Renderer tính toán sự thay đổi vị trí và cập nhật trạng thái hiển thị (zoom các "miếng bánh", hiện menu con).
    *   Nếu ở **Turbo Mode**, việc thả phím tắt sẽ kích hoạt mục đang chọn. Ở chế độ thường, việc thả chuột hoặc click sẽ kích hoạt.
4.  **Thực thi:**
    *   Renderer gửi ID của mục được chọn về Main Process qua **IPC**.
    *   Main Process tra cứu hành động tương ứng (ví dụ: chạy lệnh shell `firefox`, hoặc gửi phím `Ctrl+C`).
    *   Sau khi thực thi, cửa sổ menu sẽ đóng lại và trả lại tiêu điểm (focus) cho ứng dụng trước đó.

### Kết luận
Kando không chỉ là một ứng dụng Electron thông thường; nó là một ví dụ điển hình về việc kết hợp **Web Tech** (để có UI đẹp, linh hoạt) với **Native C++** (để có hiệu suất và khả năng can thiệp hệ thống sâu). Đây là một dự án có tính tổ chức mã nguồn rất cao, sạch sẽ và chuyên nghiệp.

![alt text](image-4.png)