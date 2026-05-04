Dưới đây là phân tích chi tiết về hệ thống **CocoIndex** dựa trên mã nguồn và tài liệu kiến trúc bạn cung cấp.

---

### 1. Công nghệ cốt lõi (Core Technology)

CocoIndex được xây dựng theo mô hình **Hybrid Engine** (Động cơ lai), kết hợp giữa hiệu suất của Rust và tính linh hoạt của Python.

*   **Ngôn ngữ động cơ (Core Engine): Rust.** 
    *   Sử dụng Rust để xử lý các tác vụ nặng: Concurrency (Đồng tử), Quản lý bộ nhớ, Cắt nhỏ dữ liệu (Chunking) và đặc biệt là quản lý trạng thái nội bộ.
    *   Các thư viện chính: `Tokio` (Async runtime), `Serde` (Serialization), `Axum` (Server), và `Blake2/SHA` (để tính toán fingerprint/băm dữ liệu).
*   **Ngôn ngữ giao tiếp (SDK/API): Python.**
    *   Cho phép người dùng viết logic chuyển đổi dữ liệu bằng Python đơn giản.
    *   Sử dụng `PyO3` và `maturin` để tạo cầu nối (Bridge) hiệu suất cao giữa Python và Rust.
*   **Lưu trữ trạng thái nội bộ (Internal State): LMDB.**
    *   Hệ thống sử dụng **LMDB** (Lightning Memory-Mapped Database) để lưu trữ metadata, kết quả memoization và vết của dữ liệu. LMDB được chọn vì tốc độ đọc cực nhanh, hỗ trợ giao dịch ACID và phù hợp với mô hình mapping bộ nhớ.
*   **Hệ sinh thái Connector:**
    *   Tích hợp đa dạng các Vector DB (Qdrant, LanceDB, Turbopuffer), RDBMS (Postgres, SQLite) và các nguồn dữ liệu (S3, Google Drive, Kafka).

---

### 2. Tư duy kiến trúc (Architectural Thinking)

Kiến trúc của CocoIndex dựa trên triết lý **"React for Data Engineering"**.

*   **Mô hình Declarative (Khai báo):** Thay vì viết script để "copy dữ liệu từ A sang B", người dùng khai báo "Trạng thái mong muốn (Target State) của hệ thống đích". Hệ thống sẽ tự tính toán những gì cần làm để đạt được trạng thái đó.
*   **Công thức cốt lõi: `Target = F(Source)`.** 
    *   Mọi dữ liệu đích đều là kết quả của một hàm xử lý dựa trên dữ liệu nguồn. Nếu mã nguồn của hàm `F` hoặc dữ liệu `Source` thay đổi, hệ thống sẽ tự động cập nhật.
*   **Incremental Processing (Xử lý gia tăng):** Đây là "linh hồn" của hệ thống. Thay vì chạy lại toàn bộ (Batch) gây tốn kém, CocoIndex chỉ xử lý các **Delta (phần thay đổi)**. Điều này giúp đạt được độ trễ dưới một giây (Sub-second freshness).
*   **Stable Paths (Định danh ổn định):** Mọi thành phần trong pipeline (Processing Component) được gán một "Stable Path" (ví dụ: `process/filename`). Đây là chìa khóa để đối chiếu giữa các lần chạy (Run) khác nhau để biết cái gì cần xóa, cái gì cần cập nhật.
*   **Failure Isolation (Cô lập lỗi):** Kiến trúc cho phép một component lỗi (ví dụ: lỗi parse một file PDF) mà không làm dừng toàn bộ pipeline. Các phần dữ liệu khác vẫn được xử lý và đồng bộ bình thường.

---

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Memoization & Fingerprinting:**
    *   Sử dụng decorator `@coco.fn(memo=True)`. Hệ thống sẽ băm (hash) cả đầu vào (input) và mã nguồn của hàm (code hash). Nếu cả hai không đổi, kết quả được lấy từ cache (LMDB) thay vì thực thi lại.
*   **Dependency Injection qua Context:**
    *   Sử dụng `ContextKey` và `ContextProvider` (giống React Context). Kỹ thuật này giúp chia sẻ các tài nguyên dùng chung (như connection pool của Database) xuyên suốt các hàm mà không cần truyền tham số thủ công.
*   **Two-Phase Processing (Xử lý hai pha):**
    *   **Giai đoạn 1 (Pure Processing):** Chạy code Python, tính toán kết quả và "khai báo" trạng thái mong muốn vào bộ nhớ đệm (không gây side-effect).
    *   **Giai đoạn 2 (Submit/Sync):** Rust engine nhận danh sách khai báo, so sánh với trạng thái cũ và thực hiện các lệnh I/O (Insert/Update/Delete) hàng loạt (Batching) vào DB đích.
*   **Strong Typing & Stubs:**
    *   Sử dụng `core.pyi` (Type stubs) để cung cấp gợi ý kiểu cho mã Rust khi gọi từ Python, đảm bảo tính an toàn về kiểu dữ liệu trong môi trường đa ngôn ngữ.

---

### 4. Luồng hoạt động hệ thống (System Operation Flow)

Quy trình thực thi một App trong CocoIndex diễn ra như sau:

1.  **Khởi tạo (Initialization):** App nạp `AppConfig`, mở cơ sở dữ liệu `LMDB` và thiết lập môi trường (Environment).
2.  **Quét nguồn (Source Discovery):** Các hàm như `localfs.walk_dir()` quét các mục dữ liệu nguồn.
3.  **Giai đoạn Xử lý & Memoize (Processing Phase):**
    *   Với mỗi mục dữ liệu, động cơ kiểm tra **Fingerprint** trong LMDB.
    *   Nếu trùng khớp: Bỏ qua thực thi hàm, lấy Target States đã lưu từ lần trước.
    *   Nếu khác: Thực thi hàm `@coco.fn`, tạo ra các đối tượng `TargetState` mới.
4.  **Đối chiếu (Reconciliation):** Động cơ Rust so sánh danh sách `TargetState` hiện tại với danh sách cũ trong LMDB dựa trên `Stable Path`.
    *   Cái mới -> Đánh dấu **Create**.
    *   Cái cũ thay đổi nội dung -> Đánh dấu **Update**.
    *   Cái cũ không còn tồn tại -> Đánh dấu **Delete**.
5.  **Đồng bộ (Synchronization/Sink):** Động cơ thực hiện các Action đã đánh dấu vào các Database đích (Postgres, Vector DB...) theo cơ chế Batching để tối ưu hiệu suất.
6.  **Lưu vết (Finalize):** Cập nhật các Fingerprint và Target State mới vào LMDB để phục vụ lần chạy tiếp theo.

### Tổng kết
CocoIndex không đơn thuần là một công cụ ETL, mà là một **Incremental Sync Engine** giúp thu hẹp khoảng cách về dữ liệu giữa hệ thống lưu trữ truyền thống và các AI Agent cần bối cảnh (context) luôn tươi mới. Nó giải quyết bài toán chi phí và độ trễ bằng cách biến dữ liệu thành một "luồng thực thi có trạng thái" (stateful dataflow).