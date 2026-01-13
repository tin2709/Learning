Dựa trên các tệp tin và mô tả từ kho lưu trữ **Amurex Backend**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của dự án này bằng tiếng Việt:

---

### 1. Phân tích Công nghệ Cốt lõi (Core Technologies)

Dự án sử dụng một "Stack" hiện đại, tập trung vào khả năng xử lý ngôn ngữ tự nhiên (NLP) và tính linh hoạt giữa đám mây (Cloud) và máy cục bộ (Local):

*   **Ngôn ngữ lập trình:** **Python 3.11** (Tối ưu cho các thư viện AI và xử lý dữ liệu).
*   **Cơ sở dữ liệu & Backend as a Service (BaaS):**
    *   **Supabase:** Sử dụng để quản lý người dùng, lưu trữ dữ liệu quan hệ (PostgreSQL) và lưu trữ tệp tin (Storage) cho các tài liệu ngữ cảnh cuộc họp.
    *   **SQLite:** Có file `database/db_manager.py` và cấu hình timeout, thường dùng cho lưu trữ đệm hoặc dữ liệu nhẹ tại địa phương.
*   **Xử lý hàng đợi & Bộ nhớ đệm:** **Redis** (Dùng để xử lý các tác vụ bất đồng bộ hoặc lưu trữ phiên làm việc).
*   **Trí tuệ nhân tạo (AI Models):**
    *   **Cloud API:** Hỗ trợ OpenAI, Groq, Mistral AI.
    *   **Local Inference:** Hỗ trợ **Ollama** (chạy LLM cục bộ) và **fast-embed** (tạo vector embedding cục bộ).
*   **Hạ tầng:** **Docker & Docker Compose** (Giúp triển khai nhanh chóng và đồng nhất môi trường).
*   **Giao tiếp:** **Resend API** (Gửi email tóm tắt cuộc họp).

---

### 2. Kỹ thuật và Tư duy Kiến trúc (Architectural Design)

*   **Kiến trúc Hybrid (Lai):** Đây là điểm nổi bật nhất. Hệ thống cho phép chuyển đổi giữa `CLIENT_MODE=ONLINE` (dùng API trả phí, tốc độ cao) và `CLIENT_MODE=LOCAL` (dùng Ollama, ưu tiên quyền riêng tư và miễn phí).
*   **Kiến trúc hướng dữ liệu (Migration-driven):** Việc sử dụng các tệp SQL migration (`supabase/migrations/`) cho thấy tư duy quản lý phiên bản cơ sở dữ liệu chặt chẽ, giúp việc nâng cấp hệ thống không làm mất dữ liệu.
*   **Containerization (Đóng gói):** Sử dụng Docker giúp tách biệt môi trường phần mềm với phần cứng, dễ dàng mở rộng (scale) hoặc triển khai lên các dịch vụ đám mây như AWS, GCP hay DigitalOcean.
*   **Quản lý Ngữ cảnh (Context Management):** Hệ thống được thiết kế để xử lý "meeting_context_files", cho thấy tư duy RAG (Retrieval-Augmented Generation) - tức là AI không chỉ trả lời dựa trên kiến thức sẵn có mà còn dựa trên tài liệu người dùng tải lên.

---

### 3. Các kỹ thuật chính nổi bật

1.  **Xử lý Embedding cục bộ:** Sử dụng `fast-embed` để chuyển đổi văn bản thành vector ngay tại máy chủ local mà không cần gửi dữ liệu ra ngoài, giúp tăng bảo mật cho các nội dung cuộc họp nhạy cảm.
2.  **Quản lý phiên làm việc với SQLite/Redis:** Kết hợp giữa Redis (tốc độ cao) và SQLite (bền vững) để đảm bảo hệ thống ổn định ngay cả khi có sự cố kết nối.
3.  **Hệ thống thông báo tự động:** Tích hợp luồng gửi Email sau khi cuộc họp kết thúc, tạo ra một vòng lặp trải nghiệm người dùng khép kín (Họp -> Tóm tắt -> Gửi kết quả).
4.  **Cấu hình linh hoạt qua Biến môi trường (.env):** Cho phép thay đổi toàn bộ hành vi của Backend (từ model AI đến endpoint database) mà không cần sửa mã nguồn.

---

### 4. Tóm tắt Luồng hoạt động của Dự án

Dựa trên cấu trúc code, quy trình hoạt động của Amurex Backend như sau:

1.  **Khởi tạo (Setup):**
    *   Người dùng thiết lập các bảng trong Supabase thông qua các file migration (Users, Meetings, Memories).
    *   Cấu hình các API Key (OpenAI, Groq...) hoặc cài đặt Ollama nếu chạy local.
2.  **Tiếp nhận dữ liệu (Ingestion):**
    *   Dữ liệu cuộc họp (âm thanh, văn bản hoặc file ngữ cảnh) được đẩy lên backend qua API.
    *   Các tệp tin đi kèm được lưu trữ vào Supabase Bucket (`meeting_context_files`).
3.  **Xử lý AI (Processing):**
    *   Hệ thống gọi đến các LLM (qua API hoặc Ollama).
    *   Thực hiện các tác vụ: Tóm tắt nội dung, trích xuất các ý chính, hành động cần làm (Action items).
    *   Tạo vector embedding cho dữ liệu để có thể tra cứu lại sau này (Memories).
4.  **Lưu trữ & Phản hồi (Storage & Output):**
    *   Kết quả tóm tắt được lưu vào bảng `Meetings` trong cơ sở dữ liệu.
    *   Hệ thống gọi Resend API để gửi một bản tóm tắt đẹp mắt qua Email cho người dùng.
5.  **Truy vấn lại (Retrieval):**
    *   Người dùng có thể tra cứu lại các "ký ức" (memories) từ các cuộc họp cũ nhờ vào dữ liệu đã được vector hóa và lưu trữ.

**Kết luận:** Amurex Backend là một hệ thống quản lý cuộc họp thông minh, chú trọng vào quyền riêng tư (hỗ trợ Local AI) và khả năng tự động hóa quy trình hậu cuộc họp.