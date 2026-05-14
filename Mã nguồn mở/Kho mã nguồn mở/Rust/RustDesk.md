Dựa trên các tệp tin cấu trúc dự án của **RustDesk**, đây là phân tích chuyên sâu về kiến trúc và kỹ thuật của một trong những phần mềm điều khiển máy tính từ xa mã nguồn mở phổ biến nhất hiện nay.

---

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

RustDesk là một minh chứng điển hình cho việc kết hợp hiệu suất cực cao của Rust với tính linh hoạt của Flutter:

*   **Ngôn ngữ lập trình:**
    *   **Rust (Cốt lõi):** Chiếm ~67%, xử lý toàn bộ các tác vụ nặng: mã hóa/giải mã video (Codec), giao thức mạng (P2P, TCP/UDP), truyền file, và xử lý hệ thống cấp thấp.
    *   **Dart/Flutter (UI):** Chiếm ~25%, được sử dụng cho giao diện người dùng hiện đại trên tất cả nền tảng (Windows, macOS, Linux, Android, iOS).
    *   **C++/C/Kotlin:** Dùng để viết các "bridge" (cầu nối) hoặc hook vào các API đặc thù của hệ điều hành mà Rust chưa hỗ trợ trực tiếp.
*   **Xử lý luồng & Bất đồng bộ:** Sử dụng **Tokio Runtime**. Đây là lựa chọn tối ưu cho ứng dụng yêu cầu IO mạng lớn và độ trễ thấp.
*   **Xử lý Video/Audio:** Tích hợp các thư viện nén hàng đầu qua `vcpkg`: **libvpx** (VP8/VP9), **aom** (AV1), **Opus** (Audio) và **FFmpeg** để hỗ trợ tăng tốc phần cứng (H.264/H.265).
*   **NAT Traversal:** Sử dụng kỹ thuật **TCP/UDP Hole Punching** để thiết lập kết nối trực tiếp giữa hai máy tính mà không cần qua server trung gian nếu điều kiện mạng cho phép.

---

### 2. Tư duy Kiến trúc (Architectural Philosophy)

Kiến trúc của RustDesk được thiết kế theo mô hình **Service-Client tách biệt**:

*   **Tách biệt đặc quyền (Privilege Separation):**
    *   Hệ thống gồm một **Service** chạy ngầm (thường có quyền root/SYSTEM) để điều khiển chuột, bàn phím và chụp màn hình.
    *   Giao diện người dùng (**UI**) chạy dưới quyền user thông thường. Hai thành phần này giao tiếp qua **IPC (Inter-Process Communication)**.
*   **Thư viện hóa (Library-driven):** Các tính năng quan trọng được tách thành các crate riêng biệt trong thư mục `libs/`:
    *   `hbb_common`: Chứa cấu hình dùng chung, giao thức Protobuf, và các hàm tiện ích.
    *   `scrap`: Thư viện chuyên biệt để chụp màn hình (hỗ trợ DirectX trên Windows, X11/Wayland trên Linux).
    *   `enigo`: Mô phỏng input (chuột/phím) đa nền tảng.
*   **Kiến trúc P2P-First:** Hệ thống ưu tiên kết nối ngang hàng (Peer-to-Peer). Server (hbbs/hbbr) chỉ đóng vai trò "môi giới" (rendezvous) hoặc chuyển tiếp (relay) khi không thể xuyên thủng tường lửa.

---

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Techniques)

*   **Flutter Rust Bridge:** Dự án sử dụng `flutter_rust_bridge` để gọi code Rust từ Dart. Điều này giúp UI của Flutter mượt mà nhưng vẫn tận dụng được tốc độ xử lý byte của Rust cho dữ liệu video.
*   **Quản lý bộ nhớ an toàn:** Tuân thủ nghiêm ngặt quy tắc của Rust (không dùng `unwrap()` bừa bãi, ưu tiên mượn - borrowing hơn là `clone()`), giúp tránh các lỗi rò rỉ bộ nhớ hoặc crash hệ thống - vốn là tử huyệt của các phần mềm điều khiển từ xa viết bằng C++.
*   **Hỗ trợ Wayland sâu:** Trong khi nhiều phần mềm từ xa gặp khó khăn với Wayland trên Linux, RustDesk có logic xử lý riêng (`libs/scrap/src/wayland.rs`) để tương tác với các giao thức bảo mật mới của Linux.
*   **Hệ thống Build đa năng:** Tệp `build.py` và `build.rs` cực kỳ phức tạp, quản lý việc tải các thư viện động (.dll, .so) và tích hợp các bộ công cụ như Sciter (cũ) và Flutter (mới) một cách tự động.

---

### 4. Luồng Hoạt động Hệ thống (System Workflows)

#### A. Luồng thiết lập kết nối (Connection Flow)
1.  **Đăng ký:** Khi mở app, máy khách gửi ID và thông tin mạng đến **Rendezvous Server** (hbbs).
2.  **Yêu cầu:** Người dùng nhập ID máy cần điều khiển. Client gửi yêu cầu "kết nối" tới hbbs.
3.  **Hole Punching:** hbbs cung cấp thông tin IP/Port của cả hai máy. Hai máy thực hiện bắt tay UDP/TCP để tạo đường ống trực tiếp.
4.  **Relay (Dự phòng):** Nếu không thể punch hole, hbbs chỉ định một **Relay Server** (hbbr) để chuyển tiếp traffic nén giữa hai máy.

#### B. Luồng truyền tải hình ảnh (Streaming Flow)
1.  **Capture:** Thư viện `scrap` chụp màn hình ở tốc độ cao (60fps).
2.  **Encode:** Frame hình ảnh được đưa vào encoder (VP9/AV1) để nén. Chỉ các phần ảnh thay đổi mới được gửi đi để tiết kiệm băng thông.
3.  **Transport:** Dữ liệu nén được đóng gói qua giao thức tùy chỉnh của RustDesk và gửi đi qua đường ống đã thiết lập.
4.  **Render:** Máy nhận giải mã (decode) và đẩy vào một `Texture` của Flutter để hiển thị lên màn hình người dùng.

#### C. Luồng điều khiển (Input Flow)
1.  Người dùng click chuột trên UI Flutter.
2.  Tọa độ được chuẩn hóa và gửi về máy bị điều khiển qua kênh lệnh (command channel).
3.  Service của RustDesk nhận lệnh và dùng `enigo` để ra lệnh cho hệ điều hành thực hiện hành vi click chuột thực tế.

---

### Tổng kết
RustDesk là một hệ thống **Native** mạnh mẽ, tận dụng tối đa các đặc tính của Rust để giải quyết bài toán khó nhất của Remote Desktop: **Độ trễ (Latency)** và **Băng thông (Bandwidth)**. Việc chuyển dịch sang Flutter giúp họ có một codebase UI duy nhất cho mọi nền tảng mà không hy sinh hiệu năng xử lý ở tầng dưới.