Khoj AI là một dự án mã nguồn mở đầy tham vọng, hướng tới việc xây dựng một "bộ não thứ hai" (AI Second Brain). Qua việc phân tích cấu trúc mã nguồn và các file cấu hình, dưới đây là phân tích chi tiết về dự án này:

---

### 1. Phân tích Công nghệ Cốt lõi (Core Technology)

Khoj kết hợp những công nghệ hiện đại nhất trong lĩnh vực AI và xử lý dữ liệu:

*   **Hybrid RAG (Retrieval-Augmented Generation):**
    *   Khoj không chỉ dựa vào kiến thức sẵn có của LLM. Nó sử dụng kỹ thuật **Bi-Encoder** (như `gte-small`) để tạo vector embeddings cho dữ liệu người dùng và **Cross-Encoder** (như `mxbai-rerank`) để tái xếp hạng (re-rank) kết quả, tối ưu độ chính xác khi tìm kiếm ngữ nghĩa.
    *   Hỗ trợ đa dạng nguồn dữ liệu: PDF, Markdown, Org-mode, Microsoft Word, Notion và GitHub.
*   **Vector Database:** Sử dụng **PostgreSQL với extension `pgvector`**. Đây là lựa chọn thông minh vì nó tận dụng được sự mạnh mẽ, tin cậy của SQL truyền thống đồng thời cho phép truy vấn vector hiệu quả cao.
*   **Đa mô hình (Model Agnostic):** Hệ thống được thiết kế để hoạt động với cả mô hình Cloud (OpenAI, Anthropic, Gemini) và mô hình cục bộ (Local LLMs qua Ollama, Llama.cpp).
*   **Computer/Browser Use (Experimental):** Một tính năng cực kỳ cao cấp cho phép AI tự vận hành trình duyệt hoặc máy tính (thông qua Playwright và Docker) để thực hiện các tác vụ phức tạp (như đặt vé, nghiên cứu sâu).

### 2. Tư duy Kiến trúc (Architectural Mindset)

Kiến trúc của Khoj thể hiện sự kết hợp khéo léo giữa tính ổn định và hiệu suất:

*   **Kiến trúc "Hybrid" Backend:**
    *   **Django:** Được dùng làm nền tảng quản lý: ORM (Giao tiếp DB), Admin Panel (Quản trị hệ thống), và Authentication. Tư duy ở đây là: "Đừng phát minh lại cái bánh xe" cho các tính năng quản lý người dùng phức tạp.
    *   **FastAPI:** Được dùng để xử lý các luồng tính năng AI và streaming. FastAPI cực kỳ nhanh, hỗ trợ Async tốt, rất phù hợp cho việc stream phản hồi từ LLM (SSE) và Websockets.
*   **Thiết kế Pluggable (Dễ mở rộng):** Các bộ xử lý nội dung (Processors) được tách riêng (PDF, Docx, Markdown...). Khi muốn thêm một định dạng file mới, nhà phát triển chỉ cần kế thừa lớp base `TextToEntries`.
*   **Hệ sinh thái Đa Client:** Khoj duy trì một Backend duy nhất phục vụ cho nhiều Frontend khác nhau: Web (Next.js), Desktop (Electron), Mobile (TWA), và cả các plugin cho công cụ năng suất như Emacs và Obsidian.

### 3. Kỹ thuật Lập trình Đặc sắc

*   **Quản lý ngữ cảnh (Context Management):**
    *   Khoj có thuật toán tự động **trộn và cắt (truncate)** tin nhắn dựa trên `max_prompt_size` của từng mô hình. Nó tính toán token (dùng `tiktoken` cho OpenAI hoặc `AutoTokenizer` cho HuggingFace) để đảm bảo không bao giờ vượt quá giới hạn của LLM mà vẫn giữ được thông tin quan trọng nhất.
*   **Xử lý đồng thời và Khóa tiến trình (Locking Mechanism):**
    *   Sử dụng `ProcessLock` trong DB để đảm bảo trong môi trường phân tán (như nhiều Docker container), chỉ có một worker được quyền chạy các tác vụ đặc quyền như `Index Content` hoặc `Schedule Leader`.
*   **Kỹ thuật "Thinking" của Agent:**
    *   Hệ thống hỗ trợ luồng "Train of Thought" (Chuỗi tư duy). Trước khi trả lời, AI được yêu cầu thực hiện các bước suy luận ngầm, giúp người dùng hiểu tại sao AI lại đưa ra câu trả lời đó (Transparency).
*   **Sử dụng Bun trong Build Web:** Dự án chuyển sang dùng `Bun` thay cho `Yarn/NPM` giúp tốc độ cài đặt và build Next.js nhanh hơn đáng kể trong CI/CD.

### 4. Luồng Hoạt động Hệ thống (System Flow)

#### A. Luồng Ingestion (Nạp dữ liệu):
1.  **Client** gửi file/URL lên Backend.
2.  **Processor** nhận diện định dạng (dùng `Magika` - thư viện nhận diện file bằng AI của Google).
3.  **Parser** trích xuất văn bản thô, xử lý line number (để sau này AI có thể trích dẫn chính xác dòng nào trong file).
4.  **Embeddings Model** chuyển đổi văn bản thành các vector số.
5.  **Database** lưu trữ văn bản thô vào `FileObject` và vector vào bảng `Entry`.

#### B. Luồng Truy vấn (Chat/Search):
1.  **User Input** đi qua bộ lọc **Intent Detection** để xác định người dùng muốn: tìm file, vẽ hình, chạy code hay tìm web.
2.  **Query Expansion:** AI tạo ra các câu truy vấn phụ (sub-queries) để tìm kiếm hiệu quả hơn.
3.  **Retrieval:** Hệ thống tìm trong `pgvector` các đoạn văn bản liên quan nhất.
4.  **Re-ranking:** Cross-encoder tính toán lại độ liên quan thực sự của các đoạn văn bản với câu hỏi.
5.  **Generation:** Gửi "Prompt = Ngữ cảnh tìm được + Lịch sử chat + Câu hỏi" cho LLM.
6.  **Streaming:** Kết quả được trả về client theo thời gian thực.

#### C. Luồng Research Mode (Nghiên cứu sâu):
Đây là luồng lặp (Loop): **Phân tích câu hỏi -> Chọn công cụ (Search/Code/Web) -> Chạy công cụ -> Tóm tắt kết quả -> Kiểm tra xem đã đủ thông tin chưa?** Nếu chưa đủ, hệ thống sẽ lặp lại cho đến khi đạt kết quả tối ưu hoặc chạm giới hạn (max iterations).

### Kết luận
Khoj AI không đơn thuần là một chatbot; nó là một **hệ thống quản lý tri thức cá nhân (PKM)** dựa trên AI. Dự án thể hiện kỹ năng kỹ thuật rất cao trong việc tích hợp các hệ thống phân tán, xử lý dữ liệu quy mô lớn và tối ưu hóa trải nghiệm người dùng trên nhiều nền tảng. Đây là một blueprint mẫu mực cho những ai muốn xây dựng ứng dụng AI RAG thực thụ cho doanh nghiệp hoặc cá nhân.