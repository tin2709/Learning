Dựa trên mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về dự án **dive** dưới các góc độ công nghệ, kiến trúc và kỹ thuật lập trình.

---

### 1. Công nghệ cốt lõi (Core Technologies)
`dive` được xây dựng chủ yếu bằng ngôn ngữ **Go (Golang)**, tận dụng sức mạnh của hệ sinh thái Go trong việc xử lý hệ thống và container:

*   **TUI (Terminal User Interface):** Sử dụng `awesome-gocui` làm framework chính để xây dựng giao diện đồ họa trong terminal. Kết hợp với `lipgloss` và `termenv` (từ team Charmbracelet) để định nghĩa style, màu sắc và layout.
*   **Container SDK:** Tương tác trực tiếp với Docker qua `docker/docker` (Docker Engine API) và `docker/cli`. Hỗ trợ cả Podman thông qua việc gọi lệnh CLI.
*   **Xử lý tệp tin:** Sử dụng các gói chuẩn của Go như `archive/tar` và `compress/gzip` để giải nén và phân tích cấu trúc các layer của Docker image (thực chất là các file tar).
*   **Hiệu năng:** Sử dụng `cespare/xxhash/v2` để băm (hash) nội dung file nhanh chóng nhằm so sánh sự thay đổi giữa các layer.
*   **CLI Framework:** Sử dụng `cobra` - tiêu chuẩn công nghiệp cho Go CLI để quản lý lệnh và tham số.
*   **Quản lý sự kiện:** Sử dụng `go-partybus` để truyền tin giữa các thành phần (ví dụ: tiến trình phân tích báo cáo trạng thái cho giao diện người dùng).

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của `dive` đi theo hướng **Component-based** và **Layered Architecture**, tách biệt rõ ràng giữa logic xử lý dữ liệu và giao diện người dùng:

*   **Tách biệt Logic Phân tích (Analysis) và Hiển thị (UI):**
    *   Phần `dive/image` và `dive/filetree` chịu trách nhiệm đọc dữ liệu thô, xây dựng cấu trúc cây thư mục và tính toán độ hiệu quả (Efficiency).
    *   Phần `cmd/dive/cli/internal/ui` chỉ tập trung vào việc render dữ liệu lên màn hình terminal.
*   **Cơ chế Resolver (Mẫu thiết kế Strategy):**
    *   Dự án sử dụng giao diện `Resolver` (`dive/image/resolver.go`) để trừu tượng hóa nguồn hình ảnh. Điều này cho phép tool hỗ trợ đa dạng nguồn: Docker Engine, tệp Tar (Docker Archive), hoặc Podman mà không làm thay đổi logic phân tích lõi.
*   **Kiến trúc hướng sự kiện (Event-driven):**
    *   Việc sử dụng `partybus` giúp các thành phần giao tiếp "lỏng lẻo" (loosely coupled). Khi một quá trình phân tích hoàn tất, nó bắn một sự kiện, và UI lắng nghe để cập nhật thay vì gọi trực tiếp lẫn nhau.
*   **Mô hình ViewModel:**
    *   Trong thư mục `viewmodel`, dự án duy trì trạng thái của giao diện (ví dụ: cây đang mở đến đâu, layer nào đang được chọn). Đây là cầu nối giữa cấu trúc cây file phức tạp và buffer hiển thị của terminal.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)
Mã nguồn của `dive` thể hiện các kỹ thuật lập trình Go nâng cao và tối ưu:

*   **Xây dựng cấu trúc cây (Tree Data Structure):**
    *   `FileNode` và `FileTree` được thiết kế để đại diện cho hệ thống tệp tin của container. Kỹ thuật này cho phép thực hiện các phép so sánh (diff) hiệu quả bằng cách duyệt cây (DFS - Depth First Search).
*   **Tối ưu bộ nhớ với Stacked Trees:**
    *   Khi người dùng chọn xem "Aggregated Changes" (thay đổi tích lũy), tool không copy toàn bộ dữ liệu mà thực hiện "chồng" các layer lên nhau thông qua logic so sánh các node, giúp tiết kiệm bộ nhớ khi xử lý image lớn.
*   **Sử dụng Reflection (Phản chiếu):**
    *   Trong `ui_keybindings.go`, tác giả sử dụng gói `reflect` để tự động ánh xạ cấu hình từ file YAML vào các cấu trúc struct nội bộ, giúp việc tùy biến phím tắt rất linh hoạt.
*   **Concurrent Caching (Bộ nhớ đệm đồng thời):**
    *   `Comparer` (`comparer.go`) sử dụng bộ nhớ đệm để lưu trữ các cây file đã được tính toán. Khi người dùng chuyển đổi qua lại giữa các layer, kết quả hiển thị gần như tức thì.
*   **Build Tags & Conditional Compilation:**
    *   Sử dụng `//go:build` để xử lý các đoạn mã đặc thù cho OS (ví dụ: Docker socket trên Windows vs Unix, hoặc hỗ trợ Podman chỉ trên Linux/Darwin).

### 4. Luồng hoạt động hệ thống (System Workflow)
Quy trình xử lý của `dive` diễn ra theo các bước sau:

1.  **Tiếp nhận đầu vào (Input):** Người dùng nhập tag image hoặc file tar. `Cobra` phân giải tham số và xác định `Source` (Docker/Podman/Archive).
2.  **Trích xuất Layer (Fetching):** `Resolver` tương ứng sẽ lấy image về. Nếu là Docker Engine, nó gọi API `ImageSave` để lấy một luồng stream dữ liệu tar.
3.  **Phân tích cấu trúc (Analysis):**
    *   Hệ thống đọc từng layer (file tar).
    *   Xây dựng cây file cho từng layer (`RefTrees`).
    *   Phát hiện "Whiteouts" (các file đặc biệt đánh dấu việc xóa file trong Docker).
4.  **Tính toán Diff & Efficiency:**
    *   So sánh các layer kế tiếp nhau để tìm file Added, Modified, hoặc Removed.
    *   Tính toán "Wasted space" dựa trên các file bị ghi đè hoặc bị xóa ở các layer sau nhưng vẫn tồn tại trong layer trước.
5.  **Khởi tạo UI & Tương tác:**
    *   Giao diện `gocui` được khởi động.
    *   Màn hình chia làm 2 pane chính: Bên trái là danh sách layer, bên phải là cây file tương ứng với trạng thái image tại layer đó.
    *   Mỗi phím bấm của người dùng kích hoạt một sự kiện trong `controller.go` để cập nhật `ViewModel` và render lại màn hình.

### Tổng kết
`dive` là một ví dụ xuất sắc về việc kết hợp giữa **phân tích dữ liệu hệ thống thấp cấp** (low-level system data) và **giao diện terminal hiện đại**. Sự thành công của nó đến từ việc trừu tượng hóa tốt các nguồn dữ liệu (Docker/Podman) và tối ưu hóa cấu trúc dữ liệu cây để xử lý hàng trăm nghìn file trong container mà vẫn đảm bảo độ mượt mà cho UI.