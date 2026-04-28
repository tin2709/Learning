Dưới đây là phân tích chi tiết về dự án **Unit-3**, một bản "rice" (tùy biến giao diện) cao cấp dành cho Arch Linux dựa trên chủ đề NieR:Automata.

---

### 1. Công nghệ cốt lõi (Core Technology)

Dự án này là sự kết hợp hiện đại giữa các công nghệ Wayland và framework UI mạnh mẽ:

*   **Hyprland (C++):** Window Manager (WM) dựa trên giao thức Wayland, cung cấp khả năng quản lý cửa sổ dạng lát gạch (tiling) với hiệu ứng đồ họa mượt mà (blur, animations).
*   **Quickshell (Qt6/QML):** Đây là "linh hồn" của bản rice này. Thay vì dùng các công cụ đơn giản như Eww hay Rofi, tác giả sử dụng Quickshell để viết các widget bằng ngôn ngữ **QML (Qt Modeling Language)**. QML cho phép tạo UI phức tạp, hỗ trợ tăng tốc phần cứng và xử lý logic bằng JavaScript.
*   **Qt Multimedia & FFmpeg:** Sử dụng để tích hợp các đoạn video transition (.mp4) trực tiếp vào giao diện (như hiệu ứng sóng pixel khi khóa màn hình).
*   **Python (Numpy/Pillow):** Được dùng để tính toán toán học và mô phỏng các hiệu ứng hình ảnh phức tạp (Pixel Wave) mà Shell script hay CSS thông thường không thể làm được.
*   **Waybar (C++/GTK):** Thanh trạng thái chính, nhẹ và ổn định.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Unit-3 không chỉ dừng lại ở việc sao chép file cấu hình, mà nó mang tư duy của một hệ thống phần mềm:

*   **Tính module hóa (Modularity):**
    *   Cấu trúc thư mục tách bạch: `widgets/` chứa các thành phần UI, `components/` chứa các phần tử dùng chung (NierButton, Scanlines), `settings/` chứa các cấu hình hệ thống toàn cục.
    *   Việc tách `user.conf` khỏi `hyprland.conf` giúp người dùng cập nhật script mà không mất cấu hình cá nhân.
*   **Tận dụng Layer-shell Protocol:** Các widget được phân lớp (Layer) rõ ràng (Overlay cho lockscreen, Top cho bar/notification, Ignore cho menu) để đảm bảo tương tác đúng với các cửa sổ ứng dụng khác.
*   **Tư duy "Logic-in-UI":** Sử dụng Singleton (như `Settings.qml`, `Theme.qml`) để quản lý màu sắc và kích thước toàn hệ thống, giúp việc thay đổi theme trở nên cực kỳ dễ dàng.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Mô phỏng vật lý trong QML/Python:**
    *   Trong `pixel_wave.py`, tác giả sử dụng **Numpy** để tính toán ma trận điểm ảnh, áp dụng công thức lò xo (Spring Physics - `SPRING_K`, `SPRING_D`) để tạo độ nảy khi các pixel "sóng" xuất hiện.
*   **Giao tiếp liên tiến trình (IPC) qua File-trigger:**
    *   Một kỹ thuật thú vị là sử dụng các file tạm như `/tmp/qs-toggle` hoặc `/tmp/qs-menu`. Hyprland sẽ "echo" vào file này, và Quickshell sử dụng `Process` kết hợp `Timer` trong QML để theo dõi sự thay đổi của file và kích hoạt UI.
*   **Tích hợp PAM (Pluggable Authentication Modules):**
    *   Sử dụng `pamtester` để kết nối giao diện QML (Lockscreen) với hệ thống xác thực của Linux, cho phép đăng nhập an toàn bằng mật khẩu thực tế thông qua một script trung gian.
*   **Kỹ thuật Canvas Drawing:**
    *   Các thành phần như `CornerDeco.qml` hay `Scanlines.qml` không sử dụng ảnh PNG mà được vẽ trực tiếp bằng mã lệnh (Canvas API), giúp giao diện cực kỳ sắc nét ở bất kỳ độ phân giải nào (HiDPI support).

### 4. Luồng hoạt động hệ thống (System Flow)

Luồng hoạt động của Unit-3 được thiết kế rất chặt chẽ:

1.  **Giai đoạn Khởi động (Boot-up):**
    *   Hyprland khởi chạy -> Gọi `install.sh` (nếu cài mới) để kiểm tra các phụ thuộc (dependencies).
    *   `wave-check.sh` kiểm tra xem các video hiệu ứng đã được tạo chưa. Nếu chưa, nó sẽ chạy các script Python để **tự động render video** phù hợp với độ phân giải màn hình hiện tại.
2.  **Giai đoạn Vận hành (Runtime):**
    *   `qs` (Quickshell) khởi chạy file `shell.qml`. File này đóng vai trò "nhà điều phối", quản lý việc hiển thị các widget dựa trên các tín hiệu (signals).
    *   Waybar hiển thị các thông tin hệ thống và Pomodoro.
3.  **Giai đoạn Tương tác (Interaction):**
    *   Khi người dùng nhấn phím tắt (ví dụ `SUPER+L` để khóa):
        *   `lock.sh` được gọi -> Chạy script Python tạo background -> Gọi `hyprlock`.
        *   Đồng thời, Quickshell hiển thị `lockscreen.qml` với hiệu ứng video "wave_reveal" che phủ màn hình.
4.  **Xác thực (Authentication):**
    *   Người dùng nhập mật khẩu vào widget QML -> QML gửi mật khẩu sang `authProc` (bash) -> `pamtester` xác thực -> Nếu OK, gửi tín hiệu "OK" về QML để chạy video "wave_hide" và mở khóa màn hình.

### Tổng kết
**Unit-3** là một ví dụ điển hình về việc đưa kỹ thuật phần mềm vào tùy biến Linux. Nó không chỉ là "làm đẹp" mà là xây dựng một **Giao diện Người dùng Động (Dynamic UI)** có khả năng tự thích nghi, tự render tài nguyên và tương tác sâu với nhân hệ thống.