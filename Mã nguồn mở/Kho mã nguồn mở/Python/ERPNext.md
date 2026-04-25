Dựa trên tệp tin và cấu trúc mã nguồn của dự án **ERPNext** (phiên bản v16-v17 phát triển), dưới đây là phân tích chi tiết về kiến trúc và công nghệ:

### 1. Công nghệ cốt lõi (Core Technology)

ERPNext không phải là một ứng dụng truyền thống mà là một ứng dụng xây dựng trên nền tảng **Frappe Framework**. Các thành phần chính bao gồm:

*   **Ngôn ngữ lập trình:**
    *   **Backend:** Python (chiếm ~81%), sử dụng kiến trúc không đồng bộ cho các tác vụ nền.
    *   **Frontend:** JavaScript (chiếm ~16%), sử dụng **Vue.js** (thông qua Frappe UI) và jQuery cho các tương tác metadata-driven.
*   **Cơ sở dữ liệu:** MariaDB (mặc định) hoặc PostgreSQL. Toàn bộ cấu trúc dữ liệu được định nghĩa qua các **DocType** (JSON files), giúp hệ thống có khả năng thay đổi schema mà không cần viết mã SQL thủ công.
*   **Công cụ build & Package:** sử dụng `package.json` (Node.js) cho các thành phần web và `pyproject.toml` cho các phụ thuộc Python (như `googlemaps`, `plaid-python`, `rapidfuzz`).
*   **Quản lý quy trình:** Sử dụng Redis cho caching và hàng đợi tác vụ (Worker), Socket.io cho thông báo thời gian thực.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của ERPNext tập trung vào tính **Module hóa (Modularity)** và **Metadata-driven**:

*   **Tư duy "Mọi thứ là một tài liệu" (Everything is a DocType):** Dù là một hóa đơn, một nhân viên hay một thiết lập hệ thống, tất cả đều tuân theo cấu trúc DocType. Điều này cho phép áp dụng các logic chung (phân quyền, workflow, bản in) cho mọi thực thể.
*   **Kiến trúc Đa thuê (Multi-tenancy):** Hỗ trợ nhiều công ty (Company) trong một site và nhiều site trên một server thông qua Frappe Bench.
*   **Tách biệt logic nghiệp vụ:** Logic dùng chung được đưa vào `controllers/`, trong khi logic đặc thù cho từng loại tài liệu nằm trong chính thư mục của DocType đó.
*   **Hệ thống kế toán Ledger-centric:** Mọi giao dịch tài chính cuối cùng đều đổ về `General Ledger` (Sổ cái) và `Payment Ledger`. Đây là "nguồn sự thật duy nhất" để truy xuất báo cáo tài chính.
*   **Thiết kế cho sự mở rộng (Extensibility):** Sử dụng `hooks.py` để ghi đè (override) hoặc bổ sung chức năng mà không cần sửa trực tiếp vào mã nguồn lõi.

### 3. Các kỹ thuật chính (Key Techniques)

*   **Regional Overrides (Ghi đè vùng miền):** Sử dụng Decorator `@erpnext.allow_regional` trong `erpnext/__init__.py`. Kỹ thuật này cho phép hệ thống tự động gọi các hàm tính thuế hoặc định dạng hóa đơn riêng cho từng quốc gia (như Ý, UAE, Ấn Độ) dựa trên quốc gia của Công ty.
*   **Hệ thống Patching tự động:** Tệp `patches.txt` quản lý các bản cập nhật dữ liệu. Khi nâng cấp phiên bản, các hàm Python trong thư mục `patches/` sẽ tự động chạy để chuyển đổi dữ liệu cũ sang cấu trúc mới (Data Migration).
*   **Deprecation Dumpster:** Sử dụng file `deprecation_dumpster.py` để quản lý mã lỗi thời. Thay vì xóa ngay lập tức, mã được đưa vào đây với thông báo cảnh báo và ngày dự kiến xóa, giúp duy trì tính tương thích ngược (Backward Compatibility).
*   **Nested Set Model:** Sử dụng cho các cấu trúc cây (Tree) như Biểu đồ tài khoản (Chart of Accounts), Nhóm hàng hóa (Item Group) để truy vấn nhanh các mối quan hệ cha-con phức tạp.
*   **Distributed Processing:** Sử dụng các tác vụ nền (Scheduler Events) được định nghĩa trong `hooks.py` (hourly, daily, monthly) để xử lý các việc nặng như tính khấu hao tài sản hoặc đóng kỳ kế toán.

### 4. Tóm tắt luồng hoạt động (Activity Flow Summary)

Một luồng nghiệp vụ điển hình (ví dụ: Bán hàng đến Kế toán) diễn ra như sau:

1.  **Giai đoạn Draft (Bản nháp):** Người dùng nhập liệu vào `Sales Invoice`. Hệ thống thực hiện `validate()` (kiểm tra tồn kho, hạn mức tín dụng).
2.  **Giai đoạn Submit (Xác nhận):**
    *   Tài liệu chuyển trạng thái `docstatus = 1`.
    *   Trigger `on_submit` được gọi.
    *   **General Ledger Entry:** Hệ thống tự động tạo các bút toán nợ/có vào bảng `tabGL Entry` thông qua `make_gl_entries` trong `general_ledger.py`.
    *   **Stock Ledger Entry:** Nếu có hàng hóa, các bút toán kho được tạo để cập nhật tồn kho tức thời.
3.  **Giai đoạn Thanh toán:** `Payment Entry` được tạo và đối chiếu (Reconcile) với hóa đơn. Luồng này sẽ cập nhật `Payment Ledger` để theo dõi công nợ quá hạn.
4.  **Giai đoạn Báo cáo:** Các công cụ báo cáo (như `financial_statements.py`) sẽ truy vấn dữ liệu từ các Ledger thay vì truy vấn trực tiếp từ Hóa đơn để đảm bảo tốc độ và tính chính xác về mặt kế toán.
5.  **Kỹ thuật Đảo ngược (Cancellation):** Nếu người dùng hủy hóa đơn, hệ thống không xóa bản ghi mà tạo ra các bút toán đảo (Reverse Entries) để đảm bảo tính minh bạch (Audit Trail), không cho phép sửa đổi lịch sử kế toán đã xác nhận.