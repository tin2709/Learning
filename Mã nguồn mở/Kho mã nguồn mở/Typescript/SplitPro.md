Dựa trên các tệp tin bạn cung cấp, dưới đây là phân tích chuyên sâu về hệ thống **SplitPro** - một giải pháp thay thế mã nguồn mở cho Splitwise.

---

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

SplitPro sử dụng một ngăn xếp công nghệ hiện đại, tối ưu cho việc triển khai tự lưu trữ (self-hosting) và hiệu suất:

*   **Framework:** **Next.js 15 (Pages Router)**. Việc sử dụng Pages Router thay vì App Router cho thấy sự ưu tiên về tính ổn định và khả năng tương thích với các thư viện hiện có như `next-i18next`.
*   **Ngôn ngữ:** **TypeScript** với cấu hình nghiêm ngặt (`strict: true`, `noUncheckedIndexedAccess: true`), đảm bảo an toàn kiểu dữ liệu ở mức cao.
*   **Cơ sở dữ liệu & ORM:** **PostgreSQL** kết hợp với **Prisma**. Điểm đặc biệt là hệ thống phụ thuộc nặng vào các tính năng nâng cao của Postgres như **Database Views** và **pg_cron**.
*   **Giao tiếp API:** **tRPC**. Điều này tạo ra một "Type-safe bridge" giữa Client và Server, giúp giảm thiểu lỗi runtime khi thay đổi cấu trúc dữ liệu.
*   **Quản lý trạng thái:** **Zustand** (Client-side state) và **React Query** (Server-side state).
*   **Styling:** **Tailwind CSS v4** và **ShadcnUI**, mang lại giao diện hiện đại và khả năng tùy biến cao.

---

### 2. Tư duy Kiến trúc (Architectural Philosophy)

Kiến trúc của SplitPro đã có bước tiến hóa quan trọng từ v1 sang v2, tập trung vào **Tính nhất quán tuyệt đối (Absolute Consistency)**:

*   **Nguồn sự thật duy nhất (Single Source of Truth):** Thay vì lưu trữ số dư (balances) trong các bảng riêng biệt (dễ dẫn đến sai lệch dữ liệu khi có lỗi logic), SplitPro v2 tính toán số dư "on-the-fly" thông qua **Database Views (`BalanceView`)**. Nếu các bản ghi chi phí (Expenses) đúng, số dư chắc chắn đúng.
*   **Kế toán kép (Double-entry accounting):** Mọi chi phí đều tạo ra các bản ghi số dư hai chiều giữa người trả và người tham gia thông qua View, đảm bảo tổng tài chính luôn bằng không.
*   **Xử lý số thực bằng BigInt:** Để tránh lỗi làm tròn của số dấu phẩy động (floating-point errors), toàn bộ giá trị tiền tệ được lưu trữ dưới dạng `BigInt` (đơn vị nhỏ nhất của tiền tệ, ví dụ: cents).
*   **Thiết kế Group-First:** Nhóm (Groups) được coi là thực thể hạng nhất. Ngay cả các chi tiêu cá nhân cũng được xử lý như một nhóm có ID là `null`.

---

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Techniques)

Hệ thống bộc lộ những kỹ thuật lập trình rất khắt khe và thực dụng:

*   **Yoda Conditions:** Dự án áp dụng phong cách so sánh Yoda (ví dụ: `if (0n === amount)`) để tránh lỗi gán nhầm giá trị trong các điều kiện logic phức tạp.
*   **Xử lý tiền lẻ thừa (Leftover Pennies):** Khi chia tiền không đều (ví dụ: $10 chia 3 người), SplitPro sử dụng thuật toán phân phối phần dư một cách **định mệnh (deterministic)** dựa trên số tiền và ngày tháng, thay vì chọn ngẫu nhiên.
*   **Hệ thống Di trú dữ liệu Phức tạp:** Sử dụng `instrumentation.ts` để chạy các bản cập nhật dữ liệu (migrations) phức tạp không thể xử lý bằng SQL thuần túy ngay khi server khởi động.
*   **Tối ưu hóa Hình ảnh Local:** Thay vì dùng S3/Cloudinary, hệ thống tự xử lý nén hình ảnh hóa đơn sang định dạng `WebP` và tạo thumbnail ngay tại máy chủ local để giảm chi phí vận hành cho người dùng self-host.
*   **Hỗ trợ PWA sâu:** SplitPro không chỉ là web, nó được thiết kế để hoạt động như một ứng dụng di động thông qua Service Workers (`serwist`) và Push Notifications (`web-push`).

---

### 4. Luồng Hoạt động Hệ thống (System Workflows)

#### A. Luồng tạo Chi phí (Expense Creation)
1.  Người dùng nhập dữ liệu qua `AddExpensePage`.
2.  Store của Zustand (`addStore.ts`) tính toán việc chia tiền dựa trên `SplitType` (EQUAL, PERCENTAGE, SHARE, vất vả nhất là ADJUSTMENT).
3.  Dữ liệu được gửi qua tRPC mutation.
4.  Backend thực hiện một **Database Transaction** duy nhất để chèn bản ghi vào bảng `Expense` và `ExpenseParticipant`.
5.  Database View tự động cập nhật số dư hiển thị mà không cần tác động thêm.

#### B. Luồng Giao dịch Định kỳ (Recurring Transactions)
1.  Người dùng tạo một template chi phí và lịch trình (Cron expression).
2.  Hệ thống sử dụng **`pg_cron`** (một extension của Postgres) để lập lịch chạy lệnh SQL ngay trong cơ sở dữ liệu.
3.  Đến thời điểm, hàm PL/pgSQL `duplicate_expense_with_participants` được gọi để nhân bản chi phí từ template sang bản ghi thực tế.

#### C. Luồng đơn giản hóa nợ (Debt Simplification)
Hệ thống sử dụng thuật toán để giảm tối đa số lượng giao dịch cần thiết trong một nhóm (ví dụ: A nợ B 10k, B nợ C 10k => A nợ C 10k). Việc này được xử lý thông qua logic tính toán lại đồ thị nợ trước khi hiển thị số dư nhóm.

#### D. Tích hợp Ngân hàng (Bank Integration)
Sử dụng Plaid hoặc GoCardless để kéo các giao dịch thực tế từ tài khoản ngân hàng của người dùng, sau đó cho phép người dùng chuyển đổi các giao dịch đó thành các khoản chi phí trong SplitPro chỉ với vài lần nhấp chuột.

---

### Tóm lại
SplitPro là một minh chứng của việc áp dụng **kiến trúc dựa trên dữ liệu (data-driven architecture)**. Bằng cách đẩy logic tính toán số dư xuống tầng Database View và sử dụng BigInt cho mọi phép tính, nó đạt được độ tin cậy về tài chính cao - điều tối quan trọng đối với một ứng dụng quản lý tiền bạc.