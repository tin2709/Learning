Dưới đây là bản phân tích kỹ thuật chi tiết về dự án **ArchiveBox** dựa trên mã nguồn và tài liệu bạn cung cấp, được trình bày bằng tiếng Việt.

---

# Phân tích Kỹ thuật Dự án ArchiveBox

ArchiveBox là một giải pháp lưu trữ web (web archiving) tự lưu trữ (self-hosted), mã nguồn mở, được thiết kế với tư duy "dữ liệu là vĩnh cửu".

## 1. Công nghệ cốt lõi (Core Technologies)

Dự án được xây dựng trên một ngăn xếp công nghệ mạnh mẽ và hiện đại:

*   **Ngôn ngữ lập trình chính:** **Python 3.11+** (Khuyến nghị 3.13). Sử dụng **Django 6.0** làm khung xương cho quản trị dữ liệu và giao diện web.
*   **Quản lý gói & Môi trường:** **uv** (thay thế cho pip). Đây là công cụ quản lý Python cực nhanh, giúp đồng bộ hóa dependencies và venv hiệu quả.
*   **API Framework:** **Django Ninja**. Một framework hiện đại giúp xây dựng REST API nhanh chóng với Pydantic để kiểm tra kiểu dữ liệu (type hinting).
*   **Cơ sở dữ liệu:**
    *   **SQLite:** Lưu trữ metadata, trạng thái các bản ghi (Snapshots, Crawls).
    *   **Sonic / Ripgrep:** Các backend cho tìm kiếm toàn văn (Full-text search).
*   **Công cụ trích xuất (Extractors/Plugins):**
    *   **Chromium (thông qua Playwright/Puppeteer):** Để chụp ảnh màn hình, tạo PDF và lưu SingleFile.
    *   **Wget & Curl:** Lưu mã nguồn HTML truyền thống.
    *   **yt-dlp:** Tải video và phương tiện truyền thông.
    *   **Readability/Mercury:** Trích xuất nội dung văn bản thuần túy từ bài báo.
*   **Hệ điều hành & Container:** **Docker & Docker Compose** là phương thức triển khai chính để đóng gói các phụ thuộc phức tạp (Chrome, Node.js, v.v.).

## 2. Kỹ thuật và Tư duy Kiến trúc (Architecture & Philosophy)

Kiến trúc của ArchiveBox phản ánh tư duy "dữ liệu quan trọng hơn mã nguồn":

### A. Triết lý "Filesystem-First"
Mặc dù sử dụng SQLite để quản lý, nhưng mọi dữ liệu lưu trữ thực tế đều được tổ chức dưới dạng các thư mục và file thông thường trên đĩa cứng. Điều này đảm bảo rằng ngay cả khi ArchiveBox không còn hoạt động, dữ liệu của bạn vẫn có thể đọc được bằng các công cụ tiêu chuẩn (trình duyệt, trình xem PDF).

### B. Hệ thống Plugin dựa trên Hook
ArchiveBox sử dụng một hệ thống hook mạnh mẽ (`archivebox/hooks.py`) để quản lý các plugin.
*   **Phân bậc thực thi:** Các hook được đánh số (00-99). Ví dụ: `on_Snapshot__50_screenshot.js` chạy ở bước 50.
*   **Đa ngôn ngữ:** Hook có thể là Python (.py), JavaScript (.js) chạy qua Node.js, hoặc Bash script (.sh).
*   **Tính cô lập:** Các plugin chạy dưới dạng các tiến trình (process) riêng biệt, giao tiếp qua tham số CLI và đầu ra JSONL.

### C. Quản lý trạng thái bằng State Machine
Quá trình lưu trữ một Snapshot đi qua nhiều trạng thái (Queued -> Started -> Succeeded/Failed). Dự án sử dụng thư viện `python-statemachine` để kiểm soát chặt chẽ luồng vòng đời này, đảm bảo tính nhất quán khi xử lý hàng nghìn URL.

### D. Bảo mật & Cô lập
*   **Ngăn chặn quyền Root:** ArchiveBox từ chối chạy dưới quyền root để bảo vệ hệ thống vật lý.
*   **Personas:** Cho phép quản lý các cấu hình trình duyệt khác nhau (cookies, user-agents) để lưu trữ các trang web yêu cầu đăng nhập một cách an toàn.

## 3. Các kỹ thuật chính nổi bật

1.  **MCP (Model Context Protocol):** Dự án tích hợp `archivebox_mcp.py`, cho phép các tác nhân AI (như Claude) có thể trực tiếp tương tác, điều khiển và trích xuất dữ liệu từ kho lưu trữ thông qua giao thức RPC.
2.  **JSONL Logging:** Sử dụng định dạng JSON Lines để ghi log và trao đổi dữ liệu giữa các tiến trình. Đây là kỹ thuật giúp xử lý dòng dữ liệu lớn (streaming) mà không cần nạp toàn bộ vào bộ nhớ.
3.  **Hệ thống phân cấp Worker/Orchestrator:** `orchestrator.py` quản lý việc phân phối các tác vụ lưu trữ cho các Worker, kiểm soát số lượng tiến trình đồng thời để tránh làm treo máy chủ.
4.  **Hỗ trợ LDAP/Auth:** Tích hợp sẵn các phương thức xác thực mạnh mẽ cho môi trường doanh nghiệp.

## 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng xử lý điển hình trong dự án diễn ra như sau:

1.  **Input (Đầu vào):** Người dùng thêm URL thông qua CLI, Web UI, hoặc Browser Extension.
2.  **Crawl Creation:** Hệ thống tạo một bản ghi `Crawl`. Nếu là crawl đệ quy (depth > 0), nó sẽ phân tích các liên kết trong trang gốc.
3.  **Snapshot Queue:** Mỗi URL trở thành một `Snapshot`. Orchestrator quét các Snapshot mới và tạo các `ArchiveResult` (các công việc cần làm cho từng plugin như screenshot, pdf, wget).
4.  **Plugin Execution (Hook System):**
    *   Hệ thống gọi các plugin tương ứng (ví dụ: gọi Chrome để chụp ảnh).
    *   Plugin lưu file trực tiếp vào thư mục snapshot trên ổ đĩa.
    *   Plugin trả về kết quả JSONL cho ArchiveBox.
5.  **Indexing & Search:** Metadata (tiêu đề, thời gian, trạng thái) được cập nhật vào SQLite. Nội dung văn bản được đẩy vào Sonic/Ripgrep để phục vụ tìm kiếm sau này.
6.  **Storage:** Kết quả cuối cùng là một thư mục chứa đầy đủ các định dạng: `index.html`, `screenshot.png`, `source.html`, `output.pdf`, v.v.

---
**Kết luận:** ArchiveBox là một dự án có kỹ thuật rất bài bản, kết hợp giữa sự linh hoạt của Plugin đa ngôn ngữ và sự bền vững của lưu trữ file truyền thống. Việc chuyển sang sử dụng `uv` và `Django Ninja` cho thấy dự án đang duy trì tiêu chuẩn mã nguồn rất cao và hiện đại.