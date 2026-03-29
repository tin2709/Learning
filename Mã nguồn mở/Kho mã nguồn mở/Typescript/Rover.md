Dưới đây là phân tích chi tiết về dự án **Rover** (phát triển bởi RTRVR AI) dựa trên cấu trúc thư mục và mã nguồn bạn đã cung cấp.

---

### 1. Công nghệ cốt lõi (Technology Core)

Rover là một nền tảng **Autonomous Web Agent** thế hệ mới, khác biệt hoàn toàn với các chatbot hay screenshot-agent thông thường.

*   **TypeScript-Native:** Dự án được viết gần như hoàn toàn bằng TypeScript (92.5%), đảm bảo tính chặt chẽ về kiểu dữ liệu cho một hệ thống phức tạp.
*   **DOM-Native & A11y Tree:** Thay vì sử dụng hình ảnh (screenshots) và thị giác máy tính (Computer Vision) chậm chạp, Rover xây dựng một **Accessibility Tree** (Cây truy cập) dựa trên tiêu chuẩn W3C. Đây là "đôi mắt" giúp LLM hiểu cấu trúc trang web một cách chính xác và nhẹ nhất.
*   **Web Workers & MessageChannel:** Toàn bộ logic "suy nghĩ" của Agent được chạy trong một **Web Worker** tách biệt. Việc giao tiếp giữa Worker và Main Thread (UI/DOM) được thực hiện qua **MessageChannel RPC**, giúp trang web của khách hàng không bị giật lag (main-thread stay responsive).
*   **Google Gemini LLM:** Dựa trên các tiện ích trong `shared/geminiUtils.ts`, dự án tối ưu hóa việc sử dụng mô hình Gemini để lập kế hoạch (planning) và suy luận.
*   **Shadow DOM:** Widget UI của Rover được bọc trong Shadow DOM để đảm bảo không bị ảnh hưởng bởi CSS/JS của trang web gốc (host site) và ngược lại.
*   **Model Context Protocol (MCP) & Agent Task Protocol (ATP):** Hỗ trợ các chuẩn giao tiếp mới nhất giữa AI và các công cụ/nhiệm vụ trên web.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Rover được chia thành hai mặt phẳng (Planes) rõ rệt:

*   **Runtime Plane (Mặt phẳng thực thi):** Chạy trực tiếp trong trình duyệt người dùng.
    *   **SDK:** Điểm khởi đầu, quản lý vòng đời của Agent.
    *   **Worker:** Trái tim của Agent, nơi lập kế hoạch.
    *   **Bridge:** Cầu nối trung gian để thực hiện các hành động DOM thực tế.
*   **Owner Plane (Mặt phẳng sở hữu):** Rover Workspace dành cho chủ sở hữu trang web để cấu hình, xem analytics, trajectories (lộ trình di chuyển của Agent) và quản lý bộ nhớ (memory).

**Kiến trúc Monorepo:** Sử dụng `pnpm-workspace` để quản lý rất nhiều gói nhỏ chuyên biệt:
*   `packages/a11y-tree`: Chuyên trách tính toán ngữ nghĩa trang web.
*   `packages/dom`: Thực thi các hành động (click, type, scroll).
*   `packages/roverbook`: Lớp phân tích (AX layer) ghi lại mọi hoạt động của Agent.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Queue-based Stub Loading:** Kỹ thuật thường thấy ở các thư viện lớn (như Google Analytics). Rover sử dụng một hàm nhỏ để xếp hàng (queue) các lệnh gọi (`rover('boot', ...)`) ngay cả khi SDK chính chưa tải xong, đảm bảo không bỏ lỡ dữ liệu.
*   **Tiered Agent Identity Attribution:** Rover phân cấp độ tin cậy của danh tính Agent:
    1.  *Verified:* Xác thực mạnh.
    2.  *Self-reported:* Do Agent tự khai báo qua metadata.
    3.  *Heuristic:* Suy luận qua User-Agent hoặc chữ ký header.
    4.  *Anonymous:* Vô danh.
*   **Execution Guardrails:** Kỹ thuật bảo mật lớp thực thi. Rover kiểm soát chặt chẽ phạm vi tên miền (`domainScopeMode`) và chính sách điều hướng (`externalNavigationPolicy`) để ngăn Agent thực hiện các hành động độc hại hoặc ngoài ý muốn.
*   **Cloud Checkpointing:** Trạng thái của phiên làm việc (Session) được đồng bộ liên tục lên Cloud. Điều này cho phép người dùng F5 trang web hoặc chuyển tab mà Agent vẫn tiếp tục nhiệm vụ mà không bị "mất trí nhớ".
*   **Semantic Targeting:** Thay vì dùng CSS Selector (rất dễ hỏng khi web cập nhật), Rover dùng các nhãn ngữ nghĩa từ A11y Tree để xác định đối tượng (ví dụ: "nút Thanh toán" thay vì `div > button.btn-0123`).

---

### 4. Luồng hoạt động hệ thống (System Workflow)

Luồng xử lý của Rover diễn ra theo chu kỳ **Observe-Plan-Act**:

1.  **Khởi tạo (Boot):** Người dùng nhúng script tag. SDK khởi tạo Bridge và Worker.
2.  **Tiếp nhận nhiệm vụ (Ingress):** Nhận prompt qua UI hoặc qua `POST /v1/tasks` (Agent Task Protocol).
3.  **Quan sát (Observation):**
    *   `a11y-tree` quét DOM, tạo snapshot ngữ nghĩa.
    *   `instrumentation` ghi nhận các event listener hiện có trên trang.
4.  **Lập kế hoạch (Planning):**
    *   Snapshot được gửi từ Bridge sang Worker.
    *   Worker gọi Backend (nơi có LLM) để phân tích snapshot và mục tiêu, sau đó trả về một "Action" (ví dụ: `click_element(id=42)`).
5.  **Thực thi (Execution):**
    *   Worker gửi lệnh action qua MessageChannel về Bridge.
    *   Bridge (chạy ở Main Thread) thực hiện hành động trên DOM thực.
6.  **Ghi chép (AX Logging):**
    *   `roverbook` ghi lại sự kiện, kết quả của action, và sự thay đổi trạng thái vào Firestore.
    *   Chủ sở hữu có thể xem lại toàn bộ quá trình này dưới dạng "Trajectory" (video replay hoặc log).

---

### Tổng kết
**Rover** là một dự án có kỹ thuật cực cao, giải quyết bài toán "Web Automation cho AI" bằng cách đi sâu vào tầng ngữ nghĩa (Accessibility) và xử lý bất đồng bộ (Web Workers). Nó không chỉ đơn thuần là một tool tự động hóa mà là một hệ điều hành nhỏ dành riêng cho AI Agent chạy trên nền tảng Web.