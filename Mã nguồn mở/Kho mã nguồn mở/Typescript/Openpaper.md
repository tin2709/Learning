Dựa trên mã nguồn và cấu trúc thư mục của dự án **Open Paper (khoj-ai/openpaper)**, dưới đây là bản phân tích chi tiết về công nghệ, kiến trúc và nội dung file README chuyên nghiệp bằng tiếng Việt.

---

# Phân Tích Dự Án Open Paper

## 1. Công Nghệ Cốt Lõi (Core Tech Stack)
Dự án được xây dựng theo kiến trúc **Decoupled (tách biệt)** với 3 thành phần chính:

*   **Frontend (ap-client):**
    *   **Framework:** Next.js 15 (App Router) - Sử dụng phiên bản mới nhất với React 19.
    *   **Ngôn ngữ:** TypeScript.
    *   **Styling:** Tailwind CSS + Shadcn UI + Magic UI (cho các hiệu ứng animation).
    *   **Xử lý PDF:** `react-pdf-highlighter-extended` và `pdfjs-dist` để hiển thị và đánh dấu (annotation) trực tiếp trên file PDF.
    *   **State Management & Data Fetching:** SWR (cho client-side fetching) và React Context API (cho Auth/Theme).
*   **Backend (ap-server):**
    *   **Framework:** FastAPI (Python) - Hiệu năng cao, hỗ trợ asynchronous.
    *   **Database:** PostgreSQL với SQLAlchemy ORM.
    *   **Migration:** Alembic (quản lý lịch sử thay đổi DB).
    *   **Authentication:** Tích hợp Google OAuth và hệ thống xác thực qua Email (OTP).
    *   **Storage:** AWS S3 (hoặc tương thích S3) để lưu trữ tệp PDF.
*   **Asynchronous Jobs (ap-jobs):**
    *   **Task Queue:** Celery phối hợp với Redis/RabbitMQ.
    *   **Xử lý văn bản:** Kết hợp các thư viện phân tích PDF chuyên sâu để trích xuất metadata, văn bản và hình ảnh.

---

## 2. Kỹ Thuật và Tư Duy Kiến Trúc
*   **Kiến trúc Micro-services (Lite):** Tách biệt logic xử lý web (Server) và logic tính toán nặng (Jobs). Điều này giúp hệ thống không bị treo khi xử lý các file PDF dung lượng lớn hoặc chạy AI.
*   **RAG (Retrieval-Augmented Generation) chuyên sâu cho Nghiên cứu:**
    *   Không chỉ là chat đơn thuần, hệ thống tập trung vào **"Grounded Citations"** (Trích dẫn có căn cứ). AI khi trả lời sẽ chỉ chính xác đoạn văn bản trong PDF nơi nó lấy thông tin.
    *   **Parallel View (Chế độ xem song song):** Tư duy UX tập trung vào người làm nghiên cứu: Một bên đọc tài liệu gốc, một bên tương tác với AI.
*   **Quản lý trạng thái Job phức tạp:** Sử dụng cơ chế Polling (truy vấn định kỳ) để cập nhật trạng thái xử lý PDF từ Worker lên giao diện người dùng theo thời gian thực.
*   **Hệ thống Plugin/Tool cho LLM:** Backend được thiết kế để dễ dàng mở rộng các công cụ (tools) cho AI như tìm kiếm file, trích xuất bảng dữ liệu (Data Tables).

---

## 3. Các Kỹ Thuật Chính Nổi Bật
1.  **Audio Overviews:** Kỹ thuật chuyển đổi nội dung bài báo khoa học thành dạng âm thanh (giống podcast), giúp người dùng "nghe" nghiên cứu khi đang di chuyển.
2.  **Data Table Extraction:** Tự động trích xuất các bảng dữ liệu phức tạp từ PDF không cấu trúc thành dạng bảng có thể truy vấn.
3.  **Citation Graph:** Xây dựng bản đồ trích dẫn (sử dụng API OpenAlex) để người dùng thấy được mối liên hệ giữa các bài báo.
4.  **Security PDF Filtering:** Có các lớp CSS/JS để lọc bỏ các script độc hại tiềm ẩn trong file PDF (trong `globals.css`).
5.  **Multi-tenant & Subscription:** Tích hợp Stripe để quản lý gói cước (Free/Researcher) và giới hạn tài nguyên (với logic kiểm tra giới hạn nghiêm ngặt ở cả Backend và Frontend).

---

## 4. Nội dung File README (Tiếng Việt)

Dưới đây là nội dung file README được biên soạn lại để phản ánh đầy đủ sức mạnh của dự án:

```markdown
# 📄 Open Paper - Trạm Làm Việc AI Cho Nhà Nghiên Cứu

Open Paper là một nền tảng mã nguồn mở hiện đại, giúp quản lý thư viện nghiên cứu, đọc, chú thích và thấu hiểu sâu sắc các bài báo khoa học trong một giao diện tập trung.

## ✨ Tính năng nổi bật

- **AI Copilot Thông Minh:** Trò chuyện trực tiếp với tài liệu. AI giúp tóm tắt, giải thích các khái niệm phức tạp và trả lời câu hỏi dựa trên nội dung bài báo.
- **Trích Dẫn Có Căn Cứ (Grounded Citations):** Mọi câu trả lời của AI đều đi kèm liên kết chính xác đến vị trí văn bản trong tệp PDF.
- **Chế Độ Xem Song Song:** Đọc tài liệu gốc một bên và ghi chú/chat với AI ở bên còn lại mà không cần chuyển ngữ cảnh.
- **Data Tables:** Tự động trích xuất dữ liệu thô từ các bài báo thành bảng dữ liệu cấu trúc.
- **Audio Overviews:** Biến các bài báo khô khan thành định dạng âm thanh dễ tiếp nhận.
- **Biểu Đồ Trích Dẫn (Citation Graph):** Khám phá mối liên hệ giữa các công trình nghiên cứu thông qua dữ liệu từ OpenAlex.
- **Tìm Kiếm Toàn Diện:** Tìm kiếm trong hàng triệu bài báo công khai và lưu trực tiếp vào thư viện cá nhân.

## 🏗 Kiến trúc hệ thống

Dự án được chia thành 3 phần chính:
1.  **Client (`/client`):** Next.js 15, TypeScript, Tailwind CSS. Giao diện người dùng mượt mà với hỗ trợ Dark Mode.
2.  **Server (`/server`):** FastAPI (Python). Xử lý logic nghiệp vụ, xác thực, quản lý database và API.
3.  **Jobs (`/jobs`):** Celery Workers. Xử lý các tác vụ nặng như phân tích PDF, trích xuất metadata và tích hợp LLM.

## 🛠 Công nghệ sử dụng

- **Frontend:** Next.js, Radix UI, Lucide Icons, SWR.
- **Backend:** FastAPI, PostgreSQL, SQLAlchemy, Alembic.
- **AI/ML:** OpenAI GPT, Claude, LangChain (hoặc tương đương cho RAG).
- **Task Queue:** Celery, Redis.
- **Infrastructure:** Docker, AWS S3.

## 🚀 Cài đặt nhanh

### Yêu cầu hệ thống
- Python 3.10+
- Node.js 20+
- Docker & Docker Compose

### Các bước thực hiện
1. **Clone dự án:**
   ```bash
   git clone https://github.com/khoj-ai/openpaper.git
   cd openpaper
   ```

2. **Thiết lập Backend:**
   ```bash
   cd server
   # Cấu hình file .env (xem hướng dẫn trong server/README.md)
   pip install -r pyproject.toml
   python app/scripts/run_migrations.py
   python -m app.main
   ```

3. **Thiết lập Frontend:**
   ```bash
   cd client
   yarn install
   yarn dev
   ```

4. **Chạy Background Jobs:**
   ```bash
   cd jobs
   ./scripts/start_worker.sh
   ```

## 📜 Giấy phép
Dự án được phát hành dưới giấy phép **GNU Affero General Public License v3.0 (AGPL-3.0)**.

---
Được phát triển bởi đội ngũ **Khoj AI**.
```

---

### Tóm tắt luồng hoạt động (Workflow Summary):
1.  **Upload:** Người dùng tải PDF lên thông qua Client -> Server lưu vào S3 -> Đẩy một Task vào Celery.
2.  **Processing:** Job Worker nhận Task -> Phân tích PDF (trích xuất text, ảnh, bảng) -> Lưu metadata vào Postgres -> (Tùy chọn) Tạo Vector Embeddings.
3.  **Consumption:** Người dùng mở bài báo -> Client tải PDF và Metadata -> Người dùng Highlight/Chat -> Server gửi ngữ cảnh tới LLM -> Trả kết quả về kèm Citation ID để Client nhảy đến đúng trang PDF.