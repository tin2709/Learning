Dưới đây là bản phân tích chuyên sâu về công nghệ, kiến trúc và tóm tắt luồng hoạt động của dự án **Keep** (keephq/keep) dưới dạng file README tiếng Việt.

---

# Phân tích Dự án Keep - Nền tảng AIOps & Alert Management Open-source

## 1. Công nghệ cốt lõi (Core Tech Stack)

Dự án được xây dựng trên mô hình Modern Fullstack với hiệu năng cao và khả năng mở rộng tốt:

*   **Backend:**
    *   **Python 3.11+:** Ngôn ngữ chính, tận dụng hệ sinh thái thư viện xử lý dữ liệu và AI phong phú.
    *   **FastAPI:** Framework web hiện đại, hỗ trợ asynchronous (async/await), giúp xử lý hàng nghìn request alert đồng thời với độ trễ thấp.
    *   **SQLModel (SQLAlchemy + Pydantic):** Sử dụng để tương tác cơ sở dữ liệu và validate dữ liệu chặt chẽ.
    *   **Common Expression Language (CEL):** Một kỹ thuật quan trọng của Google được Keep tích hợp để thực hiện các bộ lọc (filter) alert phức tạp một cách an toàn và hiệu năng cao mà không cần thực thi code Python trực tiếp.
    *   **Arq (Redis-based):** Thư viện xử lý tác vụ nền (background tasks) mạnh mẽ, dùng để chạy các Workflow và xử lý alert không đồng bộ.

*   **Frontend:**
    *   **Next.js 15 (App Router):** Framework React hiện đại nhất, hỗ trợ Server Components và tối ưu hóa hiệu năng.
    *   **TypeScript:** Đảm bảo an toàn kiểu dữ liệu trên toàn bộ giao diện.
    *   **Tailwind CSS & Tremor:** Thư viện UI chuyên dụng cho Dashboard và Visualization.
    *   **React Flow:** Thư viện dùng để xây dựng trình kéo-thả quy trình (Workflow Builder).

*   **Infrastructure & Storage:**
    *   **Database:** Hỗ trợ linh hoạt SQLite (dev), PostgreSQL/MySQL (production).
    *   **Elasticsearch (Tùy chọn):** Dùng làm storage cho các hệ thống có quy mô hàng triệu alert/ngày.
    *   **Soketi (Pusher protocol):** Webhook server để cập nhật trạng thái alert thời gian thực (real-time) lên UI.

---

## 2. Tư duy Kiến trúc & Kỹ thuật chính

Keep không chỉ là một công cụ hiển thị alert, mà là một lớp "Orchestration" (điều phối) nằm trên các công cụ giám sát hiện có.

### Kiến trúc "Provider" (Abstraction Layer)
Keep thiết kế một lớp trừu tượng cho hơn 100 tích hợp (Datadog, Prometheus, Slack, Jira, v.v.). Mỗi tích hợp là một module riêng biệt kế thừa từ `BaseProvider`. Điều này giúp việc mở rộng thêm các công cụ mới cực kỳ dễ dàng mà không ảnh hưởng đến core logic.

### Tư duy "GitHub Actions cho Monitoring"
Đây là điểm sáng nhất của Keep. Thay vì chỉ nhận alert, Keep cho phép định nghĩa các **Workflow** bằng file YAML. Bạn có thể:
*   **Triggers:** Kích hoạt khi có alert, incident hoặc theo lịch.
*   **Steps:** Lấy thêm dữ liệu (ví dụ: query logs từ Cloudwatch hoặc kiểm tra trạng thái DB).
*   **Actions:** Tự động hóa (ví dụ: restart server qua SSH, tạo ticket Jira, gửi thông báo Slack).

### Cơ chế Chống nhiễu (Deduplication & Correlation)
*   **Fingerprinting:** Mỗi alert khi vào Keep được băm (hash) dựa trên các trường dữ liệu cụ thể để tránh lặp lại.
*   **Deduplication Rules:** Cho phép gom nhóm các alert giống nhau trong một khoảng thời gian.
*   **AI Correlation:** Sử dụng LLM (OpenAI, DeepSeek, v.v.) để phân tích ngữ nghĩa và gom nhóm các alert lẻ tẻ thành một **Incident** duy nhất, giúp giảm mệt mỏi cho kỹ sư On-call.

### Quản lý bí mật (Secret Management)
Kiến trúc hỗ trợ đa dạng từ file cục bộ đến các dịch vụ Cloud như AWS Secret Manager, GCP Secret Manager, Vault hoặc Kubernetes Secrets, đảm bảo an toàn cho các API Key của tổ chức.

---

## 3. Tóm tắt luồng hoạt động dự án (Workflow Summary)

Luồng đi của một Alert trong hệ thống Keep diễn ra như sau:

1.  **Ingestion (Tiếp nhận):** Alert được đẩy vào Keep thông qua Webhook hoặc Keep chủ động đi quét (polling) từ các Provider (như Grafana, Sentry, Cloudwatch).
2.  **Standardization (Chuẩn hóa):** Dữ liệu thô từ các nguồn khác nhau được chuyển đổi về một định dạng Alert chung duy nhất của Keep.
3.  **Enrichment (Làm giàu dữ liệu):** Keep áp dụng các quy tắc trích xuất (Extraction) hoặc ánh xạ (Mapping) để bổ sung thông tin (ví dụ: mapping tên service từ tag, lấy thêm thông tin chủ sở hữu từ metadata).
4.  **Deduplication & Suppression (Khử trùng & Nén):** Hệ thống kiểm tra xem alert này đã tồn tại chưa. Nếu đang trong "Maintenance Window" (cửa sổ bảo trì), alert sẽ bị nén lại để không gây nhiễu.
5.  **Correlation (Tương quan):** Các thuật toán (Rule-based hoặc AI) sẽ xem xét alert này có thuộc về một sự cố (Incident) đang diễn ra hay không.
6.  **Workflow Execution (Thực thi quy trình):** Nếu thỏa mãn điều kiện, các Workflow tự động sẽ chạy. Ví dụ: Nếu Alert là `Critical` và thuộc `Payment-Service` -> Tự động tạo Ticket Jira -> Gửi tin nhắn khẩn cấp vào Slack Group của Team Payment.
7.  **Real-time Visualization:** Kết quả được đẩy lên Dashboard thông qua Websocket để kỹ sư theo dõi ngay lập tức.

---

## 4. Hướng dẫn Triển khai nhanh (Quick Start)

Dự án hỗ trợ Docker-compose để cài đặt nhanh nhất:

```bash
# Tải script cài đặt
curl https://raw.githubusercontent.com/keephq/keep/main/start.sh | sh
```

Hệ thống sẽ khởi chạy 3 dịch vụ chính:
*   `keep-backend`: Cổng API (Port 8080)
*   `keep-frontend`: Giao diện người dùng (Port 3000)
*   `keep-websocket-server`: Xử lý dữ liệu thời gian thực (Port 6001)

## 5. Kết luận
Keep là một dự án có kiến trúc **Extensible** (dễ mở rộng) và **Declarative** (mô tả bằng cấu hình). Nó giải quyết bài toán "Alert Fatigue" (mệt mỏi vì quá nhiều cảnh báo) bằng cách áp dụng cả kỹ thuật truyền thống (Rules/CEL) và hiện đại (AI/LLMs), biến các cảnh báo tĩnh thành hành động động.