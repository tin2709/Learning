Dựa trên kho mã nguồn của **Sniffnet**, dưới đây là phân tích chi tiết về các khía cạnh công nghệ, kiến trúc và luồng hoạt động của ứng dụng:

### 1. Công nghệ cốt lõi (Core Technology Stack)
Sniffnet là một minh chứng điển hình cho sức mạnh của hệ sinh thái **Rust** trong việc xây dựng ứng dụng hệ thống có giao diện người dùng (GUI):
*   **Ngôn ngữ chính:** **Rust (98.8%)** - Tận dụng tính an toàn bộ nhớ và hiệu suất cực cao để xử lý hàng ngàn gói tin mỗi giây mà không gây treo máy.
*   **Giao diện đồ họa (GUI):** **Iced** - Một thư viện GUI đa nền tảng lấy cảm hứng từ kiến trúc Elm, tập trung vào tính an toàn kiểu (type-safety) và mô hình phản ứng (reactive).
*   **Xử lý mạng:**
    *   **libpcap/Npcap:** Thư viện tiêu chuẩn để bắt gói tin ở tầng thấp.
    *   **etherparse:** Thư viện Rust mạnh mẽ để phân tích (parse) các header của giao thức (Ethernet, IP, TCP, UDP, ICMP, ARP).
*   **Trực quan hóa dữ liệu:** **Plotters** và **Splines** - Dùng để vẽ biểu đồ lưu lượng thời gian thực với đường cong mềm mại thông qua nội suy (interpolation).
*   **Cơ sở dữ liệu nhúng:** **MaxMinddb** - Truy vấn offline dữ liệu vị trí địa lý (Country) và số hiệu mạng (ASN) từ file `.mmdb`.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của Sniffnet được thiết kế để giải quyết bài toán: **"Làm sao để vừa bắt gói tin tốc độ cao vừa giữ giao diện mượt mà?"**

*   **Mô hình TEA (The Elm Architecture):** Chia ứng dụng thành 3 phần:
    1.  **Model:** Trạng thái của ứng dụng (dữ liệu traffic, cấu hình filter).
    2.  **Update:** Logic thay đổi trạng thái dựa trên các "Message" (ví dụ: `Tick`, `StartCapture`).
    3.  **View:** Mô tả giao diện dựa trên Model.
*   **Tách biệt Backend/Frontend (Multithreading):**
    *   **Capture Thread:** Chạy ngầm, sử dụng `pcap` để lấy dữ liệu thô từ card mạng.
    *   **Parsing Pipeline:** Chuyển dữ liệu thô thành các struct có ý nghĩa (`AddressPortPair`, `InfoTraffic`).
    *   **GUI Thread:** Chỉ nhận các bản cập nhật đã được tổng hợp thông qua các kênh bất đồng bộ (channels) để hiển thị, tránh việc render quá nhiều gây tốn tài nguyên.
*   **Thiết kế hướng đối tượng & an toàn (Strong Typing):** Sử dụng Enums rất mạnh mẽ để quản lý trạng thái (ví dụ: `RunningPage`, `StyleType`, `Language`) giúp hạn chế tối đa lỗi logic khi runtime.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)
*   **Perfect Hashing (PHF):** Trong `build.rs`, ứng dụng sử dụng kỹ thuật "Perfect Hash Functions" để tạo bản đồ ánh xạ hơn 12,000 cổng dịch vụ (port-to-service) ngay tại thời điểm biên dịch. Điều này giúp việc tra cứu tên dịch vụ (ví dụ: 443 -> HTTPS) đạt tốc độ O(1) mà không tốn tài nguyên khi chạy.
*   **Font Subsetting:** Sử dụng script `fonts.sh` để lọc bỏ các ký tự không dùng đến trong font chữ, giúp giảm kích thước file binary cuối cùng.
*   **Custom Theming hệ thống:** Sử dụng file cấu hình TOML để định nghĩa bảng màu (`Palette`), cho phép người dùng thay đổi toàn bộ giao diện (Catppuccin, Dracula, v.v.) một cách linh hoạt.
*   **Quản lý tài nguyên nhúng:** Sử dụng macro `include_bytes!` để nhúng trực tiếp icon, âm thanh thông báo và dữ liệu MMDB vào trong file thực thi duy nhất, giúp việc phân phối ứng dụng cực kỳ đơn giản (Portable).

### 4. Luồng hoạt động hệ thống (System Operation Flow)

1.  **Khởi tạo (Bootstrap):** `main.rs` nạp cấu hình từ `conf.toml`, thiết lập môi trường (ví dụ: chuyển hướng stderr trên Windows) và khởi chạy vòng lặp sự kiện của Iced.
2.  **Thiết lập (Setup):** Người dùng chọn adapter mạng và bộ lọc (BPF filter). Khi nhấn "Start", một luồng bắt gói tin mới được sinh ra.
3.  **Bắt và Phân tích (Capture & Parse):**
    *   Gói tin thô được `pcap` lấy về.
    *   `parse_packets.rs` bóc tách các lớp: Data Link -> Network -> Transport.
    *   Xác định hướng traffic (In/Out) dựa trên địa chỉ IP nội bộ của adapter.
4.  **Làm giàu dữ liệu (Enrichment):**
    *   Tra cứu vị trí địa lý qua file MMDB.
    *   Tra cứu tiến trình (Process) đang thực hiện kết nối đó dựa trên cổng (OS-specific).
    *   Tra cứu dịch vụ qua bản đồ PHF đã tạo.
5.  **Tổng hợp (Aggregation):** Dữ liệu được gom nhóm theo kết nối (`AddressPortPair`) và lưu vào `InfoTraffic`. Các chỉ số byte/gói tin được đẩy vào hàng đợi của biểu đồ.
6.  **Cập nhật giao diện (UI Sync):** Cứ sau mỗi khoảng thời gian (Tick), backend gửi tín hiệu cho GUI thread. Iced sẽ so sánh sự thay đổi và vẽ lại các thành phần cần thiết (biểu đồ, danh sách kết nối).
7.  **Thông báo (Notification):** Nếu lưu lượng vượt ngưỡng (Threshold) hoặc gặp IP trong danh sách đen (Blacklist), hệ thống kích hoạt logic âm thanh hoặc gửi Webhook.

### Tóm lại:
Sniffnet không chỉ là một công cụ giám sát mạng mà còn là một ví dụ mẫu mực về cách xây dựng **ứng dụng Desktop hiện đại bằng Rust**: Hiệu năng của C++ nhưng an toàn và dễ bảo trì hơn nhờ hệ thống kiểu dữ liệu và trình quản lý gói vượt trội.