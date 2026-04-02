Dựa trên mã nguồn và tài liệu của dự án **OpenNews**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của hệ thống này:

### 1. Công nghệ cốt lõi (Core Tech Stack)

OpenNews là một hệ thống phân tích dữ liệu tài chính phức tạp, kết hợp giữa AI hiện đại và cơ sở dữ liệu đồ thị:

*   **Ngôn ngữ & Điều phối:** **Python 3.13** là ngôn ngữ chính. Hệ thống sử dụng **LangGraph** để quản lý luồng xử lý (pipeline) dưới dạng một đồ thị có trạng thái (stateful graph).
*   **Xử lý ngôn ngữ tự nhiên (NLP):**
    *   **FinBERT:** Dùng để tạo word embeddings chuyên dụng cho lĩnh vực tài chính.
    *   **DeBERTa-v3:** Sử dụng kỹ thuật Zero-shot Classification để phân loại tin tức mà không cần gán nhãn trước.
    *   **BERT-base-NER:** Trích xuất thực thể (Named Entity Recognition).
    *   **LLM (GPT-4o-mini):** Dùng để tinh lọc chủ đề (topic refinement) và dịch thuật song ngữ.
*   **Cơ sở dữ liệu (Hybrid Storage):**
    *   **PostgreSQL:** Lưu trữ dữ liệu cấu trúc, nhật ký các lô (batches) và báo cáo Markdown.
    *   **Neo4j:** Xây dựng mạng lưới tri thức (Knowledge Graph), kết nối giữa Tin tức - Thực thể - Chủ đề.
    *   **Redis:** Lưu trữ bộ nhớ thời gian thực (temporal memory) trong vòng 30 ngày để tính toán xu hướng.
*   **Giao diện & Công cụ Web:**
    *   **Vue 3 + Vite + TypeScript:** Xây dựng Dashboard hiển thị chỉ số.
    *   **Playwright:** Sử dụng trình duyệt không đầu (Headless Chromium) để chụp ảnh Dashboard (PNG snapshot) phục vụ tính năng chia sẻ.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của OpenNews được thiết kế theo mô hình **Agentic Pipeline**:

*   **Thiết kế hướng tác nhân (Multi-agent):** Hệ thống chia nhỏ logic thành các tác nhân chuyên biệt như `ClassifierAgent` (Phân loại), `FeatureAgent` (Trích xuất đặc trưng), `MemoryAgent` (Quản lý bộ nhớ), và `ReportAgent` (Tạo báo cáo).
*   **Cơ chế GraphRAG:** Thay vì chỉ tìm kiếm văn bản đơn thuần, OpenNews trích xuất các quan hệ thực thể vào Neo4j. Khi cần đánh giá tác động, nó truy vấn các "tiểu đồ thị" (subgraph) để hiểu bối cảnh của các thực thể liên quan.
*   **Khả năng chịu lỗi và tính bền bỉ:** Sử dụng hệ thống `checkpoint` để ghi nhớ vị trí tin tức cuối cùng đã xử lý, tránh thu thập trùng lặp. Các nhãn chủ đề nếu dịch lỗi sẽ được đưa vào hàng đợi để `retry_labels` trong vòng lặp tiếp theo.

---

### 3. Kỹ thuật lập trình chính (Key Techniques)

*   **DK-CoT (Domain-Knowledge Chain-of-Thought):** Đây là kỹ thuật lõi trong `ReportAgent`. Hệ thống không chỉ đưa ra một con số ngẫu nhiên mà thực hiện suy luận theo chuỗi qua 4 chiều: Tương quan cổ phiếu, Tâm lý thị trường, Rủi ro chính sách và Độ lan tỏa truyền thông.
*   **Online Topic Clustering:** Sử dụng phân cấp cụm (Hierarchical Clustering) dựa trên độ tương đồng Cosine của các vector FinBERT, sau đó dùng LLM để "thẩm định" lại xem các tin tức có thực sự thuộc cùng một sự kiện hay không.
*   **NLI-based Scoring:** Kỹ thuật dùng mô hình suy luận ngôn ngữ (Natural Language Inference) để chấm điểm 7 chiều đặc trưng (như tính đột phá, tính gây tranh cãi) bằng cách kiểm tra xác suất của các giả thuyết (hypotheses).
*   **Server-side Rendered Snapshots:** Kỹ thuật kết hợp giữa logic backend Python và Playwright để render các component HTML/CSS phức tạp thành ảnh PNG chất lượng cao mà không cần phía client thực hiện.

---

### 4. Luồng hoạt động hệ thống (Operational Workflow)

Quy trình xử lý của OpenNews diễn ra tự động theo chu kỳ (mặc định 5 phút):

1.  **Ingestion (Tiếp nhận):** Thu thập tin tức từ NewsNow API và các file hạt giống (JSONL).
2.  **Preprocessing (Tiền xử lý):** Chuẩn hóa URL, loại bỏ trùng lặp và chuyển đổi văn bản thành vector embeddings.
3.  **NLP Analysis (Phân tích):**
    *   Trích xuất các thực thể chính (Công ty, nhân vật, địa danh).
    *   Phân loại tin tức vào các nhóm: macro, policy, company event...
    *   Tính toán 7 chiều đặc trưng để có điểm Impact Score sơ bộ.
4.  **Clustering & Refinement (Gom cụm):** Nhóm các tin tương đồng thành `Topic`. LLM tham gia để tách các tin "trông có vẻ giống nhưng không cùng sự kiện".
5.  **Memory & Trend (Bộ nhớ & Xu hướng):** Ghi dữ liệu vào Redis. So sánh với dữ liệu cũ trong cửa sổ 30 ngày để xác định xu hướng đang tăng hay giảm.
6.  **Knowledge Graph Update:** Đẩy toàn bộ quan hệ vào Neo4j, tạo liên kết `MENTIONS` giữa tin tức và thực thể, `IN_TOPIC` giữa tin tức và chủ đề.
7.  **Reporting (Báo cáo):** `ReportAgent` tổng hợp báo cáo Markdown cuối cùng và tính điểm DK-CoT.
8.  **Distribution (Phân phối):** Dashboard cập nhật dữ liệu mới qua API, sẵn sàng cho người dùng tra cứu hoặc xuất ảnh chia sẻ.

**Tổng kết:** OpenNews là một hệ thống mạnh mẽ cho phép biến dòng tin tức tài chính hỗn loạn thành một mạng lưới tri thức có thể định lượng, giúp các nhà đầu tư nhận diện nhanh các sự kiện có tác động lớn đến thị trường.