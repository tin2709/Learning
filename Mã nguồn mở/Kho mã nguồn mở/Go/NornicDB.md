Dựa trên các tệp tin bạn cung cấp, dưới đây là phân tích chuyên sâu về **NornicDB** — một hệ quản trị cơ sở dữ liệu đồ thị (Graph Database) hiện đại, tập trung vào hiệu suất và khả năng tích hợp AI.

---

### 1. Công nghệ cốt lõi (Core Technology)

NornicDB không chỉ là một bản sao của Neo4j mà là một sự kết hợp đa mô hình (Multi-model):

*   **Lưu trữ cơ bản (Storage Engine):** Sử dụng **BadgerDB (v4)** — một Key-Value store hiệu năng cao viết bằng Go, tối ưu cho SSD. NornicDB xây dựng cấu trúc đồ thị bên trên lớp này.
*   **Ngôn ngữ Truy vấn & Giao thức:** 
    *   Hỗ trợ hoàn toàn **Cypher** (thông qua bộ parser **ANTLR4** cho độ chính xác cao và một bộ parser nội bộ `nornic` cho tốc độ).
    *   Sử dụng giao thức **Bolt** (tương thích Neo4j) và **gRPC** (tương thích Qdrant).
*   **Xử lý Vector & Embeddings:**
    *   Tích hợp sẵn **llama.cpp** để chạy các mô hình nhúng (Embeddings) cục bộ dưới định dạng **GGUF** (như BGE-M3).
    *   Chỉ mục vector sử dụng thuật toán **HNSW** (Hierarchical Navigable Small World) được tối ưu hóa bằng SIMD (Neon/AVX).
*   **Tăng tốc phần cứng (Hardware Acceleration):** 
    *   Hỗ trợ đa nền tảng GPU thông qua **Metal** (Apple Silicon), **CUDA** (Nvidia), và **Vulkan** (AMD/Intel/Cross-platform).
*   **Hệ thống AI Assistant (Heimdall):** Sử dụng mô hình LLM nhỏ (như Qwen 0.5B/1.5B) để thực hiện các tác vụ tự trị (autonomous actions) và truy vấn bằng ngôn ngữ tự nhiên.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của NornicDB thể hiện tư duy "AI-Native" và "Efficiency-First":

*   **Hybrid Execution Model:** Hệ thống có hai luồng thực thi: *Streaming fast paths* cho các mẫu truy vấn phổ biến (traversal/aggregation) để đạt tốc độ tối đa, và *General engine* cho các truy vấn Cypher phức tạp.
*   **Cognitive Memory Architecture (Kiến trúc bộ nhớ nhận thức):** NornicDB mô phỏng bộ nhớ con người với cơ chế **Memory Decay** (suy giảm trí nhớ). Dữ liệu được chia thành:
    *   *Episodic (7 ngày):* Ngữ cảnh chat, phiên làm việc.
    *   *Semantic (69 ngày):* Sự thật, quyết định.
    *   *Procedural (693 ngày):* Kỹ năng, khuôn mẫu.
*   **Canonical Graph Ledger:** Một hướng tiếp cận mang tính "Ledger" (sổ cái). Mọi thay đổi dữ liệu đều có version, timestamp (temporal validity) và receipt (biên lai) để kiểm toán. Điều này cực kỳ quan trọng cho AI Governance và Compliance.
*   **Plug-and-Play Compatibility:** Thiết kế theo kiểu "thả vào là chạy" (Drop-in replacement). Nó tương thích với hệ sinh thái của cả Neo4j (Graph) và Qdrant (Vector), giúp giảm chi phí chuyển đổi.

---

### 3. Kỹ thuật Lập trình (Programming Techniques)

NornicDB áp dụng các tiêu chuẩn lập trình Go khắt khe (theo tài liệu `AGENTS.md`):

*   **Functional Go & Dependency Injection (DI):** Sử dụng Function Types để inject các implementation khác nhau (ví dụ: đổi từ GPU Embedder sang Mock Embedder trong testing).
*   **Tối ưu hóa Hot-path:** 
    *   Sử dụng **SIMD** để tăng tốc tính toán vector.
    *   Kỹ thuật **BM25-seeded HNSW construction**: Sử dụng độ tương đồng lexical (văn bản) để sắp xếp thứ tự chèn vào chỉ mục HNSW, giúp giảm 2.7 lần thời gian xây dựng chỉ mục.
*   **Tách biệt trách nhiệm (Separation of Concerns):** Hệ thống chia làm 4 lớp rõ rệt: API Layer (Protocols) -> Query Layer (Cypher/Parser) -> Storage Layer (Badger) -> Infrastructure (Cache/Pool).
*   **Quy tắc 2500 dòng:** Giới hạn kích thước tệp để đảm bảo tính dễ đọc và bảo trì, ép buộc lập trình viên phải module hóa mã nguồn.
*   **Test-Driven Development (TDD):** Yêu cầu độ bao phủ mã (Coverage) tối thiểu 90%. Mọi lỗi (bug) đều phải bắt đầu bằng một bài test tái hiện lỗi trước khi được sửa.

---

### 4. Luồng hoạt động hệ thống (System Operation Flow)

1.  **Tiếp nhận (Ingestion):** Dữ liệu vào qua Bolt, HTTP hoặc gRPC.
2.  **Phân tích & Tối ưu (Parsing & Optimization):**
    *   Query được phân tích qua bộ parser nornic (nhanh) hoặc ANTLR (chính xác).
    *   Hệ thống quyết định sử dụng Executor chuyên biệt hay Executor tổng quát.
3.  **Xử lý Vector (Vector Pipeline):**
    *   Nếu node có nội dung văn bản, **Embedding Worker** (chạy nền) sẽ gọi llama.cpp (GPU accelerated) để tạo vector.
    *   Vector được chèn vào chỉ mục HNSW.
4.  **Tự động hóa liên kết (Auto-Relationships):**
    *   Dựa trên độ tương đồng nhúng (Embedding Similarity) hoặc tần suất truy vấn đồng thời (Co-access), hệ thống tự động tạo các quan hệ `:RELATED_TO` giữa các Node.
5.  **Truy vấn & Phục hồi (Retrieval):**
    *   Hỗ trợ **Hybrid Search**: Kết hợp tìm kiếm vector (semantic) + BM25 (keyword) + Graph Traversal (quan hệ).
    *   Bộ lọc **Decay Score** có thể loại bỏ các "ký ức" đã cũ hoặc không còn quan trọng.
6.  **Ghi nhật ký & Kiểm toán:** Mọi thay đổi được ghi vào WAL (Write-Ahead Log) và có thể truy vấn lại trạng thái dữ liệu "tại một thời điểm" (as-of reads).

### Tóm lại
**NornicDB** là một nỗ lực hiện đại hóa Graph Database. Nó không chỉ lưu trữ dữ liệu mà còn chủ động "học" các mối liên kết thông qua Embeddings và quản lý vòng đời dữ liệu bằng cơ chế suy giảm (decay), biến cơ sở dữ liệu thành một hệ thống "trí nhớ" thực thụ cho AI Agents.