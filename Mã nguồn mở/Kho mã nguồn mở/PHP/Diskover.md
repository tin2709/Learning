Dựa trên các tệp mã nguồn và tài liệu bạn cung cấp, dưới đây là phân tích chi tiết về dự án **Diskover Community Edition (CE)**:

---

### 1. Công Nghệ Cốt Lõi (Core Technology Stack)

Hệ thống được xây dựng trên sự kết hợp giữa hiệu suất xử lý hệ thống của Python và khả năng hiển thị web linh hoạt của PHP:

*   **Backend Indexer (Python 3.8+):** Đóng vai trò là "máy quét" (crawler). Python được chọn vì thư viện hỗ trợ hệ thống tệp mạnh mẽ và khả năng tương tác tốt với Elasticsearch.
*   **Search Engine (Elasticsearch 8.x):** Đây là "trái tim" của hệ thống. Thay vì dùng DB truyền thống để lưu thông tin hàng triệu tệp, Diskover dùng Elasticsearch để tận dụng khả năng tìm kiếm full-text, đánh chỉ mục (indexing) tốc độ cao và khả năng phân tích (aggregations) dữ liệu lớn.
*   **Web Interface (PHP 8.4+, JavaScript, D3.js):** 
    *   PHP xử lý logic phía server và giao tiếp với Elasticsearch.
    *   **D3.js & Chart.js:** Dùng để trực quan hóa cấu trúc cây thư mục và biểu đồ dung lượng (disk usage).
*   **Configuration & Auth (SQLite):** Một thay đổi quan trọng trong bản v2.3 là chuyển từ tệp YAML sang lưu trữ cấu hình và thông tin người dùng trong cơ sở dữ liệu SQLite (`diskoverdb.sqlite3`), giúp việc quản lý qua giao diện web dễ dàng hơn.

### 2. Tư Duy Kiến Trúc (Architectural Thinking)

Kiến trúc của Diskover đi theo hướng **Decoupled (Tách rời)** và **Data-Centric (Lấy dữ liệu làm trung tâm)**:

*   **Tách biệt Crawler và UI:** Bộ phận quét dữ liệu (`diskover.py`) hoạt động độc lập với bộ phận hiển thị (`diskover-web`). Điều này cho phép người dùng có thể chạy crawler trên nhiều máy chủ khác nhau nhưng chỉ cần một giao diện quản lý tập trung.
*   **Kiến trúc Plugin-based:** Diskover được thiết kế để mở rộng. Thông qua thư mục `plugins`, người dùng có thể thêm các logic thu thập metadata tùy chỉnh (ví dụ: `unixperms` cho Linux hoặc `windows-owner` cho Windows) mà không cần can thiệp vào mã nguồn lõi.
*   **Cấu hình tập trung (Centralized Config):** Cả Python crawler và PHP web app đều đọc chung một file SQLite để đảm bảo các thiết lập về Elasticsearch, loại trừ tệp (excludes), hoặc thay thế đường dẫn (path replacement) luôn đồng bộ.

### 3. Các Kỹ Thuật Chính (Key Techniques)

*   **Quét song song (Parallel Scanning):** Sử dụng đa luồng (`MAXTHREADS`, `INDEXTHREADS`) để tối ưu hóa tốc độ đọc hệ thống tệp. Bản 2.3.1 đã cải tiến thuật toán luồng song song để nhanh hơn.
*   **Tối ưu hóa Elasticsearch (ES Tuning):**
    *   Sử dụng **Bulk API** để đẩy dữ liệu lên ES theo từng khối (`chunk_size`), giảm thiểu overhead mạng.
    *   Tự động điều chỉnh `refresh_interval` và `translog` trong quá trình quét để ưu tiên tốc độ ghi (indexing) hơn là tốc độ đọc tức thời.
*   **Xử lý Unicode và Đường dẫn dài:** Kỹ thuật xử lý các tệp có ký tự đặc biệt hoặc đường dẫn vượt quá giới hạn ký tự (đặc biệt trên Windows với tiền tố `\\?\`).
*   **Phân tích dữ liệu "Nóng/Lạnh" (Hot/Cold Data):** Sử dụng các truy vấn Aggregation của ES để phân loại tệp dựa trên thời gian truy cập cuối (`atime`) hoặc sửa đổi cuối (`mtime`), giúp xác định dữ liệu rác hoặc dữ liệu lâu không dùng.
*   **Path Replacement (Thay thế đường dẫn):** Kỹ thuật ánh xạ đường dẫn từ các mount point khác nhau (ví dụ: từ Windows `Z:\` sang Linux `/mnt/storage`) để hiển thị thống nhất trên web.

### 4. Luồng Hoạt Động Của Hệ Thống (System Workflow)

Luồng hoạt động chia làm hai giai đoạn chính:

#### Giai đoạn 1: Thu thập (Indexing)
1.  Người dùng chạy lệnh: `python3 diskover.py -i <index_name> <path>`.
2.  **Khởi tạo:** Kết nối Elasticsearch, tạo Index với các `mappings` (định nghĩa kiểu dữ liệu) tối ưu cho tìm kiếm tệp.
3.  **Crawl:** Duyệt cây thư mục. Áp dụng các bộ lọc loại trừ (Excludes) để bỏ qua tệp hệ thống, tệp tạm.
4.  **Metadata Extraction:** Thu thập thông tin (kích thước, chủ sở hữu, thời gian, inode...). Chạy các plugin bổ trợ.
5.  **Bulk Upload:** Đưa dữ liệu vào Elasticsearch theo từng đợt lớn.
6.  **Kết thúc:** Ghi nhận thông tin tổng quan (spaceinfo) như tổng dung lượng ổ đĩa, phần trăm đã dùng.

#### Giai đoạn 2: Phân tích và Hiển thị (Web App)
1.  **Người dùng truy cập:** Đăng nhập qua PHP app.
2.  **Lựa chọn Index:** Người dùng chọn Index vừa quét trong trang "Indices".
3.  **Truy vấn:** Khi người dùng tìm kiếm hoặc xem Dashboard, PHP gửi truy vấn (Query DSL) tới Elasticsearch.
4.  **Trực quan hóa:**
    *   Dữ liệu trả về được PHP định dạng lại thành JSON.
    *   D3.js vẽ biểu đồ cây (File tree) hoặc biểu đồ tròn/thanh (Pie/Bar charts) về các loại tệp chiếm dụng nhiều bộ nhớ nhất.
5.  **Quản lý:** Người dùng có thể đánh dấu, lọc tệp (Filters) trực tiếp trên giao diện để đưa ra quyết định giải phóng lưu trữ.

### Tổng kết
Diskover là một giải pháp quản trị dữ liệu thông minh, không chỉ đơn thuần là tìm kiếm tệp mà còn là một công cụ **Storage Business Intelligence**. Nó biến các thông tin thô từ hệ thống tệp thành các hiểu biết có giá trị về chi phí và hiệu quả sử dụng tài nguyên lưu trữ.