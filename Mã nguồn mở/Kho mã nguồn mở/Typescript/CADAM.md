Dựa trên mã nguồn của dự án **CADAM**, dưới đây là phân tích chi tiết về các khía cạnh công nghệ, kiến trúc và kỹ thuật của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án là một sự kết hợp hiện đại giữa Web-based CAD và Generative AI:

*   **Frontend Framework:** React 19 kết hợp với TypeScript và Vite. Sử dụng **Shadcn/UI** và **Tailwind CSS** để xây dựng giao diện chuyên nghiệp, phản hồi nhanh.
*   **3D Rendering:** 
    *   **Three.js & React Three Fiber (R3F):** Engine chính để hiển thị mô hình 3D trong trình duyệt.
    *   **OpenSCAD WASM:** Đây là "linh hồn" của phần Parametric. Toàn bộ engine OpenSCAD được biên dịch sang WebAssembly để chạy trực tiếp trên trình duyệt của người dùng mà không cần server xử lý file CAD.
*   **Backend (BaaS):** 
    *   **Supabase:** Quản lý cơ sở dữ liệu (PostgreSQL), xác thực (Auth), lưu trữ file (Storage), và các hàm serverless (Edge Functions).
    *   **Deno:** Chạy các Edge Functions phía backend với hiệu suất cao.
*   **AI Models:**
    *   **Anthropic (Claude 3.5 Sonnet/Haiku):** Sử dụng cho tư duy logic, viết code OpenSCAD và tạo prompt.
    *   **Google Gemini 3.1 & Fal.ai (Flux):** Xử lý hình ảnh và đa phương thức.
    *   **Tripo/Meshy (via Fal.ai):** Engine tạo Mesh 3D từ text/image (Creative Mode).
*   **Infrastructure:** Stripe (Thanh toán), PostHog (Phân tích), Sentry (Giám sát lỗi).

### 2. Tư duy Kiến trúc (Architectural Mindset)

Kiến trúc của CADAM thể hiện tư duy **"Hybrid Web-Agent"**:

*   **Kiến trúc Client-Side Heavy:** Việc đưa OpenSCAD WASM vào trình duyệt giúp giảm tải cho server và quan trọng nhất là tạo ra trải nghiệm chỉnh sửa tham số (parametric) theo thời gian thực (Zero-latency feedback).
*   **Cơ chế Agentic Workflow:** Backend không chỉ trả về kết quả tĩnh. Nó hoạt động như một AI Agent có khả năng gọi công cụ (Tool Calling). Ví dụ: `build_parametric_model` hoặc `apply_parameter_changes`. AI có thể tự quyết định khi nào cần viết lại code mới, khi nào chỉ cần sửa tham số.
*   **Cấu trúc dữ liệu dạng Cây (Tree-based Messages):** File `shared/Tree.ts` cho thấy hệ thống quản lý tin nhắn theo dạng nhánh. Điều này cho phép người dùng "thử sai" - quay lại một phiên bản cũ và rẽ nhánh sang một hướng thiết kế khác mà không mất dữ liệu cũ.
*   **Kiến trúc hướng sự kiện (Event-Driven):** Sử dụng Supabase Realtime để đồng bộ trạng thái. Khi một mesh 3D nặng được tạo xong ở backend thông qua Webhook, frontend sẽ nhận được thông báo ngay lập tức để cập nhật UI mà không cần refresh.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Web Workers & Multi-threading:** Dự án sử dụng Web Workers (`src/worker/worker.ts`) để chạy engine OpenSCAD. Kỹ thuật này giúp việc biên dịch các khối hình học phức tạp không gây đứng (block) luồng xử lý giao diện chính (UI Thread).
*   **Optimistic Updates:** Sử dụng React Query để thực hiện "cập nhật lạc quan". Khi người dùng kéo slider thay đổi kích thước, UI sẽ thay đổi ngay lập tức, trong khi lệnh update được gửi ngầm xuống database.
*   **Geometry Processing:**
    *   **STL Parsing & Bounding Box:** Kỹ thuật tính toán kích thước mô hình để AI có thể đặt các vật thể khác lên trên chính xác (ví dụ: đặt cái mũ lên đầu một bức tượng).
    *   **Mesh Repair:** Các hàm xử lý hình học (`meshPrintProcessUtils.ts`) để đảm bảo mô hình tạo ra là "Watertight" (kín nước) - điều kiện tiên quyết để in 3D được.
*   **Custom Shaders (GLSL):** Sử dụng shader tùy chỉnh (`points.vert`, `points.frag`) để tạo hiệu ứng "tan biến/hòa quyện" các hạt điểm khi mô hình đang trong quá trình AI xử lý, tạo cảm giác mượt mà và nghệ thuật.
*   **Atomic Token Deductions:** Sử dụng PostgreSQL Functions (RPC) để đảm bảo việc trừ token của người dùng diễn ra chính xác (ACID), tránh trường hợp race condition khi người dùng nhấn tạo nhiều lần.

### 4. Luồng hoạt động hệ thống (System Workflow)

Hệ thống chia làm 2 luồng chính (Modes):

#### Luồng A: Parametric Mode (Dựa trên Code)
1.  **Input:** Người dùng nhập: "Tạo một cái hộp 50x50mm có nắp".
2.  **AI Processing:** Claude nhận lệnh -> Trả về code OpenSCAD có tham số (variables).
3.  **WASM Execution:** Frontend nhận code -> Đẩy vào Web Worker -> OpenSCAD WASM biên dịch ra file STL (dạng Blob).
4.  **Rendering:** Three.js lấy STL Blob đó hiển thị lên màn hình.
5.  **Iteration:** Người dùng kéo slider -> Tham số thay đổi -> WASM biên dịch lại -> Mô hình cập nhật ngay lập tức.

#### Luồng B: Creative Mode (Dựa trên Mesh)
1.  **Input:** Người dùng nhập: "Một con rồng phong cách Origami" hoặc upload ảnh.
2.  **Image Gen:** Hệ thống dùng Flux/Gemini tạo ra ảnh tham chiếu 4 góc nhìn (Multiview).
3.  **Mesh Gen:** Gửi ảnh này sang API tạo 3D (Tripo/Meshy).
4.  **Webhook:** Khi API xử lý xong (mất 1-5 phút), nó gọi về `fal-webhook`.
5.  **Storage & Sync:** Backend lưu file `.glb` vào Supabase Storage -> Bắn tin hiệu qua Realtime -> Frontend tải file về và hiển thị.

### Tổng kết
CADAM là một dự án có độ phức tạp kỹ thuật cao, thể hiện sự am hiểu sâu sắc về cả **WebAssembly** lẫn **AI Orchestration**. Nó chuyển đổi khái niệm CAD từ "vẽ tay" sang "điều khiển bằng ngôn ngữ" nhưng vẫn giữ được độ chính xác kỹ thuật nhờ engine tham số hóa chạy ở phía client.