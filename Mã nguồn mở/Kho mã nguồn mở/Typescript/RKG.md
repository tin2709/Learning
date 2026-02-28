Dựa trên nội dung các tệp tin bạn cung cấp, dưới đây là phân tích chi tiết về **RKG (React Knowledge Graph)** — một hệ thống phân tích mã nguồn React mạnh mẽ, kết hợp giữa Graph Database và AI.

---

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

*   **Ngôn ngữ & Runtime:** TypeScript (Node.js >= 20), ESM mode.
*   **Phân tích mã nguồn (Static Analysis):** Sử dụng `ts-morph` (một wrapper quanh TypeScript Compiler API) để duyệt cây AST (Abstract Syntax Tree). Đây là phương pháp chính xác hơn nhiều so với Regex, cho phép hiểu sâu về Props, Hooks và quan hệ Export/Import.
*   **Cơ sở dữ liệu Đồ thị:** Neo4j 5. Đây là lựa chọn tối ưu để lưu trữ các quan hệ phụ thuộc (dependency tree) phức tạp và truy vấn đa tầng (transitive dependencies).
*   **AI & Xử lý ngôn ngữ tự nhiên:**
    *   **Embeddings:** Sử dụng thư viện `@xenova/transformers` để chạy mô hình `all-MiniLM-L6-v2` ngay tại local (không cần API key).
    *   **Vector Search:** Sử dụng tính năng Vector Index của Neo4j để tìm kiếm linh kiện (component) theo ý nghĩa ngữ nghĩa (semantic search).
*   **Giao thức kết nối AI:** MCP (Model Context Protocol). Đây là chuẩn mới giúp các trợ lý AI (như Claude) có thể gọi trực tiếp các công cụ của RKG để truy vấn code.

---

### 2. Tư duy Kiến trúc (Architectural Design)

RKG được thiết kế theo mô hình **Service-Oriented CLI**:

*   **Core Logic (`src/core/`):** Tách biệt các dịch vụ hạ tầng (`infra-service`), dịch vụ đánh chỉ mục (`index-service`) và dịch vụ kiểm tra sức khỏe (`health-service`).
*   **Data Ingestion Pipeline:**
    1.  `Parser`: Quét folder `src`, trích xuất metadata.
    2.  `Embedding`: Chuyển đổi thông tin component thành vector.
    3.  `Database`: Đẩy dữ liệu vào Neo4j theo các Nodes (`Component`, `Module`, `Prop`, `Hook`) và Edges (`DEPENDS_ON`, `HAS_PROP`, `USES_HOOK`).
*   **Atomic Design Classification:** Hệ thống tự động phân loại component vào các tầng: `atom`, `molecule`, `organism`, `template`, `page` dựa trên đường dẫn file hoặc số lượng phụ thuộc (fan-out).

---

### 3. Các Kỹ thuật Chính (Key Techniques)

#### a. Phân tích tác động (Impact Analysis)
RKG sử dụng các truy vấn Cypher đệ quy (`-[:DEPENDS_ON*1..]->`) để xác định "Blast Radius" (bán kính ảnh hưởng). Khi bạn thay đổi một Component ở tầng thấp (như `Button`), AI có thể trả lời chính xác những `Page` nào ở tầng cao nhất sẽ bị ảnh hưởng.

#### b. Tìm kiếm ngữ nghĩa & Tương đồng (Semantic & Similarity Search)
Thay vì chỉ tìm theo tên file, RKG mã hóa (embed) cả `description`, `props` và `hooks` của component. Kỹ thuật này giúp:
*   Tìm kiếm bằng ngôn ngữ tự nhiên: "Tìm component hiển thị danh sách người dùng có phân trang".
*   Gợi ý component tương tự để tránh viết code trùng lặp (DRY).

#### c. Native Graph Schema
Hệ thống không chỉ lưu component dưới dạng text mà biến các thuộc tính như **Hooks** và **Props** thành các Node riêng biệt trong đồ thị. Điều này cho phép thực hiện các truy vấn như: "Liệt kê tất cả các component sử dụng hook `useContext` và có prop tên là `theme`".

---

### 4. Luồng Hoạt động của Hệ thống (System Workflow)

1.  **Hạ tầng:** Người dùng chạy `rkg start`, Docker Compose sẽ dựng Neo4j.
2.  **Indexing:** `rkg index ./src` kích hoạt `ts-morph` để đọc toàn bộ dự án.
3.  **Semantic Processing:** Mỗi component được mô tả bằng văn bản và chuyển hóa thành Vector 384 chiều.
4.  **Graph Loading:** Dữ liệu được đẩy vào Neo4j bằng câu lệnh `UNWIND` (batch processing) để đạt hiệu suất cao.
5.  **AI Querying:** AI Assistant (thông qua MCP) gửi yêu cầu. RKG thực thi các câu lệnh Cypher tương ứng và trả về kết quả JSON đã được tối ưu hóa.

---

### 5. Đánh giá Thiết kế (System Evaluation)

*   **Ưu điểm:**
    *   **Local-first:** Không gửi mã nguồn ra ngoài, bảo mật tuyệt đối cho doanh nghiệp.
    *   **Extensibility:** Việc hỗ trợ `execute_cypher` (read-only) cho phép AI thực hiện những truy vấn tùy biến cực kỳ phức tạp mà lập trình viên chưa lường trước được.
    *   **Next.js Ready:** Có logic nhận diện đặc thù cho App Router (`page.tsx`, `layout.tsx`).

*   **Điểm cần lưu ý (Potential Challenges):**
    *   **Hiệu suất:** Với các dự án cực lớn (hàng chục nghìn file), việc tạo embedding tại local có thể tốn tài nguyên.
    *   **Độ chính xác của Parser:** Việc lọc các file không phải component (như helper, types) dựa trên regex và heuristics có thể có sai số (đã được đề cập trong `tasks/prd-parser-accuracy.md`).

**Tổng kết:** RKG là một công cụ hiện đại, giải quyết bài toán "mất phương hướng" trong các codebase React lớn bằng cách biến code thành một bản đồ tri thức sống động mà AI có thể đọc và hiểu được.