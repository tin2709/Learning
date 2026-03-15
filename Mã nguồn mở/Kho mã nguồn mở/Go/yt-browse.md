Dưới đây là phân tích chi tiết về dự án **yt-browse**, một công cụ giao diện dòng lệnh (TUI) mạnh mẽ để duyệt và tìm kiếm nội dung kênh YouTube.

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án được xây dựng dựa trên hệ sinh thái hiện đại của ngôn ngữ Go dành cho các ứng dụng terminal:

*   **Ngôn ngữ lập trình:** **Go (Golang)**, tận dụng khả năng biên dịch thành file nhị phân duy nhất, hiệu suất cao và xử lý bất đồng bộ (concurrency) tốt.
*   **TUI Framework (The Charm Stack):** 
    *   **Bubble Tea:** Framework quản lý trạng thái theo mô hình MVU (Model-View-Update), giúp xây dựng ứng dụng terminal có tính tương tác cao.
    *   **Lip Gloss:** Thư viện dùng để định nghĩa style, layout, màu sắc và border cho giao diện terminal.
    *   **Bubbles:** Các thành phần UI có sẵn như `list` (danh sách), `textinput` (ô nhập liệu), và `viewport`.
*   **YouTube Data API v3:** Sử dụng thư viện client chính thức của Google để tương tác với dữ liệu YouTube.
*   **Fuzzy Search:** Thư viện `sahilm/fuzzy` để triển khai tính năng tìm kiếm mờ (fuzzy matching).
*   **Dữ liệu & Caching:** Lưu trữ dưới dạng JSON, hỗ trợ nén bằng `gzip` để tối ưu dung lượng ổ đĩa.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của `yt-browse` tuân thủ nghiêm ngặt mô hình **Model-View-Update (Elm Architecture)**:

*   **Tính module hóa:**
    *   `internal/youtube`: Tách biệt hoàn toàn logic nghiệp vụ (gọi API, xử lý dữ liệu thô) khỏi giao diện.
    *   `internal/cache`: Lớp trừu tượng cho việc lưu trữ tạm thời, giúp ứng dụng hoạt động mượt mà và tiết kiệm Quota API.
    *   `internal/tui`: Chứa logic điều khiển giao diện.
*   **Quản lý trạng thái đa tầng:** Ứng dụng sử dụng 3 thực thể `list.Model` riêng biệt cho "Danh sách phát", "Video" và "Video trong danh sách phát". Tư duy này giúp giữ nguyên vị trí cuộn (scroll position) và lựa chọn của người dùng khi chuyển đổi giữa các tab.
*   **Xử lý bất đồng bộ:** Việc tải dữ liệu (vốn chậm do API YouTube phân trang) được đẩy vào các `Cmd` chạy ngầm. Khi dữ liệu về, nó sẽ gửi `Msg` để cập nhật Model mà không làm treo giao diện.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Pipelined detail fetching (Kỹ thuật tải chi tiết theo luồng):** Đây là kỹ thuật thông minh nhất của dự án. API YouTube yêu cầu 2 bước: lấy ID video và lấy thông tin chi tiết (lượt xem, thời lượng). `yt-browse` thực hiện lấy ID trang 1, ngay lập tức đẩy vào một goroutine để lấy chi tiết trang đó trong khi tiếp tục lấy ID trang 2.
*   **Heuristic cho YouTube Shorts:** Vì API không có flag riêng cho Shorts, tác giả sử dụng kỹ thuật suy luận dựa trên thời lượng (`duration <= 60s`) để lọc nội dung.
*   **Hệ thống lọc tùy chỉnh (Custom Filtering):** Thay vì dùng bộ lọc có sẵn của thư viện `list`, dự án tự xây dựng logic lọc hỗ trợ 3 chế độ: **Words** (khớp từ), **Regex** (biểu thức chính quy) và **Fuzzy** (tìm kiếm mờ). 
*   **Date filtering (Lọc theo ngày):** Phân tích cú pháp chuỗi tìm kiếm để trích xuất các từ khóa đặc biệt như `before:` và `after:`, cho phép lọc nội dung theo thời gian thực ngay trong thanh tìm kiếm.
*   **Smart Caching:** Cache có vòng đời (TTL) 24 giờ. Dự án còn có kỹ thuật tự động dọn dẹp các cache hết hạn hoặc vượt quá dung lượng giới hạn (200MB) để bảo vệ bộ nhớ người dùng.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Khởi động & Giải quyết kênh:** Người dùng nhập handle (ví dụ `@3blue1brown`). Ứng dụng kiểm tra cache hoặc gọi API `Channels.List` để lấy ID kênh và thông tin cơ bản.
2.  **Tải danh sách phát (Playlists):** Sau khi xác định kênh, ứng dụng tải danh sách các playlist. Song song đó, một tiến trình ngầm sẽ bắt đầu tải toàn bộ video của kênh để sẵn sàng khi người dùng chuyển tab.
3.  **Duyệt & Tìm kiếm:** 
    *   Người dùng di chuyển bằng phím mũi tên.
    *   Khi nhấn `/`, ô tìm kiếm xuất hiện. Mỗi phím bấm sẽ kích hoạt logic lọc và sắp xếp lại danh sách hiển thị.
4.  **Xem chi tiết:** Khi chọn một mục, `detail.go` sẽ render thông tin chi tiết (mô tả, ngày đăng, URL) ở bảng bên phải.
5.  **Hành động cuối:** Người dùng nhấn `Enter` hoặc `o` để mở trình duyệt, hoặc `y` để sao chép link. Toàn bộ hành trình này diễn ra mà không cần rời khỏi terminal.

### Kết luận
`yt-browse` là một ví dụ điển hình về việc sử dụng **vibe-coding** (lập trình theo cảm hứng nhưng có kiến trúc tốt) để giải quyết một vấn đề thực tế: sự bất tiện trong việc tìm kiếm trên giao diện web của YouTube. Nó tối ưu hóa cực tốt giữa trải nghiệm người dùng terminal và giới hạn kỹ thuật của YouTube API.