Dưới đây là bản **README.md** được thiết kế chi tiết, tóm tắt trọn vẹn kiến thức chuyên sâu về **Multi-Agent Patterns (MAP)** dựa trên bài viết của tác giả Nguyễn Hoàng từ Viblo.

---

# 🤖 Multi-Agent Patterns: Kiến trúc & Chiến lược Phối hợp AI Agents

![AI Agents](https://img.shields.io/badge/Tech-Agentic_AI-blue)
![Architecture](https://img.shields.io/badge/Focus-Multi--Agent_Patterns-orange)
![LLM](https://img.shields.io/badge/LLM-GPT_|_Claude_|_Grok-brightgreen)

Tài liệu này cung cấp cái nhìn tổng quan về các mẫu thiết kế (Patterns) để tổ chức và điều phối nhiều AI Agent cùng làm việc, giúp giải quyết các nhiệm vụ phức tạp mà một Agent đơn lẻ không thể đảm đương hiệu quả.

---

## 📖 1. Phân biệt MAS và MAP

| Thuật ngữ | Định nghĩa | Ví dụ |
| :--- | :--- | :--- |
| **Multi-Agent System (MAS)** | Là toàn bộ hệ sinh thái gồm nhiều Agent, bộ nhớ (Memory), công cụ (Tools) và lớp điều phối. | Đội ngũ chuyên gia AI (Researcher, Writer, Reviewer). |
| **Multi-Agent Pattern (MAP)** | Là mẫu thiết kế kiến trúc quy định **cách sắp xếp** và **giao tiếp** giữa các Agent. | Sơ đồ tổ chức, quy trình làm việc (Workflow). |

> **Nói đơn giản:** Nếu MAS là "đội ngũ chuyên gia", thì MAP là "cách tổ chức đội ngũ đó" để hoạt động hiệu quả và tin cậy.

---

## 📐 2. Mô hình Tư duy 3 Chiều (Decision Framework)

Để chọn lựa Pattern phù hợp, cần xem xét 3 trục cốt lõi:

1.  **Trục Dọc (Y) - Quyền Quyết định:** Ai quyết định bước tiếp theo? (AI tự quyết định vs. Developer định nghĩa flow).
2.  **Trục Ngang (X) - Độ phức tạp:** Nhiệm vụ đơn giản, lặp lại vs. Nhiệm vụ phức tạp, cần adaptive.
3.  **Chiều Sâu (Z) - Context Sharing:** Các Agent biết gì về nhau? (Chia sẻ toàn bộ lịch sử chat vs. Chỉ truyền dữ liệu cần thiết).

---

## 🏗️ 3. Các Multi-Agent Pattern Phổ biến

| Pattern | Mô tả | Ưu điểm | Phù hợp nhất với |
| :--- | :--- | :--- | :--- |
| **Sequential** | Chạy tuần tự: A → B → C | Dễ kiểm soát, rẻ, dễ audit. | Viết báo cáo, pipeline nội dung. |
| **Parallel** | Chạy song song nhiều Agent | Nhanh, đa góc nhìn. | Nghiên cứu đa nguồn, đánh giá rủi ro. |
| **Hierarchical** | Supervisor điều phối các Worker | Kiểm soát tốt, linh hoạt cao. | Support khách hàng, phần mềm phức tạp. |
| **Swarm** | Các Agent tự hand-off nhiệm vụ | Cực kỳ linh hoạt, sáng tạo. | Brainstorming, nghiên cứu khám phá. |
| **Graph** | Node & Edges có điều kiện | Hỗ trợ vòng lặp (Cycles), logic phức tạp. | Quy trình cần feedback loop, sửa lỗi lặp lại. |

---

## 🔍 4. So sánh Chi tiết 3 Pattern Chủ chốt

### 4.1 Sequential Pattern (Workflow)
*   **Đặc điểm:** Tuyến tính, không có vòng lặp, context được cắt tỉa (curated context).
*   **Cơ chế:** Mỗi Agent chỉ nhận output của bước trước đó.
*   **Khi nào dùng:** Cần sự chính xác tuyệt đối, tiết kiệm token, quy trình không đổi.

### 4.2 Graph Pattern (Stateful Graph)
*   **Đặc điểm:** Linh hoạt hơn Sequential, hỗ trợ chạy song song và quay lại bước trước (Iterative loop).
*   **Cơ chế:** Sử dụng `Shared State` (một từ điển dùng chung) để các Agent đọc/ghi dữ liệu.
*   **Khi nào dùng:** Cần kiểm soát luồng đi dựa trên điều kiện (Conditional routing) và có cơ chế sửa lỗi (Retry logic).

### 4.3 Swarm Pattern (Decentralized)
*   **Đặc điểm:** Phi tập trung, không có Orchestrator cứng. Agent dùng công cụ `handoff_to_agent` để chuyển việc.
*   **Cơ chế:** Sử dụng `Shared Transcript` (toàn bộ lịch sử hội thoại). Token cost tăng rất nhanh.
*   **Khi nào dùng:** Các nhiệm vụ mở (Open-ended), không biết trước cần bao nhiêu bước hoặc cần chuyên gia nào.

---

## 💡 5. Lời khuyên khi lựa chọn

1.  **Chọn theo bài toán, không chọn theo Framework:** Đừng chọn vì CrewAI hay LangGraph dễ dùng, hãy chọn dựa trên yêu cầu vận hành.
2.  **Quản lý Context là then chốt:** Chiều sâu (Context sharing) ảnh hưởng trực tiếp đến chi phí Token và khả năng Debug.
3.  **Bắt đầu đơn giản:** Luôn bắt đầu bằng **Sequential** hoặc **Graph** đơn giản trước khi chuyển sang **Swarm** đầy rủi ro và tốn kém.

---
*Tài liệu tóm tắt từ nội dung chuyên sâu về Kiến trúc AI Agent 2026.*