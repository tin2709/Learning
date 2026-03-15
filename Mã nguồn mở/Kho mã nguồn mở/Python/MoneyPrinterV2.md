Chào bạn, đây là bản phân tích chuyên sâu về hệ thống **MoneyPrinter V2 (MPV2)** dựa trên mã nguồn và cấu trúc tài liệu bạn đã cung cấp. Đây là một công cụ tự động hóa quy trình tạo nội dung và tiếp thị đa kênh nhằm mục đích kiếm tiền online.

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

MPV2 được xây dựng trên nền tảng Python hiện đại với sự kết hợp của trí tuệ nhân tạo (AI) cục bộ và đám mây:

*   **Ngôn ngữ lập trình:** Python 3.12.
*   **Trí tuệ nhân tạo (LLM):** Sử dụng **Ollama** (chạy cục bộ) để tạo kịch bản, ý tưởng nội dung và các đoạn hội thoại tiếp thị.
*   **Xử lý hình ảnh (AI Gen):** Tích hợp **Nano Banana 2** (Gemini Image API) để tạo hình ảnh minh họa cho video thay vì dùng stock footage truyền thống.
*   **Xử lý âm thanh & Video:**
    *   **KittenTTS:** Chuyển đổi văn bản thành giọng nói (Text-to-Speech).
    *   **MoviePy:** "Trái tim" của việc biên tập video, dùng để cắt ghép hình ảnh, lồng tiếng, chèn nhạc nền và tạo phụ đề.
    *   **Faster-Whisper / AssemblyAI:** Dùng để chuyển đổi giọng nói thành văn bản (Speech-to-Text) phục vụ việc tạo file phụ đề (.srt).
*   **Trình duyệt tự động hóa:** **Selenium** kết hợp với **Firefox Profiles**. Thay vì đăng nhập tự động (dễ bị chặn), hệ thống sử dụng các profile trình duyệt đã đăng nhập sẵn.
*   **Dữ liệu & Scraper:** 
    *   **Go (Golang):** Dùng để build công cụ quét dữ liệu Google Maps mạnh mẽ.
    *   **Scraping:** Trích xuất thông tin sản phẩm Amazon và thông tin doanh nghiệp địa phương.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của MPV2 đi theo hướng **Modular CLI-First Design**:

*   **Tính module hóa (Modularity):** Mỗi nguồn thu nhập (Revenue Stream) được đóng gói trong một class riêng biệt tại `src/classes/` (YouTube, Twitter, AFM, Outreach). Điều này cho phép mở rộng dễ dàng các tính năng mới mà không ảnh hưởng đến lõi hệ thống.
*   **Cơ chế lưu trữ cục bộ (Local Persistence):** Sử dụng thư mục ẩn `.mp/` làm nơi lưu trữ tạm thời (Scratchpad) cho các tệp đa phương tiện và các file JSON để quản lý trạng thái tài khoản/danh sách đã đăng.
*   **Hybrid Execution:** Hệ thống hỗ trợ hai chế độ vận hành: 
    *   **Interactive (Chế độ tương tác):** Thông qua `main.py`, người dùng thiết lập tài khoản và kiểm tra quy trình.
    *   **Headless/Cron (Chế độ tự động):** Thông qua `cron.py` và thư viện `schedule`, các tác vụ được chạy ngầm theo giờ định sẵn mà không cần sự can thiệp của con người.
*   **Tận dụng tối đa Open Source:** Thay vì xây dựng lại từ đầu, dự án tích hợp các công cụ mạnh mẽ như Ollama để giảm chi phí API và tăng tính riêng tư.

---

### 3. Các kỹ thuật chính (Key Technical Techniques)

*   **Multimedia Orchestration (Điều phối đa phương tiện):** Đây là kỹ thuật phức tạp nhất trong class `YouTube.py`. Hệ thống phải tính toán thời lượng audio của TTS, sau đó chia đều thời gian hiển thị các ảnh AI đã tạo để khớp hoàn hảo với giọng đọc.
*   **Selenium Profile Injection:** Kỹ thuật vượt rào cản bảo mật của YouTube/X. Bằng cách trỏ đường dẫn profile Firefox thực tế vào Selenium, hệ thống bỏ qua bước đăng nhập và các lớp xác thực 2 yếu tố (2FA) phức tạp.
*   **Subtitle Equalization:** Sử dụng `srt_equalizer` để đảm bảo phụ đề hiển thị trên YouTube Shorts không bị quá dài, giúp trải nghiệm người dùng trên thiết bị di động tốt hơn (mỗi dòng chỉ khoảng vài từ).
*   **Email Outreach Automation:** Sử dụng `yagmail` (SMTP) kết hợp với các template HTML để cá nhân hóa nội dung gửi đến doanh nghiệp (thay thế `{{COMPANY_NAME}}` động).

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Hệ thống hoạt động theo 3 luồng chính:

#### Luồng 1: YouTube Shorts Automation (Quy trình sản xuất video)
1.  **Ý tưởng:** LLM tạo chủ đề dựa trên "Niche" (ngách) của tài khoản.
2.  **Sản xuất nội dung:** LLM viết kịch bản -> TTS tạo file âm thanh -> LLM tạo các câu lệnh (prompts) hình ảnh -> Gemini API tạo ảnh PNG.
3.  **Hậu kỳ:** MoviePy ghép ảnh, chèn nhạc nền, chèn TTS, tạo phụ đề tự động từ Whisper.
4.  **Phân phối:** Selenium mở YouTube Studio, tải video lên và thiết lập các thông số (tiêu đề, mô tả, tag).

#### Luồng 2: Affiliate Marketing (AFM)
1.  **Thu thập:** Người dùng cung cấp link Amazon.
2.  **Phân tích:** Selenium vào link Amazon, quét tiêu đề và các tính năng nổi bật (bullet points).
3.  **Sáng tạo:** LLM dựa trên dữ liệu quét được để viết một đoạn tweet "mồi chài" hấp dẫn.
4.  **Đăng tải:** Selenium đăng bài lên X (Twitter) kèm link affiliate.

#### Luồng 3: Local Business Outreach (Tiếp thị doanh nghiệp)
1.  **Quét:** Chạy binary Go để quét danh sách doanh nghiệp trên Google Maps theo từ khóa.
2.  **Trích xuất:** Hệ thống vào website của từng doanh nghiệp để tìm địa chỉ email bằng Regex.
3.  **Tiếp cận:** Gửi email hàng loạt nhưng có cá nhân hóa thông qua SMTP.

### Kết luận
MoneyPrinter V2 là một hệ thống **Content Factory** (Nhà máy nội dung) hoàn chỉnh. Nó thể hiện sự dịch chuyển từ việc sử dụng các API trả phí đắt đỏ sang việc tự vận hành (Self-hosted) thông qua Ollama và xử lý đa phương tiện ngay tại máy trạm, giúp tối ưu hóa lợi nhuận cho người sử dụng.