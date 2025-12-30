Dưới đây là bản tóm tắt chi tiết về kho lưu trữ **Midday** dựa trên cấu trúc mã nguồn và tài liệu bạn cung cấp. Midday là một ví dụ điển hình về ứng dụng tài chính hiện đại kết hợp AI (AI-native application).

---

### 1. Công nghệ cốt lõi (Core Technology)

Dự án sử dụng một bộ công nghệ (stack) hiện đại, tập trung vào hiệu suất và khả năng mở rộng:

*   **Ngôn ngữ & Runtime:** TypeScript (98.2%), chạy trên **Bun** (thay vì Node.js) để tối ưu tốc độ thực thi và quản lý package.
*   **Kiến trúc Monorepo:** Sử dụng **Turborepo** để quản lý nhiều ứng dụng và gói thư viện trong cùng một kho lưu trữ.
*   **Frontend:**
    *   **Next.js:** Cho Dashboard và Website.
    *   **TailwindCSS & Shadcn UI:** Xây dựng giao diện đồng nhất.
    *   **Tauri:** Đóng gói ứng dụng Desktop (sử dụng Rust làm backend hệ thống).
    *   **Expo:** Cho ứng dụng di động.
*   **Backend & API:**
    *   **Hono:** Một framework web cực nhanh cho API chính.
    *   **tRPC:** Đảm bảo type-safety (an toàn kiểu dữ liệu) tuyệt đối giữa Client và Server.
    *   **Supabase:** Sử dụng toàn diện cho Database (PostgreSQL), Auth, Storage và Realtime.
*   **AI Stack:**
    *   **Vercel AI SDK:** Cốt lõi để tích hợp các mô hình ngôn ngữ lớn (LLM).
    *   **Mô hình:** OpenAI (GPT-4o), Google Gemini, Mistral.
*   **Cơ sở hạ tầng & Dịch vụ:**
    *   **Trigger.dev:** Quản lý các tác vụ chạy ngầm (background jobs) phức tạp.
    *   **Fly.io:** Hosting cho API và tRPC server.
    *   **Dịch vụ tài chính:** Plaid, GoCardLess, Teller (kết nối ngân hàng).
    *   **Typesense:** Search engine cho dữ liệu tài chính.

---

### 2. Tư duy kiến trúc (Architectural Mindset)

Kiến trúc của Midday được thiết kế theo hướng **"AI-First"** và **"Modular"**:

*   **Tư duy Agentic (Tác nhân):** Thay vì một chatbot đơn giản, hệ thống chia nhỏ trí tuệ nhân tạo thành các "Specialist Agents" (Tác nhân chuyên biệt) như `analytics`, `invoices`, `transactions`. Một `mainAgent` đóng vai trò điều phối (triage) để chuyển hướng yêu cầu của người dùng đến đúng chuyên gia.
*   **Thiết kế dựa trên Artifact (Vật phẩm):** AI không chỉ trả về văn bản. Nó trả về các `Artifacts` (như biểu đồ dòng tiền, bảng cân đối kế toán) được định nghĩa bằng schema rõ ràng để Frontend có thể hiển thị dưới dạng UI tương tác (Canvas).
*   **Type-Safe Fullstack:** Việc sử dụng tRPC và thư viện shared schema (Zod) giữa các package giúp giảm thiểu lỗi runtime khi truyền dữ liệu từ DB lên giao diện.
*   **Tách biệt logic nghiệp vụ:** Các logic về kế toán (`packages/accounting`), xử lý hóa đơn (`packages/invoice`), và kết nối ngân hàng (`apps/engine`) được tách thành các gói riêng biệt để có thể tái sử dụng.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **RAG (Retrieval-Augmented Generation) & Memory:** Sử dụng Redis để lưu trữ bộ nhớ làm việc (working memory) cho AI, giúp trợ lý nhớ được ngữ cảnh kinh doanh của người dùng qua các phiên chat.
*   **Xử lý tài liệu thông minh (OCR/Classification):** Khi người dùng tải hóa đơn lên, một worker (Trigger.dev) sẽ kích hoạt quy trình phân loại và trích xuất dữ liệu tự động, sau đó khớp (reconciliation) với các giao dịch ngân hàng thực tế.
*   **Kiến trúc Multi-Agent:** Sử dụng kỹ thuật `handoff` (bàn giao). Nếu tác nhân `General` thấy câu hỏi về tài chính, nó sẽ tự động chuyển quyền xử lý cho tác nhân `Reports`.
*   **Read-after-write Consistency:** Sử dụng Redis cache để đảm bảo rằng sau khi người dùng cập nhật dữ liệu, AI sẽ thấy dữ liệu mới ngay lập tức (tránh độ trễ của DB replication).
*   **Internationalization (i18n):** Hỗ trợ đa ngôn ngữ và định dạng tiền tệ/ngày tháng theo vùng miền của người dùng ngay từ cấp độ kiến trúc.

---

### 4. Tóm tắt luồng hoạt động của dự án (Activity Flow)

Dưới đây là luồng hoạt động chính khi người dùng tương tác với hệ thống:

#### A. Luồng Trợ lý AI (Chat & Insight):
1.  **Input:** Người dùng hỏi "Tháng này tôi chi tiêu bao nhiêu cho quảng cáo?".
2.  **Triage:** `mainAgent` nhận yêu cầu, phân tích và nhận diện đây là yêu cầu về báo cáo. Nó gọi `handoff_to_agent` để chuyển sang `reportsAgent`.
3.  **Tool Call:** `reportsAgent` nhận ngữ cảnh, quyết định gọi tool `getSpending`.
4.  **Data Fetch:** Tool này truy vấn database (qua package `@midday/db`) để lấy dữ liệu giao dịch đã phân loại.
5.  **Artifact Generation:** Hệ thống tạo ra một `spending-canvas`. AI trả về một đoạn tóm tắt văn bản kèm theo dữ liệu cấu trúc cho biểu đồ.
6.  **UI Render:** Dashboard hiển thị biểu đồ chi tiết (Canvas) cạnh cửa sổ chat.

#### B. Luồng Xử lý hóa đơn (Magic Inbox):
1.  **Upload:** Người dùng gửi hóa đơn qua Email, WhatsApp hoặc tải trực tiếp lên Vault.
2.  **Worker:** `Trigger.dev` nhận sự kiện, đẩy vào hàng đợi xử lý.
3.  **Analysis:** AI xử lý OCR để đọc số tiền, ngày tháng, nhà cung cấp.
4.  **Matching:** Hệ thống tìm kiếm trong danh sách giao dịch ngân hàng (đã đồng bộ qua Plaid/GoCardLess) để tìm giao dịch khớp với hóa đơn đó.
5.  **Reconciliation:** Nếu khớp, hệ thống tự động đánh dấu giao dịch là "Đã có chứng từ" và lưu trữ hóa đơn vào Vault.

#### C. Luồng Đồng bộ ngân hàng:
1.  Người dùng kết nối ngân hàng qua `apps/engine`.
2.  Định kỳ, `apps/worker` sẽ gọi API của nhà cung cấp (Plaid/Teller) để lấy giao dịch mới.
3.  Giao dịch được tự động phân loại (Categorization) bằng AI dựa trên lịch sử và các quy tắc có sẵn.

---
**Kết luận:** Midday là một dự án có độ phức tạp kỹ thuật cao, tận dụng tối đa sức mạnh của hệ sinh thái TypeScript và AI hiện đại để tự động hóa quy trình quản lý tài chính cho doanh nghiệp nhỏ và freelancers.