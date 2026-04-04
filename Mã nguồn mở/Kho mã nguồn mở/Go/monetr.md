Dựa trên cấu trúc thư mục và nội dung mã nguồn của dự án **monetr**, đây là phân tích chi tiết về các khía cạnh kỹ thuật và kiến trúc của hệ thống:

### 1. Công nghệ cốt lõi (Core Tech Stack)
Dự án này là một sự kết hợp thú vị giữa hiệu năng của hệ thống cấp thấp và sự linh hoạt của web hiện đại:
*   **Backend:** Sử dụng **Go (Golang)** làm ngôn ngữ chính. Các thư viện quan trọng bao gồm:
    *   **Echo:** Framework cho Web API.
    *   **go-pg/pg:** ORM mạnh mẽ cho PostgreSQL.
    *   **Cobra/Viper:** Quản lý CLI và cấu hình (YAML/Env).
*   **Frontend:** **TypeScript & React**. Đáng chú ý dự án sử dụng **Rsbuild** (dựa trên Rspack) để build thay vì Webpack truyền thống, giúp tăng tốc độ phát triển.
*   **Hệ thống Build:** Rất đặc biệt khi sử dụng **CMake** kết hợp với **Makefile**. Đây là cách tiếp cận thường thấy trong các dự án C++, dùng để quản lý việc biên dịch chéo (cross-compilation) cho nhiều nền tảng và quản lý dependencies phức tạp giữa Go và Node.js.
*   **Database & Cache:** **PostgreSQL** là kho lưu trữ chính. Dự án đã chuyển hướng sang dùng **Valkey** (bản fork mã nguồn mở của Redis) để làm caching và message queue.
*   **Hạ tầng:** Docker & Docker Compose cho môi trường phát triển và triển khai.
*   **Tích hợp bên thứ ba:** Plaid (dữ liệu ngân hàng), Stripe (thanh toán), AWS KMS/HashiCorp Vault/OpenBao (quản lý khóa mã hóa).

### 2. Tư duy Kiến trúc (Architectural Thinking)
*   **Kiến trúc Plug-and-Play (Modular):** Hệ thống được thiết kế để có thể chạy trên nhiều hạ tầng khác nhau. Ví dụ: Phần `secrets` hỗ trợ nhiều provider (AWS KMS, Vault, OpenBao hoặc Plaintext). Phần `storage` hỗ trợ cả S3 và Filesystem nội bộ.
*   **Tối ưu cho Tự triển khai (Self-hosting First):** Kiến trúc cho phép người dùng chạy toàn bộ hệ thống trên phần cứng cá nhân mà không phụ thuộc vào cloud. Việc tích hợp sẵn Nginx proxy và cơ chế tự sinh chứng chỉ TLS trong `compose/` cho thấy sự chú trọng vào UX của người quản trị hệ thống.
*   **Xử lý tác vụ nền (Background Jobs):** Sử dụng cơ chế hàng đợi dựa trên Postgres (`server/queue/postgres.go`) để xử lý các việc nặng như đồng bộ dữ liệu ngân hàng, xử lý file OFX, hay tính toán dự báo tài chính.
*   **Bảo mật đa tầng:** Mã hóa dữ liệu nhạy cảm (Tokens) bằng KMS, hỗ trợ MFA (TOTP), và quản lý phiên (Session) qua Paseto/JWT.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)
*   **Tối ưu hóa hiệu năng cực cao (SIMD & Assembly):** Đây là điểm ấn tượng nhất. Dự án có thư mục `server/internal/calc/` chứa mã **Assembly (`.s`)** và sử dụng **SIMD (Single Instruction, Multiple Data)** để tăng tốc các phép toán vector. Điều này phục vụ cho thuật toán tìm kiếm giao dịch tương tự.
*   **Thuật toán học máy offline:** Thay vì dùng AI đám mây, monetr sử dụng các thuật toán toán học thuần túy như **TF-IDF** (để định danh văn bản) và **DBSCAN** (để phân cụm dữ liệu) nhằm nhóm các giao dịch giống nhau mà vẫn đảm bảo quyền riêng tư dữ liệu (không đẩy data ra ngoài).
*   **Migration Schema tích hợp:** Toàn bộ lịch sử thay đổi DB được quản lý bằng Go mã nguồn (`server/migrations/schema/`), cho phép binary tự nâng cấp DB khi khởi chạy (`--migrate`).
*   **Testing Rigorous:** Sử dụng `gomock` cho unit test backend và `rstest` cho frontend, đảm bảo độ tin cậy của ứng dụng tài chính.

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Nạp dữ liệu (Ingestion):**
    *   Người dùng kết nối ngân hàng qua **Plaid** (Auto) hoặc upload file **OFX/QFX** (Manual).
    *   Hệ thống nhận dữ liệu thô, thực hiện chuẩn hóa thông qua `server/datasources/ofx`.
2.  **Xử lý và Phân cụm (Clustering):**
    *   Các giao dịch mới được đưa vào hàng đợi.
    *   Worker chạy thuật toán **TF-IDF** để trích xuất đặc trưng từ mô tả giao dịch, sau đó dùng **DBSCAN** để nhóm chúng vào các "Transaction Clusters".
3.  **Lập ngân sách (Budgeting Logic):**
    *   Người dùng thiết lập "Funding Schedules" (Lịch trình nạp tiền - thường là ngày nhận lương).
    *   Hệ thống tự động tính toán số tiền cần trích ra từ mỗi kỳ lương cho các "Expenses" (Chi phí định kỳ) và "Goals" (Mục tiêu tiết kiệm).
4.  **Tính toán Free-To-Use:**
    *   Đây là chỉ số quan trọng nhất: `Tổng tiền - (Tiền đã dành cho hóa đơn + Tiền dành cho mục tiêu) = Free-To-Use`.
    *   Số dư này được cập nhật thời gian thực khi có giao dịch mới phát sinh.
5.  **Giao diện và Thông báo:**
    *   Frontend React lấy dữ liệu qua REST API.
    *   Nếu có sự cố (ví dụ Plaid mất kết nối), hệ thống gửi email cảnh báo thông qua template React Email.

### Tổng kết
**monetr** không đơn thuần là một app quản lý chi tiêu. Nó là một hệ thống **tài chính kỹ thuật (FinTech)** được xây dựng với tư duy kỹ thuật hệ thống (System Engineering), ưu tiên quyền riêng tư và hiệu năng xử lý toán học tại chỗ thay vì dựa dẫm vào Cloud AI.