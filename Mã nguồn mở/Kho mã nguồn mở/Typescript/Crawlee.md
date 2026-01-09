Dưới đây là bản phân tích chi tiết và tóm tắt dự án **Crawlee** dưới dạng file README tiếng Việt, tập trung vào kiến trúc và công nghệ cốt lõi.

---

# Crawlee - Phân tích Kỹ thuật & Kiến trúc Hệ thống

**Crawlee** là một thư viện Node.js mạnh mẽ dành cho web scraping và tự động hóa trình duyệt, được thiết kế để xây dựng các bot có độ tin cậy cao, hiệu suất lớn và khả năng giả lập hành vi người dùng cực tốt.

## 1. Công nghệ Cốt lõi (Core Tech Stack)

*   **Ngôn ngữ:** TypeScript (chiếm >60%), sử dụng triệt để Generics và Interfaces để đảm bảo an toàn kiểu dữ liệu.
*   **Mô hình Monorepo:** Sử dụng **Lerna** và **Turborepo** để quản lý hàng chục gói (packages) trong cùng một kho lưu trữ, giúp tái sử dụng mã nguồn hiệu quả.
*   **Công cụ Build/Lint:** **Biome** (thay thế Prettier/ESLint cho tốc độ cao), **Vite/Vitest** để testing.
*   **Thư viện HTTP & Browser:**
    *   **Got-scraping:** Một bản build tùy chỉnh của `got` tối ưu cho việc giả lập trình duyệt (TLS fingerprints, headers).
    *   **Cheerio / JSDOM / Linkedom:** Các trình phân tích cú pháp HTML siêu nhanh.
    *   **Puppeteer / Playwright:** Điều khiển trình duyệt không đầu (headless browsers).

## 2. Kỹ thuật & Tư duy Kiến trúc (Architectural Thinking)

### Kiến trúc Đa tầng (Layered Architecture)
Crawlee được xây dựng dựa trên sự kế thừa:
*   **BasicCrawler:** Tầng thấp nhất, xử lý logic lặp (loop), quản lý hàng đợi và lỗi.
*   **HttpCrawler/CheerioCrawler:** Tầng trung gian, thêm lớp xử lý HTTP và phân tích cú pháp HTML tĩnh.
*   **BrowserCrawler (Puppeteer/Playwright):** Tầng cao nhất, tích hợp điều khiển trình duyệt thực, thực thi JavaScript.

### Tư duy "Resilient by Default" (Mặc định là bền bỉ)
Hệ thống được thiết kế để không bao giờ mất dữ liệu:
*   **RequestQueue:** Hàng đợi được lưu trữ trên đĩa (Disk-backed). Nếu scraper bị crash, nó sẽ tiếp tục từ URL đang chạy dở.
*   **Automatic Retries:** Tự động thử lại khi gặp lỗi mạng hoặc bị proxy chặn.

### Abstraction (Trừu tượng hóa)
Người dùng chỉ cần quan tâm đến `requestHandler`. Tất cả các yếu tố phức tạp như: xoay vòng proxy, quản lý session, tạo vân tay trình duyệt (fingerprinting) đều được ẩn bên dưới lớp trừu tượng.

## 3. Các kỹ thuật chính nổi bật (Key Technical Features)

### A. AutoscaledPool (Quản lý tài nguyên tự động)
Đây là "bộ não" điều phối hiệu suất. Hệ thống sẽ:
1.  Theo dõi CPU và RAM của máy chủ theo thời gian thực.
2.  Tự động tăng số lượng luồng (concurrency) khi tài nguyên còn trống.
3.  Giảm luồng ngay lập tức khi hệ thống có dấu hiệu quá tải để tránh bị crash.

### B. Anti-Fingerprinting & Stealth (Vượt rào cản bot)
Crawlee không chỉ mở trình duyệt; nó tạo ra một thực thể "người dùng" thực thụ:
*   **Fingerprint Generation:** Tạo các thông số phần cứng, độ phân giải màn hình, danh sách font... ngẫu nhiên nhưng hợp lệ.
*   **TLS/SSL Fingerprinting:** Giả lập cách các trình duyệt thực thụ (Chrome, Firefox) bắt tay với server để tránh bị phát hiện là thư viện HTTP thô.
*   **Camoufox:** Tích hợp các trình duyệt được tùy chỉnh riêng để vượt qua các hệ thống bảo mật cao cấp như Cloudflare.

### C. SessionPool (Quản lý trạng thái)
Tự động quản lý Cookies và IP Proxy. Nếu một Session bị website chặn (403/429), Crawlee sẽ:
1.  Đánh dấu Session đó là "xấu".
2.  Loại bỏ Proxy tương ứng.
3.  Tự động tạo Session mới và thử lại yêu cầu.

## 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng xử lý của Crawlee trải qua các bước sau:

1.  **Khởi tạo (Initialization):**
    *   Nạp danh sách URL ban đầu vào `RequestQueue`.
    *   Cấu hình `ProxyConfiguration` và `SessionPool`.

2.  **Lấy Request (Request Fetching):**
    *   `AutoscaledPool` kiểm tra tài nguyên hệ thống. Nếu đủ, nó sẽ lấy URL tiếp theo từ hàng đợi.

3.  **Chuẩn bị Môi trường (Pre-processing):**
    *   Gán một `Session` và `Proxy` cho yêu cầu.
    *   Tạo `Fingerprint` (nếu dùng trình duyệt).
    *   Kích hoạt các `preNavigationHooks`.

4.  **Thực thi (Navigation & Scraping):**
    *   Gửi request (qua HTTP hoặc Browser).
    *   Tải nội dung trang web.
    *   Chạy `requestHandler` của người dùng để trích xuất dữ liệu.

5.  **Mở rộng (Links Enqueueing):**
    *   Sử dụng `enqueueLinks()` để tìm các URL mới trên trang vừa tải.
    *   Lọc URL theo pattern và thêm ngược lại vào `RequestQueue`.

6.  **Lưu trữ (Storage):**
    *   Dữ liệu được đẩy vào `Dataset` (lưu thành các file JSON/CSV cục bộ hoặc đẩy lên Cloud).

7.  **Kết thúc (Finalization):**
    *   Sau khi hết URL hoặc đạt giới hạn, hệ thống chạy `postNavigationHooks` và đóng các trình duyệt/kết nối.

## 5. Cấu trúc thư mục dự án (Monorepo Overview)

*   `packages/core`: Logic cốt lõi (Storage, Crawler base, Events).
*   `packages/cheerio-crawler`: Scraping tốc độ cao dựa trên parse HTML.
*   `packages/playwright-crawler`: Scraping dựa trên trình duyệt (hiện đại, hỗ trợ nhiều browser).
*   `packages/browser-pool`: Quản lý việc tái sử dụng các instance trình duyệt để tiết kiệm RAM.
*   `packages/utils`: Các công cụ bổ trợ (xử lý URL, sitemap, social data).

---
*Phân tích bởi AI dựa trên mã nguồn và tài liệu dự án Crawlee (Cập nhật 2025).*