Dưới đây là bản phân tích chi tiết về dự án **ScrcpyGUI v3**, một ứng dụng điều khiển Android hiệu năng cao được xây dựng trên nền tảng Tauri và React.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án sử dụng mô hình **Modern Desktop Hybrid**, kết hợp giữa sức mạnh hệ thống của Rust và sự linh hoạt giao diện của React:

*   **Tauri v2 (Rust Framework):** Đóng vai trò là "Bridge" giữa giao diện và hệ điều hành. So với Electron, Tauri giúp giảm dung lượng file thực thi (chỉ vài MB) và tiết kiệm RAM đáng kể vì sử dụng Webview có sẵn của hệ thống.
*   **React 19 & TypeScript:** Sử dụng phiên bản React mới nhất để tối ưu hiệu suất rendering. Giao diện được xây dựng với tư duy Component-based, đảm bảo tính tái sử dụng cao.
*   **Rust (Backend Logic):** Xử lý các tác vụ nặng như quản lý tiến trình (process management), thao tác file hệ thống, kết nối mạng (ADB over Wi-Fi) và tải xuống/giải nén binary.
*   **Tailwind CSS 4:** Cung cấp giải pháp styling dựa trên Utility-first, giúp xây dựng giao diện hiện đại, hỗ trợ Custom Theme Engine một cách dễ dàng qua các CSS Variables.
*   **Scrcpy & ADB (Core Engines):** Đây là hai "động cơ" thực sự nằm dưới lớp giao diện, đảm nhiệm việc stream màn hình và gửi lệnh điều khiển đến thiết bị Android.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của ScrcpyGUI v3 tập trung vào **Quản lý Tiến trình (Process Management)** và **Trạng thái Đồng bộ (State Synchronization)**:

*   **Process Ownership (Quyền sở hữu tiến trình):** Toàn bộ các tiến trình `scrcpy` và `adb` được quản lý tập trung tại Backend Rust thông qua `ScrcpyState`. Dự án sử dụng `Mutex<HashMap<String, Child>>` để lưu trữ các tiến trình đang chạy theo ID thiết bị, cho phép điều khiển đa thiết bị cùng lúc.
*   **Asynchronous IPC (Giao tiếp bất đồng bộ):** Thay vì chặn luồng giao diện (UI blocking), các tác vụ như quét thiết bị hoặc kết nối không dây được thực hiện bất đồng bộ. Rust sử dụng `tokio` để xử lý các luồng dữ liệu (logs, status) và đẩy ngược lên Frontend qua cơ chế `emit/listen`.
*   **Cross-platform Abstraction (Trừu tượng hóa đa nền tảng):** Mã nguồn Rust chứa nhiều đoạn xử lý đặc thù cho từng OS (Windows: `CREATE_NO_WINDOW`, Linux: xử lý Wayland/Nvidia, macOS: xử lý Path), nhưng Frontend hoàn toàn không biết về sự khác biệt này nhờ lớp abstraction của Tauri.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

Dự án áp dụng nhiều kỹ thuật lập trình nâng cao để tối ưu UX:

*   **Log Streaming & Buffering:** Để tránh làm tràn bộ nhớ và treo UI khi Scrcpy in ra hàng nghìn dòng log, Backend Rust sử dụng `BufReader` kết hợp với một khoảng thời gian chờ (interval 100ms) để gom các dòng log lại trước khi gửi về Frontend một lần.
*   **Binary Lifecycle Management:** Kỹ thuật tự động hóa cài đặt. Ứng dụng có khả năng tự phát hiện nền tảng (OS/Arch), gọi API GitHub để tìm bản Scrcpy mới nhất, tải xuống, giải nén và cấu hình đường dẫn môi trường (Environment Variable) cho ADB ngay trong runtime.
*   **HID Hardware Simulation (OTG Mode):** Chuyển đổi các sự kiện phím/chuột từ máy tính thành các gói tin chuẩn HID thông qua tham số `--keyboard=uhid` và `--mouse=uhid`. Đây là kỹ thuật giúp Scrcpy đạt độ trễ gần bằng 0 và hỗ trợ các ký tự đặc biệt (như tiếng Việt, tiếng Ba Lan).
*   **Facade Pattern (Frontend):** Hook `useScrcpy.ts` đóng vai trò là một lớp Facade, che giấu sự phức tạp của các lời gọi lệnh `invoke` đến Rust, cung cấp một giao diện lập trình đơn giản cho các component UI.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng hoạt động được chia làm 3 giai đoạn chính:

#### Giai đoạn 1: Khởi tạo (Bootstrap)
1.  **Rust Backend** khởi động, tạo `splashscreen` để che giấu thời gian load Webview.
2.  Kiểm tra sự tồn tại của các tệp thực thi `scrcpy` và `adb`.
3.  Nếu thiếu, ứng dụng kích hoạt **Onboarding Flow** dẫn dắt người dùng tải binary.
4.  Khi sẵn sàng, `main_window` hiện lên và đóng `splashscreen`.

#### Giai đoạn 2: Kết nối & Cấu hình (Discovery)
1.  Frontend gọi `get_devices` định kỳ. Rust thực thi lệnh `adb devices`, parse kết quả trả về mảng String.
2.  Người dùng cấu hình thông số (Bitrate, FPS, Resolution) trên UI. Các thông số này được lưu vào `localStorage` và đồng bộ với một `ScrcpyConfig` object trong React.
3.  Kết nối không dây: UI thu thập IP/Code -> Rust thực hiện `adb pair` và `adb connect` với cơ chế timeout 5 giây để tránh treo.

#### Giai đoạn 3: Thực thi Session (Execution)
1.  Người dùng nhấn "Start Mission".
2.  Frontend gửi `ScrcpyConfig` cho Rust.
3.  **Rust:** Xây dựng danh sách tham số CLI -> Thiết lập biến môi trường (Environment) -> `spawn` tiến trình con (child process).
4.  **Monitoring:** Rust mở 2 thread riêng để đọc `stdout` và `stderr` của tiến trình đó, liên tục bắn sự kiện `scrcpy-log` về giao diện.
5.  Khi tiến trình kết thúc (do người dùng tắt hoặc lỗi), Rust bắt được tín hiệu thoát và báo về Frontend để cập nhật trạng thái "Ready".

---

### Tổng kết
**ScrcpyGUI v3** là một ví dụ điển hình về cách sử dụng **Tauri** để hiện đại hóa các công cụ dòng lệnh (CLI tools). Sự kết hợp giữa khả năng quản lý tiến trình chặt chẽ của **Rust** và giao diện phản hồi nhanh của **React 19** tạo ra một công cụ chuyên nghiệp, vượt xa các phiên bản GUI cũ chạy trên Java hoặc Python về cả thẩm mỹ lẫn hiệu năng.