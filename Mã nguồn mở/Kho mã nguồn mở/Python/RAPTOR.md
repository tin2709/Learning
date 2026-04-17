Dựa trên mã nguồn và tài liệu của dự án **RAPTOR** (Recursive Autonomous Penetration Testing and Observation Robot), dưới đây là phân tích chi tiết về hệ thống này:

---

### 1. Công nghệ cốt lõi (Core Technologies)

RAPTOR là một khung (framework) lai ghép cực kỳ mạnh mẽ giữa các công cụ bảo mật truyền thống và trí tuệ nhân tạo (LLM).

*   **Ngôn ngữ lập trình:** **Python 3** đóng vai trò là "nhạc trưởng" điều phối (orchestration), kết hợp với các script **Bash** để quản lý vòng đời hệ thống.
*   **Công cụ phân tích tĩnh (SAST):**
    *   **Semgrep:** Dùng để quét nhanh các mẫu mã nguồn lỗi phổ biến (ruleset tùy chỉnh nằm trong `engine/semgrep/rules`).
    *   **CodeQL:** Dùng để phân tích luồng dữ liệu (dataflow) sâu và biến đổi mã nguồn thành cơ sở dữ liệu để truy vấn các lỗ hổng phức tạp.
*   **Công cụ Fuzzing & Debugging:**
    *   **AFL++ (American Fuzzy Lop):** Công cụ chính để tìm kiếm lỗi crash trong các file thực thi (binary).
    *   **rr (Record-Replay Debugger):** Dùng cho phân tích pháp y lỗi crash (crash forensics), cho phép quay ngược thời gian thực thi của tiến trình.
    *   **GDB/LLDB:** Các công cụ gỡ lỗi tiêu chuẩn để trích xuất ngữ cảnh khi xảy ra lỗi.
*   **Hệ sinh thái AI (LLM):**
    *   **Claude Code:** Nền tảng gốc mà RAPTOR dựa vào.
    *   **Direct SDKs:** Tích hợp trực tiếp Anthropic (Claude), OpenAI (GPT-4), Google (Gemini/Gemma) và Mistral.
    *   **Ollama:** Hỗ trợ chạy các mô hình AI cục bộ (như DeepSeek-R1) để phân tích offline.
*   **Phân tích mã nguồn:** **tree-sitter** được sử dụng để xây dựng Inventory (danh mục) mã nguồn, trích xuất metadata của hàm, lớp, và các annotation bảo mật.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của RAPTOR được xây dựng theo mô hình **"Agentic Workflow" (Luồng công việc tự trị)**:

*   **Phân tầng chỉ thị (Tiered Instructions):** Dự án sử dụng hệ thống "Progressive Loading". Thông tin được nạp dần dần từ `CLAUDE.md` (cơ bản) đến các `Personas` (chuyên gia cụ thể) để tối ưu hóa context window của mô hình AI.
*   **Hợp nhất Tĩnh và Động:** RAPTOR không chỉ tìm lỗi bằng công cụ quét (tĩnh), mà còn cố gắng xác thực lỗi đó bằng cách tạo ra mã khai thác PoC (động) hoặc phân tích crash thực tế.
*   **Tính module hóa tuyệt đối (Modular Packages):** Mọi khả năng (fuzzing, codeql, web, forensics) đều nằm trong thư mục `packages/`. Điều này cho phép mở rộng hệ thống mà không phá vỡ logic lõi của `raptor.py`.
*   **Cơ chế "Bridge" (Cầu nối kiến thức):** File `understand_bridge.py` cho thấy tư duy kết nối: Kết quả của quá trình "Hiểu mã" (`/understand`) sẽ tự động được dùng làm dữ liệu đầu vào cho quá trình "Xác thực" (`/validate`), giúp AI không phải đọc lại từ đầu.
*   **An toàn hệ thống (Security Hardening):** Dự án có cơ chế kiểm tra các file `.claude/settings.json` độc hại trong repo mục tiêu để tránh bị tấn công ngược (Command Injection) khi nhà nghiên cứu quét mã lạ.

---

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **Structured Output (Đầu ra cấu trúc):** Sử dụng thư viện `instructor` kết hợp với **Pydantic** để ép LLM trả về dữ liệu định dạng JSON nghiêm ngặt, giúp máy tính có thể xử lý tiếp các bước sau (như tự động vá lỗi).
*   **Phân tách luồng logic:** 
    *   `raptor_agentic.py` xử lý luồng song song (Parallel Orchestration) bằng cách đẩy nhiều sub-agents chạy cùng lúc để phân tích hàng loạt lỗ hổng.
    *   Sử dụng `subprocess.Popen` với kỹ thuật streaming để hiển thị log thời gian thực từ các công cụ bảo mật nặng nề lên màn hình người dùng.
*   **Hệ thống Inventory & Checksum:** Mỗi file mã nguồn đều được băm (SHA-256). Kỹ thuật này giúp hệ thống nhận diện được file nào đã thay đổi để chỉ quét lại những phần cần thiết (Incremental Scanning).
*   **Xử lý SARIF (Static Analysis Results Interchange Format):** RAPTOR coi SARIF là ngôn ngữ chung để hợp nhất kết quả từ Semgrep, CodeQL và các công cụ khác về một mối.
*   **Quản lý tài chính (Cost/Budget Management):** Tính toán chi phí LLM theo thời gian thực, cho phép đặt giới hạn ngân sách (max_cost) để tránh việc AI tự chạy tốn kém hàng ngàn USD.

---

### 4. Luồng hoạt động của hệ thống (System Workflow)

Một chu trình phân tích điển hình của RAPTOR (đặc biệt trong chế độ `/agentic`) diễn ra như sau:

1.  **Giai đoạn Khởi tạo (Discovery & Inventory):**
    *   Tạo bản đồ mã nguồn (`build_inventory.py`), đếm SLOC (số dòng code), phân loại hàm/lớp bằng `tree-sitter`.
2.  **Giai đoạn Quét (Scanning - Phase 1):**
    *   Chạy Semgrep và CodeQL song song.
    *   Hợp nhất các kết quả thô vào định dạng SARIF chung.
3.  **Giai đoạn Khử nhiễu & Xác thực (Validation & Dedup - Phase 2):**
    *   Dùng thuật toán loại bỏ các kết quả trùng lặp hoặc sai số (False Positives).
    *   Nếu là lỗi binary, thực hiện phân tích tính khả thi (Mitigation check: ASAN, Stack Canaries...).
4.  **Giai đoạn Phân tích sâu (Autonomous Analysis - Phase 3):**
    *   AI đọc luồng dữ liệu (Dataflow) liên quan đến lỗi.
    *   Sử dụng "Adversarial Thinking" để đánh giá tác động (Impact) và khả năng khai thác thực tế.
5.  **Giai đoạn Khai thác & Vá lỗi (Exploit & Patch - Phase 4):**
    *   AI viết mã khai thác PoC (thường bằng Python hoặc C).
    *   AI đề xuất các bản vá mã nguồn để sửa lỗi.
6.  **Giai đoạn Báo cáo (Reporting):**
    *   Xuất báo cáo chi tiết dưới dạng Markdown.
    *   Tự động vẽ sơ đồ luồng dữ liệu (Mermaid diagrams) để người dùng dễ hình dung.

### Tổng kết
RAPTOR là một bước tiến từ "công cụ bảo mật" sang "đồng nghiệp bảo mật AI". Nó không chỉ là một launcher cho Semgrep hay AFL++, mà là một hệ thống có **trí nhớ (Fuzzing Memory)**, có **tư duy ưu tiên (Risk-based Prioritization)** và có khả năng **tự động hóa toàn bộ chuỗi tấn công/phòng thủ**.