Dưới đây là bản phân tích chi tiết về dự án **Jaaz** (Canva AI mã nguồn mở) dưới dạng file README tiếng Việt, tập trung vào kiến trúc và kỹ thuật cốt lõi.

---

# Phân Tích Kỹ Thuật Dự Án Jaaz (11cafe/jaaz)

Jaaz là một agent sáng tạo đa phương thức mã nguồn mở dựa trên mô hình "Infinite Canvas" (vô hạn khung vẽ), được thiết kế để thay thế Canva và Manus với ưu tiên hàng đầu là quyền riêng tư và khả năng chạy cục bộ.

## 1. Công Nghệ Cốt Lõi (Core Technologies)

Dự án được xây dựng dựa trên sự kết hợp mạnh mẽ giữa hệ sinh thái Python (AI/Backend) và JavaScript (Desktop/Frontend):

### **Backend (Python & AI Orchestration)**
*   **LangGraph & LangChain:** Sử dụng để điều phối luồng làm việc của các AI Agent (Multi-agent orchestration).
*   **FastAPI:** Framework web hiệu năng cao để xử lý các API truyền tải dữ liệu và quản lý server.
*   **Socket.io (python-socketio):** Đảm bảo giao tiếp hai chiều thời gian thực giữa backend AI và frontend (streaming tokens, trạng thái tool call).
*   **SQLite (aiosqlite):** Lưu trữ cục bộ lịch sử chat, cấu hình workflow và dữ liệu canvas.
*   **Playwright:** Sử dụng để tự động hóa trình duyệt (Browser Automation), cho phép Agent thực hiện các tác vụ như đăng bài lên mạng xã hội (Xiaohongshu, Bilibili).

### **Frontend & UI (React & Canvas)**
*   **Excalidraw:** Thư viện lõi cho phép tạo hệ thống "Infinite Canvas" (khung vẽ vô hạn), hỗ trợ vẽ, chèn ảnh, video và quản lý các phần tử đồ họa.
*   **React & TypeScript:** Xây dựng giao diện người dùng hiện đại, kiểu dữ liệu an toàn.
*   **Zustand:** Quản lý trạng thái (state management) nhẹ nhàng và hiệu quả.
*   **TanStack Query & Router:** Xử lý việc gọi API, cache dữ liệu và điều hướng trang.

### **Desktop App (Electron)**
*   **Electron:** Đóng gói ứng dụng chạy trên Windows/macOS.
*   **IPC (Inter-Process Communication):** Giao tiếp giữa tiến trình hệ thống (Node.js) và tiến trình AI (Python).

---

## 2. Kiến Trúc và Tư Duy Thiết Kế (Architectural Thinking)

### **Mô hình Hybrid Local-Cloud**
Jaaz được thiết kế theo tư duy **Local-first**. Người dùng có thể kết nối với các API mạnh mẽ (OpenAI, Claude, Midjourney) hoặc chạy hoàn toàn cục bộ thông qua **Ollama** và **ComfyUI**.

### **Hệ thống Multi-Agent Swarm**
Kiến trúc AI không chỉ là một bot chat đơn lẻ mà là một nhóm các Agent chuyên biệt:
*   **Planner Agent:** Nhận yêu cầu từ người dùng, phân tích và lập kế hoạch thực hiện.
*   **Image/Video Creator Agent:** Chuyên trách việc gọi các công cụ tạo ảnh/video.
*   **Handoff Mechanism:** Kỹ thuật chuyển giao quyền kiểm soát giữa các Agent (ví dụ: Planner chuyển giao cho Creator sau khi lập kế hoạch xong).

### **Tích hợp Canvas Động**
Thay vì chỉ trả về văn bản, kết quả của AI (ảnh/video) được tiêm trực tiếp vào tọa độ trên canvas. Dự án có thuật toán **`find_next_best_element_position`** để tự động tìm khoảng trống trên khung vẽ nhằm sắp xếp kết quả AI một cách thông minh, tránh đè lên các đối tượng cũ.

---

## 3. Các Kỹ Thuật Chính (Key Techniques)

### **Tự động hóa ComfyUI Workflow**
Dự án có khả năng đọc các file JSON API của ComfyUI, phân tích các đầu vào (input) và tự động tạo ra các **LangChain Tools** động. Điều này cho phép AI Agent hiểu và vận hành các workflow Stable Diffusion phức tạp mà người dùng tự định nghĩa.

### **Xử lý Metadata hình ảnh (PNG Info)**
Khi AI tạo ra hình ảnh, Jaaz ghi đè metadata (prompt, model, seed) vào file PNG. Khi người dùng kéo ảnh cũ vào chat, Agent có thể đọc ngược lại metadata này để hiểu bối cảnh và thực hiện các chỉnh sửa (Image Editing) chính xác.

### **Streaming Bidirectional (Dòng dữ liệu hai chiều)**
Sử dụng WebSocket để stream:
1.  **Văn bản:** Hiệu ứng gõ chữ thời gian thực.
2.  **Trạng thái công cụ:** Hiển thị Agent đang sử dụng công cụ gì (ví dụ: "Đang tạo ảnh...", "Đang suy nghĩ...").
3.  **Tiến trình ComfyUI:** Hiển thị phần trăm (%) tiến độ render ảnh ngay trong giao diện chat.

### **Xác nhận Tool Call (Human-in-the-loop)**
Đối với các tác vụ nhạy cảm hoặc tốn kém (như tạo video dài), hệ thống tích hợp cơ chế `tool_confirmation`, yêu cầu người dùng bấm nút xác nhận trước khi Agent thực hiện hành động.

---

## 4. Luồng Hoạt Động Của Dự Án (Project Workflow)

1.  **Khởi động:** Electron khởi chạy một server Python FastAPI ngầm và một ứng dụng React. Server Python kiểm tra kết nối với các mô hình local (ComfyUI/Ollama) hoặc Cloud API.
2.  **Nhận yêu cầu:** Người dùng nhập yêu cầu qua ChatTextarea (ví dụ: "Thiết kế một poster phim về cyberpunk").
3.  **Lập kế hoạch (Planning):** `Planner Agent` nhận yêu cầu, viết một Design Strategy (Chiến lược thiết kế) và gọi tool `write_plan` để hiển thị các bước cho người dùng thấy.
4.  **Thực thi (Execution):**
    *   Agent chuyển giao (handoff) sang `ImageVideoCreator`.
    *   Creator gọi công cụ tạo ảnh (ví dụ: `generate_image_by_flux`).
    *   Nếu là ComfyUI, server Python gửi lệnh qua WebSocket tới ComfyUI cục bộ.
5.  **Cập nhật Canvas:** Sau khi ảnh được tạo, nó được lưu vào thư mục `user_data/files`. Server gửi một sự kiện `image_generated` qua Socket.io tới frontend.
6.  **Hiển thị:** React nhận sự kiện, gọi Excalidraw API để chèn hình ảnh vào tọa độ đã tính toán và tự động zoom tới vị trí ảnh mới để người dùng xem.
7.  **Hậu kỳ:** Người dùng có thể chọn các ảnh trên canvas, nhấn "Magic Generate" để thực hiện thay đổi dựa trên các phần tử đã chọn.

---

## 5. Kết luận
Jaaz không chỉ là một giao diện cho AI, mà là một **hệ điều hành sáng tạo** thu nhỏ. Kỹ thuật tinh vi nhất của dự án nằm ở cách họ đồng bộ hóa trạng thái giữa một Canvas đồ họa (Excalidraw) và một hệ thống logic AI phức tạp (LangGraph) một cách mượt mà.