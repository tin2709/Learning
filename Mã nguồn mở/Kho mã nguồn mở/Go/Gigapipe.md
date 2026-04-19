Dựa trên mã nguồn và cấu trúc thư mục của dự án **Gigapipe** (trước đây là **qryn**), tôi xin phân tích các khía cạnh kỹ thuật cốt lõi của hệ thống quan sát (observability) đa ngôn ngữ này như sau:

### 1. Công nghệ cốt lõi (Core Technology Stack)

Gigapipe được xây dựng như một "Warehouse" tập trung cho dữ liệu quan sát, thay thế cho việc phải chạy nhiều hệ thống riêng biệt:

*   **Ngôn ngữ lập trình:** **Go (Golang)** phiên bản mới (1.24+). Tận dụng tối đa tính năng `concurrency` (Goroutines/Channels) để xử lý hàng triệu bản ghi mỗi giây.
*   **Cơ sở dữ liệu chủ đạo:** **ClickHouse**. Đây là một OLAP database cực nhanh. Dự án tận dụng ClickHouse để lưu trữ cả Logs, Metrics, Traces và Profiles thay vì dùng Loki (cho logs), Prometheus (cho metrics) và Tempo (cho traces) riêng biệt.
*   **Giao thức hỗ trợ:** OpenTelemetry (OTLP), Loki API, Prometheus Remote Write, Tempo/Zipkin, Pyroscope.
*   **Thư viện Parser:** Sử dụng `alecthomas/participle/v2` để xây dựng trình phân tích cú pháp (parser) cho các ngôn ngữ truy vấn như LogQL và PromQL.
*   **Runtime:** Chạy trên **Alpine Linux** (Dockerfile) để tối ưu dung lượng container và tốc độ khởi động.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Gigapipe dựa trên tư tưởng **"All-in-One Polyglot"**:

*   **Kiến trúc hợp nhất (Unified Storage):** Thay vì phân tán dữ liệu ra nhiều DB, dự án quy nạp tất cả về cấu trúc bảng của ClickHouse. Điều này giúp giảm chi phí vận hành (Ops) và cho phép truy vấn chéo (cross-correlation) giữa logs và metrics một cách dễ dàng.
*   **Tách biệt Reader và Writer:**
    *   **Writer:** Tập trung vào việc nhận dữ liệu từ các nguồn (ingestion), giải mã (unmarshal), tạo fingerprint cho các labels và thực hiện `Bulk Insert` vào ClickHouse để tối ưu hiệu năng ghi.
    *   **Reader:** Đóng vai trò là một "Translation Layer". Nó nhận các truy vấn PromQL/LogQL từ Grafana, biên dịch chúng thành SQL của ClickHouse, thực thi và trả về kết quả định dạng chuẩn mà Grafana có thể hiểu.
*   **Kiến trúc hướng Cluster:** Hỗ trợ mô hình `Distributed Tables` của ClickHouse. Dữ liệu có thể được phân tán và truy vấn trên nhiều node (shards/replicas) thông qua các script SQL được quản lý tự động trong thư mục `ctrl/qryn/sql/`.

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **Transpilation (Chuyển dịch mã):** Đây là kỹ thuật quan trọng nhất. Dự án không thực thi LogQL/PromQL trực tiếp mà chuyển đổi (transpile) chúng thành SQL phức tạp. Ví dụ: các hàm `rate()`, `count_over_time()` trong PromQL được ánh xạ thành các hàm `argMax`, `countMerge` trong ClickHouse SQL.
*   **Fingerprinting & Hashing:** Sử dụng **DJBHash** và **CityHash** (trong `helputils`) để băm các bộ nhãn (labels/tags) thành một ID duy nhất (Fingerprint). Kỹ thuật này giúp việc nhóm và tìm kiếm dữ liệu theo series nhanh hơn gấp nhiều lần.
*   **Embedded SQL Scripts:** Sử dụng tính năng `//go:embed` của Go để nhúng trực tiếp các file `.sql` vào trong file thực thi. Điều này giúp hệ thống tự động khởi tạo (init) và nâng cấp (migration) schema database khi khởi động mà không cần công cụ ngoài.
*   **Smart Buffering:** Trong `reader/utils/smart_buffer`, dự án triển khai cơ chế đệm dữ liệu thông minh để xử lý kết quả truy vấn lớn, giúp tránh lỗi tràn bộ nhớ (OOM) khi người dùng truy vấn một lượng lớn logs.
*   **Middleware Pattern:** Sử dụng middleware cho việc kiểm soát nén dữ liệu (Gzip), xác thực (Basic Auth), và CORS, đảm bảo tính bảo mật và tối ưu băng thông API.

### 4. Luồng hoạt động của hệ thống (System Workflow)

Hệ thống vận hành qua hai luồng chính:

#### A. Luồng Ghi dữ liệu (Ingestion Path)
1.  **Receive:** Các Agent (như Otel Collector, Fluentbit, Prometheus) gửi dữ liệu đến các endpoint tương ứng ở module `writer`.
2.  **Unmarshal:** `writer` giải mã dữ liệu (Protobuf hoặc JSON).
3.  **Fingerprint:** Hệ thống tính toán hash cho các bộ label để định danh time-series.
4.  **Batching:** Dữ liệu không ghi ngay mà được gom thành các batch (dựa trên `BULK_MAX_SIZE_BYTES` hoặc `BULK_MAX_AGE_MS`).
5.  **Store:** Thực hiện lệnh `INSERT` vào các bảng như `samples_v3`, `tempo_traces`, `profiles`.

#### B. Luồng Truy vấn (Query Path)
1.  **Request:** Grafana hoặc Explorer gửi truy vấn (VD: một đoạn mã LogQL) đến module `reader`.
2.  **Parse:** `logql_parser` phân tích đoạn mã thành cây cú pháp (AST).
3.  **Transpile:** `clickhouse_planner` duyệt cây AST và tạo ra câu lệnh ClickHouse SQL tương ứng.
4.  **Execute:** Truy vấn được gửi đến ClickHouse.
5.  **Format:** Kết quả từ DB được định dạng lại thành cấu trúc JSON chuẩn của Prometheus/Loki và trả về cho client.

#### C. Luồng Bảo trì (Maintenance)
*   Module `ctrl` chạy các tiến trình nền để thực hiện **Rotation** (xoay vòng dữ liệu), xóa dữ liệu cũ dựa trên TTL (Time To Live) và tối ưu hóa các bảng (Optimize parts) để duy trì hiệu suất.

### Tổng kết
**Gigapipe** là một ví dụ điển hình về việc tối ưu hóa hạ tầng quan sát bằng cách tận dụng sức mạnh xử lý khối lượng lớn của ClickHouse. Kỹ thuật lập trình tập trung sâu vào việc **xử lý ngôn ngữ (Parsing)** và **biên dịch truy vấn (Transpilation)**, giúp nó trở thành một giải pháp "nhẹ đầu" cho các kỹ sư DevOps khi chỉ cần quản lý duy nhất một kho lưu trữ cho mọi loại dữ liệu quan sát.