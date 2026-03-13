Dưới đây là phân tích chi tiết về dự án **Promptfoo**, một bộ công cụ mã nguồn mở mạnh mẽ dành cho việc kiểm thử (evals) và tấn công thử nghiệm (red teaming) các ứng dụng LLM.

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án được xây dựng trên một hệ sinh thái hiện đại, tối ưu cho hiệu suất và trải nghiệm nhà phát triển:

*   **Ngôn ngữ chính:** **TypeScript (96.9%)**. Việc sử dụng TypeScript giúp đảm bảo tính an toàn về kiểu dữ liệu (type-safety) cho các cấu trúc cấu hình phức tạp của LLM.
*   **Runtime:** **Node.js** (Yêu cầu phiên bản >= 20.20.0).
*   **Giao diện người dùng (UI/UX):**
    *   **Frontend:** React 19, Vite, và Material UI (MUI).
    *   **Terminal UI (TUI):** Đang trong quá trình triển khai **Ink** (React for CLI) để tạo ra các bảng điều khiển tương tác ngay trong Terminal.
*   **Cơ sở dữ liệu & ORM:** **SQLite** (lưu trữ local) kết hợp với **Drizzle ORM** để quản lý kết quả đánh giá và cấu hình.
*   **Hệ thống kiểm thử (Testing):** **Vitest** được dùng cho toàn bộ các cấp độ kiểm thử: Unit, Integration, và Smoke tests.
*   **Công cụ xây dựng (Build tools):** **tsdown** (dựa trên esbuild) để đóng gói mã nguồn siêu tốc.
*   **Phân tích mã nguồn:** **Biome** được sử dụng thay thế cho ESLint/Prettier để tăng tốc độ linting và formatting.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Promptfoo tuân theo các triết lý sau:

*   **Monorepo (Hệ thống đa dự án):** Tổ chức rõ ràng các thành phần:
    *   `src/`: Thư viện lõi và logic CLI.
    *   `src/app/`: Web Dashboard để trực quan hóa kết quả.
    *   `src/providers/`: Các adapter để kết nối với vô số mô hình (OpenAI, Anthropic, Azure, Ollama...).
    *   `src/redteam/`: Logic chuyên biệt cho việc dò tìm lỗ hổng bảo mật.
*   **Kiến trúc Plugin/Provider:** Cho phép mở rộng dễ dàng. Người dùng có thể viết các "Provider" tùy chỉnh bằng Python, JavaScript hoặc Go để kết nối với các API nội bộ.
*   **Privacy-First (Ưu tiên quyền riêng tư):** Toàn bộ quá trình đánh giá chạy 100% tại máy cục bộ (local). Dữ liệu cấu hình và prompts không bao giờ rời khỏi máy người dùng trừ khi họ chủ động sử dụng tính năng "Share".
*   **Declarative Configuration (Cấu hình dạng khai báo):** Mọi thứ (prompts, test cases, assertions) đều được định nghĩa trong file YAML/JSON, giúp việc quản lý phiên bản (Git) trở nên dễ dàng.

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Model-Graded Evaluation:** Sử dụng một LLM mạnh (như GPT-4) để chấm điểm đầu ra của một LLM khác dựa trên các tiêu chí (rubrics) định nghĩa trước.
*   **Red Teaming Strategies:** Tự động hóa các kỹ thuật tấn công như *Prompt Injection*, *PII Leakage*, *Jailbreaking* thông qua các thuật toán tạo dữ liệu đối kháng (adversarial data generation).
*   **Sandwich Parsing:** Kỹ thuật bóc tách dữ liệu từ phản hồi của LLM bằng cách tìm thẻ mở đầu tiên và thẻ đóng cuối cùng, giúp xử lý các phản hồi bị nhiễu hoặc chứa code block.
*   **Dynamic Module Loading:** Sử dụng `import()` động để tải các cấu hình hoặc script tùy chỉnh của người dùng, giúp giảm dung lượng bộ nhớ và tăng tính linh hoạt.
*   **Caching & Caching Migration:** Hệ thống lưu trữ đệm thông minh giúp tránh lặp lại các lời gọi API tốn kém, kèm theo logic migrate cache khi cấu trúc dữ liệu thay đổi.
*   **OIDC Authentication:** Tích hợp xác thực GitHub OIDC cho các luồng CI/CD, cho phép chạy scan bảo mật an toàn mà không cần lưu trữ API keys dài hạn.

---

### 4. Tóm tắt luồng hoạt động (Operational Flow)

Luồng làm việc đặc trưng của một người dùng Promptfoo gồm 4 bước:

1.  **Khởi tạo (Init):** Chạy `promptfoo init` để tạo cấu hình mẫu. Người dùng khai báo mô hình cần test (providers) và danh sách các biến đầu vào (vars).
2.  **Thiết lập Kiểm thử (Configure):**
    *   Định nghĩa các **Assertions** (Xác nhận): Có thể là so sánh chuỗi đơn giản, Regex, hoặc logic JavaScript/Python phức tạp.
    *   Thiết lập **Red Teaming Plugins** nếu mục tiêu là kiểm tra bảo mật.
3.  **Thực thi (Execute - Eval):**
    *   CLI đọc file cấu hình, gửi các yêu cầu song song đến các LLM providers.
    *   Kết quả trả về được đưa qua bộ lọc Assertions để quyết định Pass/Fail.
    *   Dữ liệu được lưu vào SQLite.
4.  **Trực quan hóa (Visualize):**
    *   **CLI Table:** Xem kết quả nhanh ngay tại terminal.
    *   **Web View:** Chạy `promptfoo view` để mở Dashboard React, cho phép so sánh các mô hình theo dạng ma trận (Matrix View), phân tích chi tiết từng trường hợp thất bại.

**Đánh giá chung:** Promptfoo không chỉ là một công cụ test, mà là một nền tảng **LLM-Ops** giúp chuẩn hóa quy trình đưa AI vào sản xuất, từ bước tối ưu hóa prompt đến bước đảm bảo an toàn bảo mật.