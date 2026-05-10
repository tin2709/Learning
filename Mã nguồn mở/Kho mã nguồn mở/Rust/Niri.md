Dựa trên mã nguồn và tài liệu kỹ thuật của dự án **Niri**, một "scrollable-tiling Wayland compositor" được viết bằng Rust, dưới đây là bản phân tích chuyên sâu:

### 1. Phân tích Công nghệ Cốt lõi (Core Technology)

*   **Smithay Framework:** Đây là "xương sống" của Niri. Thay vì viết lại từ đầu các giao thức Wayland phức tạp, Niri sử dụng Smithay – một thư viện cung cấp các khối xây dựng (building blocks) để tạo compositor. Điều này giúp Niri tập trung vào logic layout thay vì quản lý bộ đệm cấp thấp.
*   **Giao thức Wayland hiện đại:** Niri hỗ trợ đầy đủ các giao thức quan trọng như `layer-shell` (cho thanh bar/panel), `gamma-control`, `screencopy`, và `xdg-activation`. Đặc biệt, việc tích hợp **Xwayland-satellite** cho phép chạy ứng dụng X11 một cách cô lập, giữ cho lõi compositor sạch sẽ và an toàn hơn.
*   **KDL Configuration Language:** Niri sử dụng ngôn ngữ cấu hình KDL. Đây là lựa chọn độc đáo so với TOML hay JSON, vì KDL có cấu trúc phân cấp rõ ràng, rất phù hợp để định nghĩa các thuộc tính phức tạp như `window-rule` hay các khối `animations`.
*   **Rust làm ngôn ngữ chủ đạo:** Tận dụng tối đa tính an toàn bộ nhớ (memory safety) và hiệu năng cao. Việc sử dụng `anyhow`, `tracing`, và `tokio/async-io` (cho D-Bus) cho thấy một hệ thống hiện đại, xử lý bất đồng bộ tốt.

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Scrollable Tiling (Lát gạch cuộn):** Đây là tư duy đột phá nhất. Thay vì cố gắng ép tất cả cửa sổ vào một màn hình (gây resize liên tục), Niri sắp xếp chúng trên một dải băng ngang vô hạn (infinite strip). Cửa sổ mới mở không bao giờ làm thay đổi kích thước cửa sổ cũ.
*   **Độc lập hóa Monitor:** Mỗi monitor có một dải cửa sổ riêng biệt. Cửa sổ không thể "tràn" từ monitor này sang monitor khác một cách vô tình. Tư duy này giúp quản lý đa màn hình cực kỳ tường minh.
*   **Workspaces động theo phong cách GNOME:** Các không gian làm việc được sắp xếp theo chiều dọc. Mỗi monitor có tập hợp workspace độc lập, giúp người dùng tập trung vào từng tác vụ cụ thể trên từng màn hình.
*   **Thiết kế Immediate & Predictable:** Triết lý của Niri là các hành động phải áp dụng ngay lập tức. Layout được tính toán sao cho cửa sổ đang focus không bao giờ tự ý di chuyển do tác động từ các cửa sổ bên ngoài tầm nhìn.

### 3. Kỹ thuật Lập trình Đặc sắc (Distinctive Techniques)

*   **Hệ thống Clock tùy chỉnh (LazyClock & AdjustableClock):** Để xử lý hoạt ảnh (animation) mượt mà, Niri cài đặt một hệ thống Clock cho phép:
    *   Dự đoán thời điểm frame tiếp theo xuất hiện trên màn hình.
    *   Giữ nguyên một timestamp trong suốt một lần lặp (iteration) của vòng lặp sự kiện để tránh sai số logic.
    *   Hỗ trợ "slowdown" (làm chậm hoạt ảnh) toàn hệ thống để debug shader.
*   **Xử lý Layout số thực (Fractional Layout Rounding):** Niri tính toán vị trí cửa sổ bằng số thực (logical coordinates) nhưng có kỹ thuật làm tròn (rounding) cực kỳ khắt khe trước khi render. Điều này đảm bảo các đường viền (border) và khoảng cách (gaps) luôn sắc nét (pixel-perfect), không bị mờ do hiện tượng răng cưa khi dùng scale không nguyên (ví dụ scale 1.5x).
*   **Vòng lặp Redraw tối ưu (Stateful Redraw Loop):** Sử dụng một máy trạng thái (Idle -> Queued -> WaitingForVBlank) để kiểm soát việc vẽ lại. Nếu không có thay đổi (damage), hệ thống sẽ không đánh thức GPU, giúp tiết kiệm điện năng đáng kể.
*   **Randomized Property Testing:** Tác giả sử dụng `proptest` để "fuzz" (thử nghiệm ngẫu nhiên) công cụ layout. Điều này đảm bảo rằng không có bất kỳ tổ hợp hành động nào (mở, đóng, kéo, thả cửa sổ) có thể đưa hệ thống vào trạng thái lỗi hoặc crash.

### 4. Luồng Hoạt động Hệ thống (System Workflow)

1.  **Giai đoạn Khởi tạo:**
    *   Đọc cấu hình KDL -> Khởi tạo backend (DRM/KMS cho máy thật hoặc Winit cho môi trường cửa sổ).
    *   Thiết lập socket Wayland và các giao thức cần thiết qua Smithay.
2.  **Xử lý Input:**
    *   Sự kiện từ chuột/bàn phím/touchpad đi qua `libinput`.
    *   Niri ánh xạ các phím tắt (binds) vào các `Action` (như `focus-column-left`, `spawn`).
    *   Touchpad gestures được xử lý đặc biệt để đồng bộ với hiệu ứng cuộn dải băng cửa sổ.
3.  **Tính toán Layout:**
    *   Khi một cửa sổ yêu cầu thay đổi (hoặc cửa sổ mới mở), Niri cập nhật trạng thái dải băng ngang.
    *   Tính toán vị trí logic dựa trên `preset-column-widths` và `gaps`.
4.  **Render & Presentation:**
    *   Máy trạng thái Redraw phát hiện vùng bị thay đổi (damage).
    *   Gửi lệnh vẽ đến GPU (OpenGL/GLES).
    *   Áp dụng các Custom Shader (nếu có) cho hiệu ứng đóng/mở cửa sổ.
    *   Chờ tín hiệu VBlank từ màn hình để trình diễn frame mới, đảm bảo không bị xé hình (tearing).
5.  **Tương tác Ngoại vi:**
    *   Thông qua IPC socket, công cụ `niri msg` có thể gửi lệnh điều khiển từ bên ngoài (scripting) vào thẳng core của compositor.

### Tổng kết
Niri không chỉ là một Window Manager, nó là một thí nghiệm thành công về **UX cuộn** trong môi trường desktop. Việc kết hợp giữa tính an toàn của **Rust**, sức mạnh của **Smithay** và sự tỉ mỉ trong **xử lý đồ họa pixel-perfect** khiến nó trở thành một trong những compositor Wayland thú vị nhất hiện nay.