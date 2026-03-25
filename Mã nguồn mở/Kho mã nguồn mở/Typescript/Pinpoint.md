Dưới đây là phân tích chi tiết về dự án **Pinpoint** dựa trên mã nguồn và tài liệu bạn cung cấp.

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án là một sự kết hợp đa ngôn ngữ (Polyglot) giữa hệ sinh thái Python (xử lý dữ liệu) và Node.js (giao tiếp).

*   **Backend & Processing (Python):**
    *   **FastAPI:** Cung cấp RESTful API hiệu năng cao để kết nối bot với hệ thống tệp cục bộ.
    *   **SQLite + FTS5:** Sử dụng bộ máy tìm kiếm toàn văn (Full-Text Search) tích hợp sẵn của SQLite với thuật toán BM25 để xếp hạng kết quả mà không cần server Elasticsearch phức tạp.
    *   **Google Gemini (Pro/Flash/Embedding 2):** Đóng vai trò là "não bộ" cho việc hiểu ngôn ngữ tự nhiên, trích xuất sự thật (facts), nhận diện hình ảnh/video và gọi công cụ (tool calling).
    *   **InsightFace & ONNX:** Xử lý nhận diện khuôn mặt (Face Detection/Recognition) cục bộ.
    *   **Chonkie:** Một thư viện chunking hiện đại giúp chia nhỏ văn bản dựa trên cấu trúc để tìm kiếm ngữ nghĩa chính xác hơn.
    *   **Pandas:** Phân tích dữ liệu trong các tệp Excel/CSV ngay trên bộ nhớ.

*   **Bot Layer (Node.js):**
    *   **WhatsApp Web API:** Giao tiếp trực tiếp với người dùng qua WhatsApp.
    *   **Gemini SDK:** Quản lý vòng lặp hội thoại và thực thi chức năng (Function Calling).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Pinpoint theo triết lý **"Local-first, Cloud-augmented"** (Ưu tiên cục bộ, hỗ trợ bởi đám mây):

*   **Phân lớp lưu trữ (Layered Memory):** Hệ thống ghi nhớ qua 4 cấp độ: Hội thoại ngắn hạn (50 tin nhắn), Sự thật cá nhân dài hạn (SQLite), Facts trích xuất từ tài liệu, và Bộ nhớ khuôn mặt. Điều này giúp AI có "ngữ cảnh" sâu sắc về người dùng.
*   **Hệ thống Kỹ năng Modul (Skill-based System):** Thay vì nạp tất cả hướng dẫn vào một prompt khổng lồ, hệ thống chỉ nạp các tệp "skill" (markdown) liên quan dựa trên ý định của người dùng. Điều này tối ưu hóa token và độ chính xác.
*   **Pipeline tìm kiếm phân tầng (Tiered Search):** Luồng tìm kiếm đi từ: *Tìm chính xác (Strict)* $\rightarrow$ *Tìm mở rộng (Relaxed - đồng nghĩa)* $\rightarrow$ *Tìm diện rộng (Broad - OR)* $\rightarrow$ *Tìm kiếm ngữ nghĩa (Semantic)*. Cách tiếp cận này đảm bảo tốc độ cực nhanh cho các truy vấn đơn giản và độ thông minh cho các truy vấn phức tạp.
*   **Persistence Job Lifecycle:** Các tác vụ nặng (như index hàng ngàn ảnh) không chỉ chạy ngầm mà được lưu trạng thái vào DB. Nếu hệ thống restart, job có thể tiếp tục từ vị trí cũ.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Content-Addressable Storage (CAS):** Sử dụng SHA-256 để băm nội dung tệp. Nếu bạn có 2 tệp nội dung giống hệt nhau ở 2 thư mục khác nhau, Pinpoint chỉ lưu trữ văn bản và nhúng (embedding) một lần duy nhất để tiết kiệm không gian.
*   **BM25 & Reciprocal Rank Fusion (RRF):** Kết hợp kết quả từ tìm kiếm từ khóa truyền thống và tìm kiếm vector bằng thuật toán RRF để đưa ra kết quả cuối cùng chính xác nhất.
*   **Smart Normalization:** Khi tìm kiếm số điện thoại hoặc ID trong Excel, hệ thống tự động chuẩn hóa (ví dụ: loại bỏ dấu gạch ngang) để so sánh chuỗi thô, giúp tìm thấy dữ liệu ngay cả khi định dạng không khớp.
*   **Circuit Breaker & Budgeting:** Hệ thống có bộ ngắt mạch chi phí ($0.10 mỗi tin nhắn) để ngăn chặn việc gọi API AI quá đà gây tốn kém.
*   **Query Expansion:** Sử dụng LLM để mở rộng một từ khóa đơn giản thành nhiều biến thể từ khóa khác nhau trước khi truy vấn vào SQLite FTS5.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Giai đoạn Indexing (Nạp dữ liệu):**
    *   Quét thư mục $\rightarrow$ Trích xuất văn bản (PDF/Office/Image OCR) $\rightarrow$ Chia nhỏ (Chunking) $\rightarrow$ Tạo vector nhúng (Embedding) $\rightarrow$ Lưu vào SQLite.
    *   Đồng thời, Gemini trích xuất các "Facts" quan trọng (tên khách hàng, số tiền, ngày hạn) để tìm kiếm nhanh sau này.

2.  **Giai đoạn Xử lý truy vấn (Querying):**
    *   Người dùng nhắn tin qua WhatsApp.
    *   Bot Node.js nhận tin nhắn $\rightarrow$ Gửi đến Gemini để phân tích ý định (Intent).
    *   Gemini quyết định gọi một "Tool" (ví dụ: `search_documents`).
    *   FastAPI nhận yêu cầu $\rightarrow$ Thực thi tìm kiếm đa tầng trong SQLite $\rightarrow$ Trả về đoạn văn bản (snippets) kèm đường dẫn tệp.

3.  **Giai đoạn Phản hồi:**
    *   Gemini tổng hợp thông tin từ các snippets trả về để trả lời người dùng bằng ngôn ngữ tự nhiên.
    *   Nếu người dùng yêu cầu, hệ thống đẩy tệp tin vào hàng đợi `outgoing_file_queue` để Bot WhatsApp tải tệp từ PC và gửi trực tiếp cho điện thoại.

### Tổng kết
Pinpoint không chỉ là một ứng dụng tìm kiếm, nó là một **Hệ điều hành tệp cá nhân qua tin nhắn**. Sự kết hợp giữa tốc độ của SQLite cục bộ và khả năng hiểu của Gemini tạo ra một trải nghiệm "hỏi gì đáp nấy" trên chính dữ liệu riêng tư của người dùng.