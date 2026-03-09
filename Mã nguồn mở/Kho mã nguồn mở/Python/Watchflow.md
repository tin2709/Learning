Dựa trên cấu trúc thư mục và nội dung các tệp tin của dự án **Watchflow**, dưới đây là phân tích chi tiết về hệ thống quản trị GitHub dựa trên AI (Agentic GitHub Guardrails) này:

---

### 1. Phân tích Công nghệ cốt lõi (Core Technology Stack)

Watchflow là một ứng dụng Python hiện đại, tận dụng sức mạnh của AI Agent để thực thi các chính sách quản trị (governance) trên GitHub:

*   **Ngôn ngữ & Framework:** Python 3.12, **FastAPI** (hiệu năng cao, hỗ trợ async hoàn toàn).
*   **AI Orchestration (Điều phối AI):** **LangGraph** và **LangChain**. Dự án sử dụng LangGraph để xây dựng các quy trình làm việc (workflows) có trạng thái (stateful), cho phép các AI Agent thực hiện nhiều bước suy luận (như phân tích quy tắc, chọn chiến lược kiểm tra).
*   **LLM Providers:** Hỗ trợ đa nền tảng qua **OpenAI** (mặc định gpt-4.1-mini), **AWS Bedrock** (Claude), và **Google Vertex AI**.
*   **Quản lý gói:** **uv** - Công cụ quản lý gói và môi trường ảo cực nhanh của Astral.
*   **Tương tác GitHub:** Sử dụng cả **REST API** và **GraphQL** của GitHub để lấy dữ liệu chi tiết (files, reviews, CODEOWNERS) và cập nhật Check Runs.
*   **Hạ tầng:** Docker, Docker Compose, và **Helm Chart** để triển khai lên Kubernetes (AWS EKS).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Watchflow được thiết kế theo hướng **"Hybrid Governance"** (Quản trị lai) và **"Agent-based"** (Dựa trên tác tử):

*   **Hybrid Evaluation (Kiểm tra lai):** Đây là điểm sáng nhất. Thay vì đẩy mọi thứ cho AI (tốn kém và chậm), hệ thống ưu tiên các **Validators** (mã nguồn thuần) cho các quy tắc đơn giản (như số lượng dòng code, patterns tiêu đề). AI chỉ được sử dụng làm "fallback" hoặc cho các quy tắc đòi hỏi hiểu biết về ngữ nghĩa (như kiểm tra mô tả PR có khớp với code thay đổi hay không).
*   **Governance as Code (Quản trị dưới dạng mã):** Mọi quy tắc nằm trong tệp `.watchflow/rules.yaml`. Điều này giúp các quy tắc có thể được phiên bản hóa và kiểm soát bằng chính quy trình PR của repo.
*   **Kiến trúc Đa tác tử (Multi-agent):**
    *   **Rule Engine Agent:** Đánh giá PR/Push đối với các quy tắc.
    *   **Feasibility Agent:** Kiểm tra xem một quy tắc viết bằng ngôn ngữ tự nhiên có khả thi để triển khai hay không.
    *   **Repository Analysis Agent:** Phân tích lịch sử repo để gợi ý các quy tắc phù hợp.
    *   **Acknowledgment Agent:** Hiểu các bình luận "ack" của lập trình viên để tạm bỏ qua vi phạm.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Webhook Deduplication & Task Queue:** Sử dụng `X-GitHub-Delivery` ID để đảm bảo không xử lý trùng lặp một sự kiện từ GitHub, đồng thời sử dụng hàng đợi tác vụ (Task Queue) nội bộ để xử lý bất đồng bộ, tránh làm nghẽn Webhook response.
*   **Pull Request Enrichment:** Trước khi đánh giá quy tắc, một bước "Enricher" sẽ lấy toàn bộ ngữ cảnh cần thiết (nội dung file, kết quả review, tệp CODEOWNERS) thông qua GraphQL để Agent có cái nhìn toàn diện mà không cần clone code về local.
*   **Structured Output:** Sử dụng tính năng `with_structured_output` của LLM kết hợp với **Pydantic** để đảm bảo phản hồi từ AI luôn tuân thủ đúng định dạng JSON/Object, tránh lỗi phân tích văn bản tự do.
*   **Glob-to-Regex Conversion:** Tự động chuyển đổi các mẫu đường dẫn file (globs) sang Regex để kiểm tra các quy tắc liên quan đến tệp tin một cách chính xác.
*   **AI Immune System:** Một kỹ thuật đo lường "vệ sinh" mã nguồn thông qua các chỉ số như: tỷ lệ PR không có issue liên kết, tỷ lệ code do AI tạo ra (heuristic-based), tỷ lệ bỏ qua CODEOWNERS.

---

### 4. Tóm tắt luồng hoạt động (Operation Flow)

1.  **Tiếp nhận (Ingestion):**
    *   GitHub gửi Webhook (PR opened, Push, v.v.) tới `/webhooks/github`.
    *   Hệ thống xác thực chữ ký (HMAC) và đưa vào hàng đợi `task_queue`.

2.  **Làm giàu dữ liệu (Enrichment):**
    *   `PullRequestProcessor` tải tệp cấu hình `.watchflow/rules.yaml` từ nhánh mặc định.
    *   `PullRequestEnricher` gọi GitHub API để lấy danh sách file thay đổi, diff, và các review hiện tại.

3.  **Suy luận & Đánh giá (Reasoning - LangGraph):**
    *   **Analyze:** Phân tích các quy tắc đang hoạt động.
    *   **Select Strategy:** AI quyết định quy tắc nào dùng "Validator" (nhanh), quy tắc nào cần "LLM Reasoning" (thông minh).
    *   **Execute:** Thực thi kiểm tra. Nếu vi phạm, AI sẽ tạo ra thông điệp "How to fix" (Làm sao để sửa) cực kỳ chi tiết.

4.  **Phản hồi (Reporting):**
    *   Cập nhật trạng thái **Check Run** trên GitHub (Success/Failed/Neutral).
    *   Nếu có vi phạm, hệ thống tự động đăng một bình luận (Comment) vào PR với định dạng thu gọn (collapsible), hướng dẫn lập trình viên cách khắc phục.

5.  **Tương tác (Interaction):**
    *   Lập trình viên có thể bình luận `@watchflow acknowledge "lý do"`.
    *   `AcknowledgmentAgent` sẽ phân tích lý do, nếu hợp lý sẽ cập nhật Check Run thành trạng thái Neutral để cho phép merge code.

### Kết luận
Watchflow là một hệ thống tinh vi kết hợp giữa tính **deterministic** (xác định) của lập trình truyền thống và tính **heuristic** (trực giác) của AI. Nó không thay thế các công cụ CI/CD hiện có mà đóng vai trò như một **"Hệ thống miễn dịch"** cấp cao, đảm bảo các tiêu chuẩn về quy trình và bảo mật luôn được thực thi một cách thông minh.