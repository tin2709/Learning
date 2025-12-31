Dưới đây là bản phân tích chi tiết về dự án **Katana** của ProjectDiscovery dựa trên mã nguồn và tài liệu bạn cung cấp.

---

# Phân tích Kỹ thuật Dự án Katana (ProjectDiscovery)

## 1. Công nghệ cốt lõi (Core Technology)
Katana được xây dựng bằng ngôn ngữ **Go**, tận dụng tối đa khả năng xử lý song song và hiệu suất cao của ngôn ngữ này. Các thư viện và công nghệ chính bao gồm:

*   **Go (1.24+):** Sử dụng các tính năng mới nhất của Go để tối ưu quản lý bộ nhớ và concurrency.
*   **Headless Engine (go-rod):** Sử dụng thư viện `rod` để điều khiển trình duyệt Chrome/Chromium, cho phép render JavaScript và tương tác với các ứng dụng Web hiện đại (Single Page Apps).
*   **HTML Parsing:** Sử dụng `goquery` (tương tự jQuery) để bóc tách dữ liệu từ cấu trúc DOM.
*   **JavaScript Parsing:** Sử dụng `jsluice` để trích xuất các endpoint (URL) ẩn bên trong các file mã nguồn JavaScript mà không cần thực thi chúng hoàn toàn.
*   **Networking:** Sử dụng `fastdialer` và `retryablehttp-go` (từ hệ sinh thái của ProjectDiscovery) để tùy biến sâu vào quá trình bắt tay TLS, xử lý lỗi mạng và tăng tốc độ truy vấn.
*   **DSL (Domain Specific Language):** Cho phép người dùng viết các điều kiện lọc (filter/match) phức tạp thông qua cú pháp giống lập trình.

## 2. Tư duy kiến trúc (Architectural Thinking)
Kiến trúc của Katana được thiết kế theo hướng **"Module hóa và Chuyên biệt hóa"**:

*   **Cơ chế Engine kép (Standard & Hybrid):**
    *   *Standard:* Chạy bằng thư viện HTTP thuần (nhanh, ít tốn tài nguyên).
    *   *Hybrid/Headless:* Chạy bằng trình duyệt thực (độ phủ cao, render được JS).
    *   Kiến trúc này cho phép người dùng đánh đổi giữa **tốc độ** và **độ chính xác**.
*   **Thiết kế cho Pipeline:** Katana hỗ trợ mạnh mẽ việc nhận đầu vào từ `STDIN` và xuất ra `JSONL`. Điều này giúp nó dễ dàng kết hợp với các công cụ khác như `subfinder`, `httpx`, `nuclei`.
*   **Kiểm soát phạm vi (Scope-First):** Tránh việc crawl tràn lan (infinite loop) bằng cách định nghĩa chặt chẽ Scope (RDN, FQDN, Regex). Đây là tư duy sống còn của một công cụ spidering chuyên nghiệp.
*   **Tách biệt Logic và Runner:** Phần `internal/runner` quản lý vòng đời của ứng dụng (khởi tạo, xử lý cờ, quản lý trạng thái), trong khi `pkg/engine` tập trung vào logic crawl thực sự.

## 3. Các kỹ thuật chính (Key Techniques)
*   **Concurrency & Rate Limiting:** Sử dụng `SizedWaitGroup` để quản lý số lượng luồng chạy đồng thời (`parallelism` cho đầu vào và `concurrency` cho từng mục tiêu) và `ratelimit` để tránh bị hệ thống đích chặn (WAF/Anti-bot).
*   **Automatic Form Filling:** Tự động điền dữ liệu vào các thẻ `<form>` dựa trên cấu hình (YAML) để khám phá các trang sau khi submit.
*   **State Management (Resume):** Khả năng lưu trạng thái vào file `resume.cfg` khi bị ngắt quãng (Ctrl+C) và tiếp tục crawl sau đó.
*   **Technological Detection:** Tích hợp `wappalyzergo` để nhận diện công nghệ (CMS, Framework) của trang web ngay trong quá trình crawl.
*   **Custom Output Templating:** Cho phép người dùng tự định nghĩa định dạng đầu ra bằng Go templates (ví dụ: chỉ lấy email và URL).
*   **XHR Extraction:** Kỹ thuật đặc biệt trong chế độ Headless để bắt các request ngầm (AJAX) mà trình duyệt thực hiện.

## 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Khởi tạo (Initialization):**
    *   Đọc các tham số từ dòng lệnh (Flags).
    *   Load cấu hình tùy chỉnh (Form config, Field config).
    *   Khởi tạo Engine phù hợp (Standard hoặc Headless).
2.  **Xử lý đầu vào (Input Processing):**
    *   Nhận URL từ tham số `-u`, file `-list` hoặc qua đường ống `STDIN`.
    *   Kiểm tra tính hợp lệ của URL và chính sách mạng (Network Policy - tránh crawl các IP nội bộ nếu không cho phép).
3.  **Thực thi Crawling (Execution):**
    *   **Dispatcher:** Điều phối các URL vào hàng đợi (Queue).
    *   **Fetcher:** Thực hiện yêu cầu HTTP hoặc render trình duyệt.
    *   **Parser:** Trích xuất link từ: HTML tags (a, src, href), JS code, Headers (Location), và các file đặc biệt (robots.txt, sitemap.xml).
    *   **Scope Filter:** Kiểm tra các link mới tìm thấy có nằm trong phạm vi cho phép (Scope) hay không.
4.  **Lọc và Kết quả (Filtering & Output):**
    *   Áp dụng các bộ lọc Extension, Regex hoặc DSL.
    *   Ghi kết quả ra Terminal (STDOUT), file văn bản hoặc file JSONL theo thời gian thực.
5.  **Hoàn tất (Finalization):**
    *   Dọn dẹp tài nguyên (đóng trình duyệt, đóng file).
    *   Lưu trạng thái nếu cần thiết.

---
*Bản readme tóm tắt này giúp lập trình viên nhanh chóng nắm bắt cấu trúc và logic vận hành của Katana mà không cần đọc từng dòng code.*