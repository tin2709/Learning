Dưới đây là nội dung bài viết về **7 công cụ Open Source cho AI Agent** đã được chuyển đổi thành định dạng **README.md** chuyên nghiệp, phù hợp để lưu trữ làm tài liệu kỹ thuật hoặc lộ trình học tập cho Dev trong kỷ nguyên mới.

---

# 🤖 AI Agent Engineering Stack (2026 Edition)

![AI Agent Banner](https://img.shields.io/badge/Role-AI_System_Architect-blueviolet)
![Open Source](https://img.shields.io/badge/Focus-Open_Source-brightgreen)
![Context](https://img.shields.io/badge/Year-2026-blue)

Chào mừng bạn đến với kỷ nguyên **AI Agent Orchestration**. Khi vai trò của lập trình viên dịch chuyển từ "thợ gõ code" sang "Kiến trúc sư điều phối Agent", việc nắm vững các công cụ mã nguồn mở (OSS) là yếu tố sống còn để xây dựng hệ thống AI mạnh mẽ và ổn định trên Production.

---

## 🚀 Tại sao cần AI Engineering Stack?
Xây dựng ứng dụng AI không chỉ dừng lại ở việc gọi API từ OpenAI hay Anthropic. Để đưa AI lên Production, bạn cần giải quyết các bài toán:
- **Hallucination:** Kiểm soát ảo giác AI.
- **Evaluation:** Đánh giá Prompt dựa trên dữ liệu thay vì "tâm linh".
- **Context Management:** Quản lý bộ nhớ AI hiệu quả và tiết kiệm.
- **Agent Communication:** Giúp các Agent giao tiếp mượt mà.

---

## 🛠️ 7 Công cụ OSS "Must-know"

### 1. [Agency](https://github.com/msitarzewski/agency-agents) —— Multi-Agent Orchestration
*   **Vai trò:** Điều phối đa Agent theo chức danh.
*   **Đặc điểm:** Cung cấp các template role như FE Engineer, BE Engineer, Security, Growth Hacker... Giúp setup nhanh một "công ty AI thu nhỏ" để xử lý task phức tạp.

### 2. [PromptFoo](https://github.com/promptfoo/promptfoo) —— Test-Driven Development cho Prompt
*   **Vai trò:** Đánh giá và A/B Test Prompt.
*   **Đặc điểm:** So sánh output giữa các model, tích hợp vào CI/CD, kiểm tra bảo mật (Prompt Injection). Đưa việc viết prompt vào quy trình kiểm thử nghiêm ngặt.

### 3. [MiroFish](https://github.com/666ghj/MiroFish) —— Data-Driven Decision
*   **Vai trò:** Hệ thống ra quyết định dựa trên dữ liệu.
*   **Đặc điểm:** Crawl dữ liệu thực tế, dựng môi trường mô phỏng và cho các Agent tranh luận để đưa ra chiến lược tối ưu nhất.

### 4. [Impeccable](https://github.com/impeccable-ai/impeccable) —— UI/UX Enhancer cho AI
*   **Vai trò:** Tối ưu hóa Front-end do AI sinh ra.
*   **Đặc điểm:** Cung cấp các lệnh như `distill` (làm gọn UI), `colorize` (phủ màu thương hiệu), `animate` (nhúng interaction). Biến giao diện "phèn" của AI thành sản phẩm cao cấp.

### 5. [OpenViking](https://github.com/volcengine/OpenViking) —— Context OS
*   **Vai trò:** Quản lý cấu trúc "Não bộ" (Context) của AI.
*   **Đặc điểm:** Load context theo tầng để tiết kiệm token, tự động nén thông tin rác. Đây là lớp dữ liệu AI-native thay thế các cơ chế memory nặng nề truyền thống.

### 6. [Heretic](https://github.com/p-e-w/heretic) —— Uncensored Model Explorer
*   **Vai trò:** Giải phóng giới hạn của LLM.
*   **Đặc điểm:** Bypass các rào cản an toàn mặc định cho các task nghiên cứu đặc thù hoặc hệ thống nội bộ yêu cầu tự do tuyệt đối. *Lưu ý: Cần sử dụng thận trọng.*

### 7. [NanoChat](https://github.com/karpathy/nanochat) —— SLM End-to-End
*   **Vai trò:** Xây dựng Small Language Model từ con số 0.
*   **Đặc điểm:** Từ Tokenization, Pre-training đến Fine-tuning. Giúp Dev sở hữu model riêng cho dự án với chi phí thấp và tài nguyên GPU hạn chế.

---

## 🏗️ Kiến trúc AI Application Stack 2026

| Tầng (Layer) | Công cụ tiêu biểu |
| :--- | :--- |
| **Agent Layer** | Agency |
| **Evaluation Layer** | PromptFoo |
| **Decision Layer** | MiroFish |
| **UI Layer** | Impeccable |
| **Context Layer** | OpenViking |
| **Model Layer** | NanoChat / Heretic |
| **Integration Layer** | API Management (Apidog, v.v...) |

---

## ⚖️ 3 Luật chơi mới cho Developer

1.  **Từ "Thợ gõ" sang "Kiến trúc sư":** Tập trung thiết kế luồng data, phân chia task cho Agent và quản lý tài nguyên.
2.  **Đo lường thay vì dự đoán:** Tuyệt đối không dùng prompt nếu không có hệ thống Evaluation đánh giá tỉ lệ thành công.
3.  **Ưu tiên Context Design:** Tầng Context (dữ liệu cho AI đọc) quan trọng hơn kích thước của Model.

---

## ⚠️ 3 Sai lầm "Chết người" cần tránh

*   **Spam Agent:** Đẻ quá nhiều Agent gây rối loạn hệ thống và lãng phí token.
*   **Thần thánh hóa Prompt:** Prompt hay đến đâu cũng vô dụng nếu hạ tầng API bên dưới không ổn định (Timeout, sai format JSON).
*   **Bỏ quên hạ tầng API:** AI Agent mạnh mẽ nhất cũng sẽ thất bại nếu gọi API bị lỗi. Hãy sử dụng các công cụ quản trị API (như Apidog) để đảm bảo độ tin cậy.

---
*Tài liệu tổng hợp dựa trên bài viết của Quang @ Viblo. Hãy share và đóng góp nếu bạn thấy hữu ích!*