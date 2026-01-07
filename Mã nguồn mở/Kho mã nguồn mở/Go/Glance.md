Dưới đây là bản phân tích chi tiết về dự án **Glance** – một dashboard tự lưu trữ (self-hosted) gọn nhẹ và tối ưu, được trình bày dưới dạng file README tiếng Việt.

---

# Phân tích Kỹ thuật Dự án Glance

Dự án **Glance** là một ví dụ điển hình về tư duy tối giản (minimalism) nhưng hiệu quả cao trong việc phát triển phần mềm hiện đại. Dưới đây là cái nhìn sâu sắc về công nghệ và kiến trúc của nó.

## 1. Công nghệ Cốt lõi (Core Technologies)

*   **Backend (Ngôn ngữ chính):** **Go (Golang)**. Lựa chọn này mang lại hiệu năng cao, khả năng xử lý đồng thời (concurrency) tuyệt vời và tạo ra một file thực thi duy nhất (single binary) cực kỳ nhỏ gọn (< 20MB).
*   **Frontend (Giao diện):** 
    *   **Vanilla JS:** Không sử dụng các framework nặng nề như React, Vue hay Angular. Dự án dùng JavaScript thuần để tối ưu tốc độ tải trang.
    *   **CSS thuần:** Sử dụng các kỹ thuật như CSS Variables và CSS Container Queries để xử lý giao diện đáp ứng (responsive).
*   **Templating:** **Go `html/template`**. Toàn bộ giao diện được render từ phía server (Server-Side Rendering - SSR), giúp trang web hiển thị gần như ngay lập tức khi tải.
*   **Định cấu hình:** **YAML**. Sử dụng thư viện `gopkg.in/yaml.v3` để quản lý toàn bộ thiết lập hệ thống và giao diện.
*   **Dữ liệu & API:** 
    *   `gofeed`: Để xử lý các luồng RSS/Atom.
    *   `gopsutil`: Để thu thập thông tin hệ thống (CPU, RAM, Disk).
    *   `fsnotify`: Theo dõi thay đổi file cấu hình để tự động nạp lại (Hot Reload).

## 2. Kỹ thuật và Tư duy Kiến trúc (Architecture & Design Thinking)

### Kiến trúc hướng Widget (Widget-based Architecture)
Glance được thiết kế xung quanh một Interface chung là `widget`. Mỗi tính năng (Weather, Reddit, RSS, Monitor...) đều thực hiện (implement) các phương thức chung:
*   `initialize()`: Khởi tạo các giá trị mặc định.
*   `update()`: Lấy dữ liệu từ nguồn bên ngoài (API, RSS, System).
*   `Render()`: Trả về mã HTML sau khi đã đổ dữ liệu vào template.

Tư duy này giúp dự án cực kỳ dễ mở rộng (Extensible). Muốn thêm một widget mới, lập trình viên chỉ cần tạo một cấu trúc dữ liệu mới tuân thủ interface này.

### Xử lý đồng thời (Concurrency)
Tận dụng tối đa sức mạnh của Go Routines. Khi người dùng tải trang, thay vì đợi từng widget lấy dữ liệu xong mới hiện trang, Glance khởi chạy các widget song song thông qua `sync.WaitGroup` và `Worker Pool`. Điều này giúp giảm tổng thời gian chờ xuống bằng thời gian của widget chậm nhất.

### Tư duy "Không phụ thuộc" (Zero-Dependency Mindset)
Glance tránh xa việc sử dụng `package.json` hay các trình quản lý gói frontend phức tạp. Toàn bộ tài nguyên tĩnh (Static assets) như CSS, JS được nhúng trực tiếp vào file nhị phân Go (sử dụng `embed` package). Kết quả là một công cụ "chạy ngay không cần cài đặt thêm".

## 3. Các kỹ thuật chính nổi bật (Key Technical Highlights)

1.  **Dynamic Configuration Substitutions:** Cho phép sử dụng biến môi trường trực tiếp trong file YAML (ví dụ: `${API_KEY}`). Điều này rất hữu ích cho bảo mật khi triển khai qua Docker.
2.  **Custom API Widget:** Một tính năng cực kỳ mạnh mẽ cho phép người dùng tự định nghĩa cách lấy dữ liệu JSON từ bất kỳ đâu và sử dụng cú pháp template của Go ngay trong file cấu hình để hiển thị dữ liệu đó.
3.  **Intelligent Caching:** Cơ chế cache được thiết lập riêng cho từng widget (theo giờ, theo khoảng thời gian hoặc vô hạn) để tránh bị khóa API (Rate limiting) và tăng tốc độ phản hồi.
4.  **HSL-based Theming:** Hệ thống theme sử dụng không gian màu HSL và các công thức toán học để tự động tính toán độ tương phản và màu sắc văn bản dựa trên màu nền, giúp người dùng tùy biến giao diện chỉ với vài thông số đơn giản.
5.  **Extension qua HTTP:** Một cơ chế extension độc đáo nơi Glance có thể fetch nội dung HTML từ một server khác thông qua các Header đặc biệt, cho phép mở rộng tính năng bằng bất kỳ ngôn ngữ lập trình nào.

## 4. Tóm tắt luồng hoạt động (Operational Workflow)

1.  **Khởi động:**
    *   Chương trình đọc file `glance.yml`.
    *   Xử lý các biến môi trường và gộp các file bao gồm (`!include`).
    *   Khởi tạo bộ theo dõi file (`watcher`) để sẵn sàng cập nhật khi cấu hình thay đổi.
2.  **Xử lý yêu cầu (Request Handling):**
    *   Khi người dùng truy cập trang chủ, Server render bộ khung (skeleton) HTML.
    *   Giao diện gọi API nội bộ `/api/pages/.../content`.
3.  **Cập nhật dữ liệu:**
    *   Hệ thống kiểm tra xem dữ liệu widget nào đã hết hạn cache.
    *   Kích hoạt các Go Routines để fetch dữ liệu mới từ Reddit, Twitch, YouTube, Weather API... một cách song song.
    *   Dữ liệu được đổ vào các file `.html` template tương ứng.
4.  **Hiển thị:**
    *   Toàn bộ HTML của các widget được gửi về trình duyệt.
    *   JavaScript phía frontend thực hiện "Hydration" (kích hoạt các hiệu ứng động như Clock, biểu đồ Stock, Masonry layout cho các widget có kích thước khác nhau).

---

### Kết luận
**Glance** không chỉ là một dashboard, nó là một bài học về việc xây dựng phần mềm **"Lean & Mean"**: Sử dụng đúng công cụ (Go), đúng kiến trúc (Interface/Concurrency) và loại bỏ những thứ không cần thiết (Frontend Frameworks) để đạt được hiệu suất tối đa.