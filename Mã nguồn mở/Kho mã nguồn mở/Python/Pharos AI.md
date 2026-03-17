Chào bạn, đây là bản phân tích chi tiết về dự án **Pharos AI** – một nền tảng tình báo nguồn mở (OSINT) hiện đại chuyên theo dõi các cuộc xung đột địa chính trị (hiện tại là xung đột Iran 2026).

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án sử dụng một "stack" công nghệ rất hiện đại, tập trung vào hiệu suất hiển thị dữ liệu thời gian thực và khả năng xử lý ngôn ngữ tự nhiên:

*   **Frontend Framework:** Next.js (phiên bản mới nhất hỗ trợ React 19) với App Router, đảm bảo tối ưu hóa Server-Side Rendering (SSR) và Client-side interactivity.
*   **Data Visualization (Bản đồ & Biểu đồ):**
    *   **DeckGL + MapLibre:** Đây là bộ đôi mạnh mẽ để render các lớp dữ liệu lớn trên bản đồ (như đường đi của tên lửa, vùng đe dọa, điểm nóng quân sự).
    *   **Lightweight Charts:** Dùng để hiển thị các chỉ số kinh tế và dự đoán thị trường.
*   **Backend & Database:**
    *   **Prisma 7:** ORM thế hệ mới giúp quản lý schema database phức tạp một cách chặt chẽ.
    *   **PostgreSQL 17 + pgvector:** Sử dụng cơ sở dữ liệu quan hệ kết hợp với phần mở rộng vector để hỗ trợ tìm kiếm ngữ nghĩa (Semantic Search) cho hệ thống RAG (Retrieval-Augmented Generation).
*   **AI & Intel Ingestion:**
    *   **OpenAI & X AI (Grok):** Được sử dụng để xác minh thông tin từ mạng xã hội (X/Twitter) và cung cấp khả năng chat thông minh.
    *   **OpenClaw:** Hệ thống Agent tự động để thu thập và biên soạn dữ liệu tình báo.
*   **UI/UX:** Tailwind CSS phối hợp với **shadcn/ui** để tạo ra giao diện "Tactical UI" (giao diện tác chiến) chuyên nghiệp.

### 2. Tư duy Kiến trúc (Architectural Mindset)

Kiến trúc của Pharos được thiết kế theo hướng **Feature-driven Modular Architecture** (Kiến trúc mô-đun dựa trên tính năng):

*   **Tính mô-đun cao:** Toàn bộ mã nguồn được chia vào `src/features/`. Mỗi tính năng (như bản đồ, sự kiện, diễn biến kinh tế, actor) đều tự chứa các thành phần (components), logic truy vấn (queries) và hooks riêng. Điều này giúp hệ thống dễ dàng mở rộng khi có loại dữ liệu mới.
*   **Phân tách Agent và Dashboard:** Phần Agent thu thập dữ liệu (`agent/`) và phần hiển thị (`src/`) hoạt động độc lập qua hệ thống Admin API. Agent đóng vai trò là "người thực thi" (fulfillment), chuyển hóa tin tức thô thành dữ liệu có cấu trúc.
*   **Tư duy hướng dữ liệu (Data-Centric):** Xung đột được mô hình hóa qua các thực thể: *Conflict* (Gốc) -> *Events* (Sự kiện) -> *Actors* (Các bên liên quan) -> *Map Features* (Dữ liệu không gian). Mối quan hệ này cho phép người dùng xem một sự kiện từ nhiều góc độ (bản đồ, timeline, hoặc hồ sơ đối tượng).
*   **Hệ thống Snapshot:** Một tư duy rất hay là dự án công khai các bản sao lưu database (Snapshots) sạch (đã loại bỏ dữ liệu nhạy cảm) mỗi 12 giờ. Điều này giúp các nhà phát triển mới có thể tham gia dự án với dữ liệu thực tế ngay lập tức mà không cần cấu hình phức tạp.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Map-driven Storytelling:** Kỹ thuật nhóm các sự kiện quân sự thành các "Map Stories", cho phép người dùng xem lại diễn biến chiến sự dưới dạng một câu chuyện có trình tự không gian và thời gian.
*   **RAG (Retrieval-Augmented Generation):** Tích hợp AI chat trực tiếp trên dữ liệu tình báo. Kỹ thuật này sử dụng `pgvector` để tìm các đoạn tài liệu liên quan nhất đến câu hỏi của người dùng rồi đưa vào LLM để trả lời, đảm bảo tính chính xác và giảm thiểu "ảo giác" (hallucination).
*   **Automated Cross-Verification:** Tích hợp với X AI (Grok) để tự động kiểm tra xem một bài đăng trên mạng xã hội có thực sự tồn tại và nội dung có khớp với các báo cáo chính thống hay không.
*   **Strict Design Tokens:** Sử dụng hệ thống biến CSS (CSS Variables) thay vì mã hex trực tiếp. Điều này giúp duy trì tính nhất quán của giao diện tactical tối màu và hỗ trợ khả năng mở rộng sang các chế độ hiển thị khác (như Minimalist Mode).
*   **Admin Workflow Engine:** API admin không chỉ là các endpoint CRUD thông thường mà hoạt động như một cỗ máy quy trình (Workflow Engine), hướng dẫn Agent AI từng bước nhập liệu để tránh sai sót và đảm bảo dữ liệu luôn có đầy đủ các liên kết chéo.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng dữ liệu của Pharos AI có thể tóm gọn như sau:

1.  **Thu thập (Ingestion):** Agent (OpenClaw) quét 30+ nguồn tin (RSS, X, News).
2.  **Xử lý & Xác minh (Validation):** AI phân tích mức độ nghiêm trọng, xác minh nguồn tin qua X AI và gắn thẻ (tag) các đối tượng (actors) liên quan.
3.  **Lưu trữ (Fulfillment):** Dữ liệu được đẩy vào PostgreSQL qua Admin API. Tại đây, hệ thống tự động tạo ra các `DocumentEmbedding` để phục vụ tra cứu sau này.
4.  **Đóng gói (Snapshot):** Hệ thống định kỳ tạo snapshot database để đồng bộ hóa môi trường local và production.
5.  **Hiển thị (Visualization):** Người dùng truy cập Dashboard. Dữ liệu được tải qua React Query. Bản đồ render qua DeckGL để hiển thị các quỹ đạo tên lửa hoặc vùng kiểm soát một cách mượt mà.
6.  **Tương tác (Intelligence Retrieval):** Người dùng có thể hỏi Chatbot về các tình huống cụ thể (ví dụ: "Thiệt hại kinh tế của Iran sau đợt tấn công vừa qua là gì?"), AI sẽ truy xuất dữ liệu từ DB và trả lời dựa trên ngữ cảnh thực tế.

**Kết luận:** Pharos AI là một dự án OSINT được đầu tư kỹ lưỡng về mặt kỹ thuật, kết hợp xuất sắc giữa công nghệ bản đồ số (GIS) và trí tuệ nhân tạo (AI) để cung cấp cái nhìn toàn cảnh về các biến động địa chính trị.