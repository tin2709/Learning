Dựa trên mã nguồn của dự án **Sparkle**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc, kỹ thuật và luồng hoạt động của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Framework chính:** **Electron** (với `electron-vite`), cho phép xây dựng ứng dụng desktop bằng công nghệ web nhưng có khả năng can thiệp sâu vào hệ thống Windows.
*   **Frontend:** **React 19** kết hợp với **TypeScript**. Giao diện được xây dựng bằng **Tailwind CSS 4** và thư viện icon **Lucide**.
*   **Quản lý trạng thái:** **Zustand** được sử dụng để quản lý state ở phía Renderer (như trạng thái cài đặt app, thông tin hệ thống).
*   **Ngôn ngữ thực thi hệ thống:** **PowerShell**. Hầu hết các logic tối ưu hóa (Tweaks) thực tế được viết bằng script `.ps1`.
*   **Lưu trữ cấu hình:** **electron-store** để lưu các tùy chỉnh của người dùng và trạng thái của các tweak đã áp dụng.
*   **Thu thập thông tin phần cứng:** **systeminformation** để lấy dữ liệu CPU, RAM, GPU hiển thị trên dashboard.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án Sparkle được thiết kế theo mô hình **Data-Driven Execution** (Thực thi dựa trên dữ liệu):

*   **Tách biệt Logic và UI:** Toàn bộ các tweak (tinh chỉnh) không được code cứng vào giao diện. Thay vào đó, mỗi tweak là một module độc lập trong thư mục `tweaks/`, bao gồm file mô tả (`meta.json`) và file thực thi (`apply.ps1`, `unapply.ps1`).
*   **Mô hình Main - Preload - Renderer:** Tuân thủ chặt chẽ kiến trúc bảo mật của Electron. Main process xử lý các tác vụ đặc quyền (chạy PowerShell), Renderer xử lý giao diện, và Preload đóng vai trò cầu nối bảo mật (IPC bridge).
*   **Cơ chế Registry tập trung:** Dự án có một script `build.js` làm nhiệm vụ quét toàn bộ thư mục tweak để tạo ra file `registry.json`. File này đóng vai trò là "mục lục" cho ứng dụng, giúp việc thêm tweak mới chỉ đơn giản là thêm thư mục mà không cần sửa code React.
*   **Đặc quyền Quản trị (Elevation):** Ứng dụng được cấu hình trong `package.json` với `requestedExecutionLevel: requireAdministrator`, đảm bảo mọi script PowerShell đều có quyền can thiệp vào Registry và tệp tin hệ thống.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **PowerShell Bridge:** Kỹ thuật quan trọng nhất là việc tạo ra một lớp trung gian (`powershell.ts`) để gọi các lệnh PowerShell từ Node.js. Nó xử lý việc truyền tham số, bắt lỗi (stderr) và trả kết quả về cho UI.
*   **Dynamic Component Rendering:** React render danh sách tweak một cách năng động dựa trên file `registry.json`. Kỹ thuật này giúp ứng dụng cực kỳ linh hoạt và dễ mở rộng.
*   **Reversible Logic:** Hầu hết các tweak đều được thiết kế có cặp `apply` và `unapply`. Trạng thái "Bật/Tắt" được đồng bộ giữa Registry của Windows và `electron-store`.
*   **IPC Modularization:** Các trình xử lý IPC (Inter-Process Communication) được chia nhỏ thành các file chuyên biệt (`dnsHandler.ts`, `tweakHandler.ts`, `backup.ts`) thay vì dồn tất cả vào file `index.ts` chính.
*   **Build-time Automation:** Sử dụng script Node.js (`build.js`) để tự động hóa việc tính toán mã băm SHA-256 cho các bản build và cập nhật metadata cho các tweak.

### 4. Luồng hoạt động hệ thống (System Workflow)

#### A. Luồng khởi tạo:
1.  **Main Process** khởi động -> Kiểm tra quyền Admin -> Khởi tạo khay hệ thống (Tray).
2.  **Renderer** tải giao diện -> Gọi qua **Preload** để yêu cầu thông tin hệ thống từ Main (CPU, RAM qua `systeminformation`).
3.  Ứng dụng đọc danh sách tweak từ `registry.json` và trạng thái đã lưu trong `Store`.

#### B. Luồng áp dụng Tweak:
1.  Người dùng nhấn Toggle trên UI.
2.  **Renderer** gửi tín hiệu IPC `apply-tweak` kèm theo ID của tweak.
3.  **Main Process** tìm đường dẫn file `apply.ps1` tương ứng.
4.  Main gọi hàm thực thi PowerShell với quyền admin.
5.  PowerShell can thiệp vào Windows Registry hoặc thực hiện các lệnh hệ thống.
6.  Kết quả (Thành công/Lỗi) được gửi ngược lại UI để hiển thị Toast thông báo.

#### C. Luồng quản lý ứng dụng (Apps Page):
1.  Người dùng chọn các ứng dụng từ danh sách (dữ liệu lấy từ `apps.json`).
2.  Renderer gửi danh sách ID cho Main.
3.  Main thực thi lệnh `winget` hoặc `choco` thông qua PowerShell để cài đặt/gỡ bỏ hàng loạt (Bulk actions).

#### D. Luồng cập nhật:
1.  `electron-updater` kiểm tra version trên GitHub Releases.
2.  Nếu có bản mới, nó tải ngầm và thông báo cho người dùng qua UI.
3.  Người dùng xác nhận -> Ứng dụng tự khởi động lại và cài đặt bản cập nhật.

### Tổng kết
Sparkle là một ứng dụng **Desktop mạnh mẽ** nhờ sự kết hợp giữa sự linh hoạt của React và sức mạnh can thiệp hệ thống của PowerShell. Kiến trúc của nó hướng tới việc **cộng tác cộng đồng**, nơi người dùng có thể đóng góp tweak mới mà không cần kiến thức sâu về lập trình Electron hay React.