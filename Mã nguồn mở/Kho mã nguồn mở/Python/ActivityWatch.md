Dựa trên các tệp mã nguồn và tài liệu bạn cung cấp cho dự án **ActivityWatch**, dưới đây là phân tích chuyên sâu về hệ thống này:

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

ActivityWatch là một dự án đa ngôn ngữ, đang trong quá trình chuyển đổi chiến lược từ Python sang Rust để tối ưu hiệu suất.

*   **Ngôn ngữ lập trình:**
    *   **Python (Chủ đạo hiện tại):** Dùng cho `aw-server` (Flask/REST-X), các watchers (`aw-watcher-window`, `aw-watcher-afk`) và logic xử lý dữ liệu.
    *   **Rust (Tương lai):** Đang dần thay thế lõi hệ thống với `aw-server-rust`, `aw-notify-rs` và `aw-sync`. Rust được chọn để giảm mức chiếm dụng RAM và cải thiện tốc độ xử lý truy vấn.
    *   **JavaScript/TypeScript:** Sử dụng cho `aw-webui` (Frontend) và các thư viện client.
*   **Giao diện & UI:**
    *   **Web-based UI:** Dashboard hiển thị qua trình duyệt hoặc cửa sổ nhúng.
    *   **Tauri & Qt:** `aw-qt` là wrapper cũ, trong khi dự án đang chuyển sang `aw-tauri` để tạo ứng dụng desktop hiện đại, nhẹ hơn.
*   **Hạ tầng dữ liệu:**
    *   Dữ liệu được lưu trữ cục bộ (SQLite hoặc các file dạng timeseries).
    *   **REST API:** Cung cấp các endpoint để Watchers đẩy dữ liệu (Heartbeats) và Frontend truy vấn dữ liệu (Queries).
*   **Công cụ đóng gói (Packaging):** Sử dụng `PyInstaller` (cho Python), `Makefile` phức tạp để điều phối các submodules, và `Inno Setup` cho Windows.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của ActivityWatch dựa trên nguyên tắc **Decoupling (Tách biệt hoàn toàn)**:

*   **Mô hình Client-Server:** Hệ thống không phải là một khối thống nhất. Nó bao gồm một Server trung tâm và nhiều Watchers (Clients). Watchers không cần biết Server lưu trữ dữ liệu thế nào, chúng chỉ cần gửi JSON qua HTTP.
*   **Cấu trúc Submodule (Meta-package):** Repo chính này không chứa mã nguồn lõi mà đóng vai trò "người điều phối". Nó quản lý các module độc lập (`aw-core`, `aw-client`, `aw-server`,...) thông qua `git submodule`. Tư duy này giúp mỗi thành phần có thể được phát triển và kiểm thử riêng biệt.
*   **Mô hình dữ liệu "Buckets & Events":** Thay vì lưu log thô, dữ liệu được tổ chức thành các *Bucket* (thùng chứa). Mỗi Bucket chứa các *Event* có mốc thời gian và thời lượng.
*   **Cơ chế Heartbeat (Nhịp đập):** Thay vì ghi dữ liệu mỗi giây, các Watchers gửi nhịp đập định kỳ. Nếu sự kiện mới giống sự kiện cũ, Server chỉ cập nhật thời gian kết thúc của sự kiện hiện tại, giúp tiết kiệm bộ nhớ cực lớn.

### 3. Kỹ thuật Lập trình Đặc sắc (Coding Patterns)

*   **Inside-out Signing (trong file `build_app_tauri.sh`):** Kỹ thuật ký số (codesign) cho macOS cực kỳ phức tạp. Mã nguồn cho thấy cách họ xử lý các " Mach-O binary leaves" trước, sau đó mới đến các framework và cuối cùng là bundle app. Điều này giải quyết lỗi "bundle format is ambiguous" thường gặp khi đóng gói Python trong app macOS.
*   **Cross-platform Compatibility Layers:** Trong `aw.spec`, dự án sử dụng các logic điều kiện (`if platform.system() == "Darwin"`) để tùy biến binaries và icon cho từng HĐH. Đặc biệt là việc xử lý các `hiddenimports` cho thư viện đồ họa X11 trên Linux hoặc Win32 trên Windows.
*   **Query Scripting Language:** Hệ thống có một ngôn ngữ truy vấn riêng (Query API) cho phép thực hiện các phép tính phức tạp (như "Lấy thời gian dùng trình duyệt trừ đi thời gian AFK") ngay tại phía Server thay vì tải toàn bộ dữ liệu về Client.
*   **Xử lý lỗi thực thi (trong `integration_tests.py`):** Sử dụng `ctypes` để can thiệp sâu vào hệ thống Windows nhằm kill process một cách triệt để (`TerminateProcess`) khi các phương thức thông thường của Python thất bại.

### 4. Luồng Hoạt động Hệ thống (System Flow)

1.  **Giai đoạn Thu thập (Collection):**
    *   `aw-watcher-window` liên tục hook vào API của HĐH để lấy tiêu đề cửa sổ đang hoạt động.
    *   `aw-watcher-afk` giám sát sự kiện chuột/bàn phím.
    *   Cứ sau mỗi $n$ giây, chúng gửi một **Heartbeat** chứa metadata (tên app, URL, title) về Server.

2.  **Giai đoạn Xử lý tại Server (Ingestion):**
    *   Server nhận Heartbeat, kiểm tra `pulsetime` (ngưỡng thời gian). Nếu sự kiện mới nằm trong ngưỡng và nội dung giống sự kiện cũ, nó sẽ "merge" vào làm một.
    *   Dữ liệu được ghi vào Datastore cục bộ của người dùng.

3.  **Giai đoạn Truy vấn & Hiển thị (Visualization):**
    *   Người dùng mở Web UI.
    *   Web UI gửi các Query (ví dụ: `sum_durations`) đến Server.
    *   Server tính toán và trả về kết quả JSON để vẽ biểu đồ Timeline hoặc Pie chart.

4.  **Giai đoạn Đồng bộ (Synchronization - WIP):**
    *   `aw-sync` theo dõi thư mục dữ liệu và sử dụng các dịch vụ như Syncthing hoặc Dropbox để đồng bộ hóa các file bucket giữa các thiết bị mà không cần máy chủ trung gian (Decentralized).

### Tổng kết
ActivityWatch là một ví dụ điển hình về việc xây dựng một hệ thống **Quantified Self** (Định lượng bản thân) đặt quyền riêng tư lên hàng đầu. Tư duy kiến trúc của họ rất hiện đại trong việc tách biệt các nguồn dữ liệu (Watchers) và xử lý dữ liệu (Server), cho phép cộng đồng dễ dàng mở rộng bằng cách viết thêm Watchers cho bất kỳ ứng dụng nào.