Chào bạn, dựa trên cấu trúc thư mục và nội dung các tệp tin cốt lõi của dự án **OpenSandbox** từ Alibaba, dưới đây là phân tích chi tiết về kiến trúc và kỹ thuật của hệ thống này:

---

### 1. Công nghệ cốt lõi (Core Technologies)

OpenSandbox là một hệ sinh thái đa ngôn ngữ, kết hợp giữa quản lý hạ tầng và môi trường thực thi an toàn:

*   **Ngôn ngữ lập trình:**
    *   **Go (Golang):** Dùng cho các thành phần hệ thống hiệu năng cao (Runtime components) như `execd` (daemon thực thi), `ingress`, và `egress` (kiểm soát mạng).
    *   **Python:** Dòng chính cho `server` (FastAPI) – đóng vai trò điều phối (orchestrator) và quản lý vòng đời sandbox.
    *   **Đa ngôn ngữ cho SDK:** Hỗ trợ Python, JavaScript/TypeScript, Java/Kotlin, C#/.NET để AI Agent tích hợp dễ dàng.
*   **Hạ tầng & Containerization:**
    *   **Docker:** Chạy sandbox cục bộ.
    *   **Kubernetes (K8s):** Chạy sandbox ở quy mô lớn với Custom Resource Definitions (CRDs) như `BatchSandbox` và `Pool`.
    *   **Secure Runtimes:** Tích hợp các công nghệ cô lập mạnh như **gVisor**, **Kata Containers**, và **Firecracker** (MicroVM) để ngăn chặn thoát khỏi container (container escape).
*   **Giao thức thực thi:**
    *   **Jupyter Protocol:** Sử dụng nhân Jupyter để thực thi mã Python/Java/JS có trạng thái (stateful).
    *   **SSE (Server-Sent Events):** Truyền phát trực tiếp (streaming) kết quả thực thi mã và log từ sandbox về client theo thời gian thực.
    *   **nftables/iptables:** Kiểm soát lưu lượng mạng ra (egress) dựa trên FQDN (tên miền).

---

### 2. Tư duy Kiến trúc (Architectural Mindset)

Kiến trúc của OpenSandbox được xây dựng theo triết lý **"Sandbox as a Tool"** dành cho AI:

*   **Phân tách Vòng đời và Thực thi (Lifecycle vs. Execution):**
    *   *Lifecycle (Server):* Quản lý việc tạo, xóa, tạm dừng, và gia hạn sandbox.
    *   *Execution (Execd):* Một daemon chạy "bên trong" sandbox để nhận lệnh thực thi mã, thao tác file mà không cần SSH.
*   **Kiến trúc Sidecar & Gateway:**
    *   Sử dụng `egress` sidecar để kiểm soát mạng ở mức hạt nhân (kernel level) ngay trong namespace của sandbox.
    *   Sử dụng `ingress` gateway để định tuyến yêu cầu từ bên ngoài vào đúng sandbox dựa trên ID.
*   **Mô hình Cloud-Native:** Coi sandbox là một tài nguyên Kubernetes tiêu chuẩn, cho phép tận dụng khả năng lập lịch (scheduling) và tự phục hồi của K8s.
*   **Thiết kế Stateless & Stateful:** Hỗ trợ cả lệnh chạy một lần (Shell command) và phiên làm việc duy trì trạng thái (Jupyter Session).

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Kiểm soát Egress dựa trên tên miền (FQDN Egress Control):** Đây là kỹ thuật khó. OpenSandbox sử dụng một DNS Proxy nội bộ (`egress` component). Khi sandbox yêu cầu DNS, proxy sẽ kiểm tra whitelist. Nếu hợp lệ, nó sẽ tự động cập nhật `nftables` (IP set) để cho phép lưu lượng đi qua IP vừa giải mã được.
*   **Quản lý Pool Sandbox (Client-side & Server-side Pooling):** Để giảm độ trễ (latency) khi khởi tạo (thường mất vài giây), hệ thống duy trì các sandbox "nóng" (pre-provisioned) trong một `Pool`, giúp AI Agent lấy được môi trường thực thi trong mili giây.
*   **Chuyển đổi Giao thức (Protocol Translation):** `execd` đóng vai trò cầu nối, chuyển đổi các yêu cầu HTTP/JSON đơn giản từ AI Agent thành các bản tin phức tạp của giao thức Jupyter (ZeroMQ/Websocket) bên trong sandbox.
*   **Hệ thống File ảo & Mount:** Hỗ trợ mount dữ liệu từ nhiều nguồn như OSS (Object Storage), PVC (Kubernetes) hoặc host path vào sandbox, cho phép AI Agent làm việc với tập dữ liệu lớn.
*   **Cơ chế Heartbeat & Tự động hủy:** Mỗi sandbox có một `expires_at`. Nếu client không "renew", server sẽ tự động thu hồi tài nguyên để tránh lãng phí.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Khởi tạo (Provisioning):**
    *   Client (SDK) gửi yêu cầu `create` tới **Server**.
    *   Server gọi Docker API hoặc K8s API để tạo container với image chỉ định (ví dụ: `code-interpreter`).
    *   Container khởi động cùng với **Execd** và **Egress sidecar**.
2.  **Thực thi (Execution):**
    *   Client gửi mã (Python/Bash) tới Server.
    *   Server định tuyến yêu cầu qua **Ingress** tới **Execd** bên trong sandbox.
    *   **Execd** chuyển mã cho **Jupyter Kernel**.
    *   Kết quả thực thi (stdout, stderr, hình ảnh/biểu đồ) được đẩy ngược lại qua **SSE** về client.
3.  **Kiểm soát mạng (Network Control):**
    *   Nếu mã cố gắng truy cập `google.com`, **Egress sidecar** sẽ chặn lại, kiểm tra chính sách (Policy). Nếu nằm trong whitelist, nó mở cổng ở tầng firewall cho phép truy cập.
4.  **Dọn dẹp (Cleanup):**
    *   Sau khi xong việc hoặc hết thời gian (timeout), Server xóa container và giải phóng các tài nguyên mạng/lưu trữ liên quan.

---

### Kết luận
**OpenSandbox** không chỉ là một môi trường chạy code, mà là một **hệ điều hành ảo hóa lớp trên** (Abstraction Layer) giúp các AI Agent như Claude, Gemini hay GPT có thể tương tác với thế giới thực (file, mạng, tính toán) một cách an toàn và có kiểm soát. Việc Alibaba mã nguồn mở dự án này cung cấp một nền tảng quan trọng cho các kỹ sư xây dựng AI Agent thế hệ mới (Coding Agents, GUI Agents).