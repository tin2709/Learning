Dựa trên các tệp tin cấu trúc và nội dung mã nguồn cung cấp, dưới đây là phân tích chi tiết về dự án **GrowthBook** - nền tảng mã nguồn mở hàng đầu về Feature Flags và A/B Testing.

---

### 1. Công nghệ cốt lõi (Core Technology)

GrowthBook sử dụng một tập hợp công nghệ hiện đại, tối ưu cho việc xử lý dữ liệu và tích hợp đa nền tảng:

*   **Ngôn ngữ & Framework:** 
    *   **TypeScript (96%):** Là ngôn ngữ chính cho cả Front-end (Next.js) và Back-end (Node.js/Express).
    *   **Python:** Được sử dụng riêng cho **Stats Engine** (gói `gbstats`), tận dụng các thư viện khoa học dữ liệu như `numpy`, `pandas`, `scipy` để tính toán thống kê phức tạp.
*   **Cơ sở dữ liệu:** 
    *   **MongoDB (Mongoose):** Lưu trữ cấu hình feature flags, thông tin người dùng, metadata của thí nghiệm.
    *   **Warehouse Native (Điểm khác biệt nhất):** GrowthBook không lưu trữ dữ liệu sự kiện (event) của người dùng. Thay vào đó, nó kết nối trực tiếp đến các Data Warehouse hiện có của doanh nghiệp (BigQuery, Snowflake, Redshift, ClickHouse, Postgres...) để truy vấn dữ liệu gốc.
*   **SDK:** Hỗ trợ 24+ ngôn ngữ/framework (React, Go, Python, Swift, Android...), đảm bảo việc triển khai feature flag có hiệu năng cao (đánh giá cục bộ - local evaluation) và không có độ trễ mạng.

### 2. Tư duy Kiến trúc (Architectural Thinking)

GrowthBook được tổ chức theo mô hình **Monorepo** sử dụng `pnpm workspaces` với tư duy module hóa cực cao:

*   **Phân tầng Ứng dụng:**
    *   `packages/front-end`: Ứng dụng Next.js xử lý giao diện quản trị.
    *   `packages/back-end`: API server xử lý logic nghiệp vụ và điều phối truy vấn dữ liệu.
    *   `packages/shared`: Chứa các Zod schemas, types và tiện ích dùng chung, đảm bảo sự đồng bộ tuyệt đối giữa server và client.
    *   `packages/stats`: "Bộ não" thống kê chạy bằng Python.
*   **Mô hình Open Core:** Kiến trúc tách biệt rõ ràng phần mã nguồn mở (MIT) và các tính năng doanh nghiệp (Enterprise) nằm trong các thư mục `/enterprise`.
*   **Tính không phụ thuộc dữ liệu:** Kiến trúc được thiết kế để GrowthBook "đứng cạnh" luồng dữ liệu của bạn. Nó lấy kết quả từ kho dữ liệu, thực hiện tính toán thống kê và trả về báo cáo mà không cần thay đổi cách bạn thu thập dữ liệu hiện tại.

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **Zod-Driven Development:** Sử dụng `Zod` làm "nguồn sự thật duy nhất" (Source of Truth). Mọi Schema dữ liệu được định nghĩa bằng Zod trong `shared`, từ đó suy diễn (infer) ra các Type cho TypeScript và tự động tạo tài liệu OpenAPI.
*   **BaseModel Pattern:** Trong `back-end`, dự án sử dụng một mẫu `BaseModel` tùy chỉnh (thông qua `MakeModelClass`) để tiêu chuẩn hóa các thao tác CRUD, quản lý quyền (permissions) và tiền tố ID (ví dụ: `exp_`, `met_`).
*   **Strict Import Boundaries:** Sử dụng ESLint để kiểm soát ranh giới nhập khẩu dữ liệu. Ví dụ: Front-end chỉ có thể import từ `shared`, không được phép import trực tiếp từ `back-end` để tránh rò rỉ mã nguồn hoặc lỗi phụ thuộc vòng.
*   **Adapter Pattern cho Data Sources:** Xây dựng các trình kết nối (integrations) riêng biệt cho từng loại kho dữ liệu (Snowflake, BigQuery...). Mỗi adapter chịu trách nhiệm chuyển đổi các metric logic thành các truy vấn SQL đặc thù của kho dữ liệu đó.
*   **Local Evaluation in SDKs:** Các SDK được thiết kế để tải xuống bộ quy tắc (rules) một lần và thực hiện logic phân chia biến thể (bucketing) ngay trên thiết bị người dùng bằng thuật toán hashing (thường là MurmurHash), giúp tối ưu tốc độ.

### 4. Luồng hoạt động của hệ thống (System Workflow)

Luồng hoạt động của GrowthBook có thể chia thành hai chu kỳ chính:

#### A. Chu kỳ Feature Flag (Luồng thực thi):
1.  **Cấu hình:** Người dùng tạo Feature Flag trên Dashboard và đặt quy tắc (ví dụ: 10% người dùng thấy tính năng mới).
2.  **Phân phối:** Back-end xuất cấu hình này dưới dạng JSON qua SDK Endpoints (hoặc qua Proxy/CDN).
3.  **Đánh giá:** SDK trong ứng dụng khách tải JSON về, thực hiện tính toán dựa trên thuộc tính người dùng (User Attributes) để quyết định bật/tắt tính năng mà không cần gọi lại Server.
4.  **Theo dõi:** SDK kích hoạt một `trackingCallback` để gửi sự kiện "Người dùng X đã thấy biến thể Y" vào hệ thống tracking hiện có của bạn (như GA4, Segment).

#### B. Chu kỳ Thử nghiệm (Luồng phân tích):
1.  **Kích hoạt:** Người dùng yêu cầu làm mới kết quả thí nghiệm trên giao diện.
2.  **Truy vấn:** `back-end` lấy định nghĩa Metric (SQL), kết hợp với thông tin thí nghiệm để tạo ra một câu lệnh SQL tổng hợp lớn.
3.  **Thực thi:** Câu lệnh SQL được gửi đến Data Warehouse của bạn. Kết quả trả về là dữ liệu tổng hợp (aggregate data) như số lượng chuyển đổi, giá trị trung bình, phương sai...
4.  **Thống kê:** Dữ liệu này được đẩy vào **Python Stats Engine**. Tại đây, các thuật toán như Bayesian hoặc Frequentist, kết hợp với các kỹ thuật giảm phương sai (CUPED), sẽ tính toán tỷ lệ thắng, chỉ số tin cậy.
5.  **Hiển thị:** Kết quả cuối cùng được trình bày dưới dạng biểu đồ và báo cáo trực quan trên Dashboard.

### Tóm lại
GrowthBook là một hệ thống được xây dựng rất bài bản, chú trọng vào **tính an toàn dữ liệu** và **khả năng mở rộng**. Việc sử dụng TypeScript làm nền tảng kết hợp với Python cho xử lý toán học là một sự lựa chọn tối ưu, cho phép dự án vừa có giao diện linh hoạt, vừa có khả năng tính toán thống kê chính xác ở quy mô lớn.