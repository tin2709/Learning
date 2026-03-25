Dưới đây là phân tích chi tiết về dự án **pdf-cli** dựa trên mã nguồn Go:

### 1. Công nghệ cốt lõi (Core Technologies)
Dự án tận dụng các thư viện mạnh mẽ để xử lý đồ họa và văn bản trong môi trường Terminal:
*   **MuPDF (via `go-fitz`):** Đây là "trái tim" của hệ thống, chịu trách nhiệm phân tích (parsing) và kết xuất (rendering) các định dạng PDF, EPUB, DOCX. MuPDF nổi tiếng với tốc độ và khả năng hỗ trợ LaTeX cực tốt.
*   **Terminal Graphics Protocols:** Sử dụng `go-termimg` để hỗ trợ các giao thức đồ họa hiện đại như **Kitty**, **Sixel**, và **iTerm2**. Điều này cho phép hiển thị hình ảnh độ phân giải cao trực tiếp trong terminal thay vì dùng ASCII art.
*   **Fuzzy Searching:** Thư viện `fuzzy` giúp tìm kiếm file nhanh chóng bằng cách tính toán điểm số tương đồng, cho phép người dùng gõ sai lệch ít mà vẫn tìm thấy tài liệu.
*   **Cgo Integration:** Dự án sử dụng Cgo để gọi trực tiếp các hàm cấp thấp của MuPDF (`fz_layout_document`, `fz_resolve_link`) nhằm xử lý layout cho các tài liệu có khả năng tự dàn trang (reflowable) như EPUB và HTML.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của `pdf-cli` được chia theo mô hình **Modular CLI**:
*   **Phân tách logic thiết bị (Terminal Abstraction):** Gói `internal/terminal` đóng vai trò là một lớp trung gian. Nó không chỉ lấy kích thước ký tự mà còn cố gắng lấy kích thước điểm ảnh (pixel) chính xác thông qua các mã điều khiển (escape sequences) đặc thù của từng loại terminal (như Kitty).
*   **Quản lý trạng thái bền vững (State Persistence):** Sử dụng mã băm MD5 của đường dẫn tệp để tạo ID duy nhất cho mỗi tài liệu (gói `internal/config`). Điều này giúp ứng dụng ghi nhớ chế độ hiển thị, mức độ thu phóng (zoom) và chế độ Dark Mode riêng biệt cho từng file.
*   **Vòng lặp sự kiện (Event-driven Loop):** Ứng dụng chạy trong một vòng lặp vô hạn, lắng nghe đồng thời từ nhiều nguồn: phím bấm từ người dùng, sự thay đổi tệp tin trên đĩa (cho LaTeX workflow), và thậm chí là từ một file FIFO (để nhận lệnh từ bên ngoài).

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)
*   **Xử lý hình ảnh thích ứng (Adaptive Rendering):** 
    *   Tính toán **DPI động** dựa trên kích thước pixel của cell terminal và hệ số scale để đảm bảo hình ảnh hiển thị sắc nét nhất.
    *   Kỹ thuật **Smart Invert (HSL Invert):** Trong `imgutil/imgutil.go`, thay vì đảo ngược màu RGB thô (làm biến đổi màu sắc), ứng dụng chuyển sang không gian màu HSL, chỉ đảo ngược độ sáng (Lightness) và giữ nguyên màu sắc (Hue/Saturation).
*   **Nhận diện nội dung thông minh:** Hàm `getPageContentType` tự động đếm số lượng từ và kiểm tra dữ liệu hình ảnh để quyết định xem nên hiển thị trang đó dưới dạng văn bản thuần (để copy/search dễ dàng) hay dạng ảnh (để giữ nguyên công thức toán học/sơ đồ).
*   **Atomic Display (Kitty Synchronized Updates):** Sử dụng mã điều khiển `\033[?2026h` để thông báo cho terminal bắt đầu và kết thúc một lần cập nhật giao diện lớn. Kỹ thuật này ngăn chặn hiện tượng màn hình bị nhấp nháy (flickering) khi render trang mới.
*   **Cơ chế Auto-reload ổn định:** Khi phát hiện file thay đổi (LaTeX re-compile), ứng dụng không load ngay mà đợi file "ổn định" (kích thước không đổi sau một khoảng thời gian ngắn) để tránh lỗi đọc file đang ghi dở.

### 4. Luồng hoạt động hệ thống (System Workflow)
1.  **Khởi tạo:** Kiểm tra đối số dòng lệnh. Nếu là thư mục, kích hoạt `FilePicker`.
2.  **Chọn file:** `FilePicker` quét danh mục tệp hỗ trợ, sử dụng tìm kiếm mờ (fuzzy) để người dùng chọn.
3.  **Tải tài liệu:** `DocumentViewer` mở file qua `go-fitz`, đọc mục lục (ToC) và khởi tạo cấu hình đã lưu trước đó.
4.  **Phát hiện Terminal:** Hệ thống gửi các mã truy vấn ẩn tới terminal để xác định khả năng hiển thị đồ họa.
5.  **Chu kỳ Hiển thị:**
    *   Trích xuất dữ liệu trang (Text hoặc Image).
    *   Áp dụng bộ lọc màu (Dark Mode/Invert).
    *   Tính toán layout (Reflow text hoặc Scale image).
    *   Đẩy luồng byte đồ họa xuống terminal.
6.  **Tương tác:** Lắng nghe phím bấm để chuyển trang, tìm kiếm văn bản trong trang hiện tại hoặc thay đổi chế độ hiển thị (Dual page/Half page).

### Tổng kết
`pdf-cli` không chỉ đơn thuần là một trình đọc PDF; nó là một công cụ tối ưu cho lập trình viên và người dùng Linux chuyên sâu. Việc kết hợp giữa khả năng xử lý tài liệu mạnh mẽ của C-library (MuPDF) và tính linh hoạt của Go tạo ra một ứng dụng cực nhẹ, hỗ trợ tốt các workflow phức tạp như biên soạn LaTeX ngay trong môi trường terminal.