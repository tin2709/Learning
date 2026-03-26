Dựa trên mã nguồn và tài liệu của dự án **DevLens (Open Source)**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và quy trình vận hành của hệ thống:

### 1. Công nghệ cốt lõi (Core Tech Stack)

DevLens sử dụng những công nghệ hiện đại nhất trong hệ sinh thái JavaScript/TypeScript để tối ưu hóa hiệu suất phân tích mã nguồn cục bộ:

*   **Runtime:** **Bun** được chọn làm nền tảng thực thi chính thay cho Node.js. Bun cung cấp tốc độ vượt trội cho các tác vụ đọc/ghi file và xử lý dữ liệu lớn (như quét hàng nghìn file mã nguồn).
*   **Phân tích mã nguồn (AST Engine):** **ts-morph** (một wrapper mạnh mẽ quanh TypeScript Compiler API). Đây là "trái tim" của hệ thống, cho phép DevLens "hiểu" cấu trúc code, trích xuất Component, Hook, và Function mà không cần thực thi code đó (Static Analysis).
*   **Giao diện người dùng (Frontend):** **Next.js 15 (App Router)**. Sử dụng các tính năng mới nhất của React để tạo ra một Dashboard mượt mà.
*   **Thư viện đồ thị:** **Cytoscape.js**. Đây là thư viện tiêu chuẩn để vẽ các mạng lưới liên kết phức tạp, hỗ trợ các thuật toán sắp xếp (layout) như force-directed để tự động dàn trải các node code.
*   **Trí tuệ nhân tạo (AI):** Hỗ trợ đa nền tảng (Multi-provider) bao gồm **Anthropic, OpenAI, Gemini, OpenRouter** và đặc biệt là **Ollama** (cho phép chạy mô hình `qwen2.5-coder` hoàn toàn cục bộ để bảo mật mã nguồn).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của DevLens được thiết kế theo mô hình **Pipeline-based Analysis** (Phân tích theo đường ống):

*   **Tách biệt Engine và Frontend:** 
    *   `engine/`: Chịu trách nhiệm "nặng" như quét filesystem, phân tích AST, tính toán điểm số và gọi LLM.
    *   `frontend/`: Tập trung vào việc hiển thị đồ thị và tương tác người dùng.
*   **Hệ thống phân tầng đồ thị (Hierarchical Graph):** DevLens không chỉ tìm các liên kết `import`, nó xây dựng 10 loại cạnh (edge types) khác nhau như `PROP_PASS` (truyền prop), `READS_FROM` (đọc state), giúp mô hình hóa logic ứng dụng một cách chân thực nhất.
*   **Đánh giá tầm quan trọng (Importance Scoring):** Sử dụng thuật toán đa tầng (không dùng AI) dựa trên độ phức tạp, số lượng liên kết vào/ra (fan-in/fan-out) để xác định đâu là các file "xương sống" của dự án.
*   **Tóm tắt theo thứ tự liên kết (Topological Summarization):** Đây là tư duy cực kỳ thông minh. Hệ thống tóm tắt các node "lá" (không phụ thuộc ai) trước, sau đó dùng tóm tắt đó làm ngữ cảnh (context) để tóm tắt các node cha. Điều này giúp AI hiểu được luồng logic từ thấp lên cao.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Xử lý bất đồng bộ và Streaming (SSE):** Sử dụng **Server-Sent Events (SSE)** để cập nhật tiến độ phân tích và tóm tắt AI theo thời gian thực từ Engine lên Frontend, giúp người dùng không phải chờ đợi trong vô vọng khi xử lý repo lớn.
*   **MapReduce cho file lớn:** Trong `engine/src/summarizer/mapreduce.ts`, nếu một file quá lớn (>1,200 tokens), hệ thống sẽ tự động chia nhỏ file, tóm tắt từng phần (Map) và sau đó tổng hợp lại thành một bản tóm tắt cuối cùng (Reduce).
*   **Thiết kế Checkpoint & Resume:** Vì quá trình gọi AI tóm tắt hàng nghìn node có thể mất nhiều thời gian hoặc bị lỗi mạng, DevLens lưu trạng thái (checkpoint) vào tệp JSON. Nếu hệ thống sập, nó có thể tiếp tục từ vị trí cũ mà không phải bắt đầu lại từ đầu.
*   **Deduplication & Hash Matching:** Mỗi node được gắn một `codeHash`. Khi phân tích lại một commit mới, DevLens so sánh hash này; nếu code không đổi, nó sẽ tái sử dụng bản tóm tắt cũ, giúp tiết kiệm 90% chi phí và thời gian.

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng xử lý đi qua các bước được định nghĩa trong `engine/src/pipeline/index.ts`:

1.  **Fingerprint:** Quét thư mục gốc để nhận diện framework (Next.js, Express...), quản lý state (Redux, Zustand...) và các thư viện đang dùng.
2.  **Filesystem Scan:** Phát hiện cấu trúc định tuyến (routes) của Next.js (App/Pages router) hoặc các endpoint của backend (Express/Fastify).
3.  **AST Parse:** `ts-morph` duyệt qua cây cú pháp của từng file, bóc tách các hàm, component và biến thể.
4.  **Edge Detection:** 10 bộ dò chạy song song để tìm các mối quan hệ: "Ai gọi ai?", "Ai truyền prop cho ai?", "Ai đang đọc store nào?".
5.  **Scoring:** Tính toán trọng số kiến trúc để phân loại các node quan trọng (màu sắc/kích thước trên đồ thị sẽ khác nhau).
6.  **AI Summarization:** Gọi LLM theo thứ tự cấu trúc (topological) để tạo ra hai loại tóm tắt: **Business Summary** (dành cho PM/Non-tech) và **Technical Summary** (dành cho Developer).
7.  **Visualization:** Toàn bộ dữ liệu (Nodes + Edges + Summaries) được gửi về Frontend để render lên canvas của Cytoscape.

### Tổng kết
DevLens là một công cụ **Code Intelligence** mạnh mẽ, biến mã nguồn trừu tượng thành một bản đồ tương tác có thể "đọc" được. Nó kết hợp nhuần nhuyễn giữa **Static Analysis** (độ chính xác cao) và **AI Summarization** (hiểu ngữ nghĩa), giúp giải quyết bài toán "quá tải thông tin" trong các dự án phần mềm hiện đại.