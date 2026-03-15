Chào bạn, đây là bản phân tích chi tiết về dự án **waoowaoo**, một nền tảng "Studio AI" dành cho sản xuất phim và video ngắn chuyên nghiệp. Dự án này thể hiện một trình độ hoàn thiện cực cao về mặt kỹ thuật, đặc biệt là trong việc điều phối các luồng AI phức tạp (AI Agentic Workflows).

---

### 1. Công nghệ cốt lõi (Core Technology Stack)

Dự án sử dụng các công nghệ hiện đại nhất (Bleeding edge) hiện nay:

*   **Frontend & Framework:** **Next.js 15 (App Router)** kết hợp với **React 19**. Đây là phiên bản mới nhất, tối ưu hóa mạnh mẽ cho việc xử lý Server Components và streaming dữ liệu. Giao diện được xây dựng bằng **Tailwind CSS v4** với phong cách **Glassmorphism** (Glass Surface).
*   **Quản lý dữ liệu:** **MySQL** là cơ sở dữ liệu chính, được vận hành thông qua **Prisma ORM**.
*   **Hệ thống hàng đợi (Background Jobs):** Sử dụng **Redis** và **BullMQ**. Đây là phần quan trọng nhất vì các tác vụ AI (sinh ảnh, sinh video) thường mất nhiều thời gian, cần xử lý bất đồng bộ để tránh treo server.
*   **AI Orchestration:** Tích hợp trực tiếp SDK của **OpenAI, Google (Gemini), Fal.ai, Ark (Volcano Engine)**. Đặc biệt có sự xuất hiện của **LangGraph** để xây dựng các kịch bản AI Agent có khả năng tự suy luận và lặp lại (loops).
*   **Lưu trữ (Storage):** Hỗ trợ đa nền tảng qua S3-compatible API (MinIO), Tencent COS hoặc lưu trữ nội bộ (Local).
*   **Đồ họa & Video:** Sử dụng **Sharp** để xử lý ảnh và **Remotion** để biên tập/render video bằng React code.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án không chỉ là một ứng dụng web đơn giản mà là một **AI Agent Platform** có tư duy kiến trúc phân tầng:

*   **Kiến trúc Model Gateway:** Tọa lạc tại `src/lib/model-gateway`. Thay vì gọi trực tiếp API của OpenAI hay Gemini, hệ thống đi qua một lớp Router. Điều này cho phép người dùng cấu hình linh hoạt (Bring Your Own Key) và chuyển đổi giữa các nhà cung cấp khác nhau mà không làm thay đổi logic nghiệp vụ.
*   **Tư duy Asynchronous-First:** Mọi hành động "nặng" đều được chuyển thành `Task`. Trạng thái của Task được đồng bộ hóa thời gian thực giữa Worker (xử lý dưới nền) và UI thông qua **SSE (Server-Sent Events)**.
*   **Cấu trúc Prompt-Driven:** Toàn bộ "tri thức" của hệ thống nằm trong `lib/prompts`. Các prompt được tách rời thành các file `.txt` riêng biệt cho từng ngôn ngữ (zh/en), giúp việc bảo trì và tối ưu hóa câu lệnh cho AI trở nên khoa học.
*   **Hệ thống Guard & Migration mạnh mẽ:** Thư mục `scripts/guards` chứa các script kiểm tra hợp đồng dữ liệu (data contract), đảm bảo logic giữa các bước (từ Novel -> Script -> Storyboard) không bao giờ bị sai lệch về mặt cấu trúc JSON.

---

### 3. Các kỹ thuật chính (Key Technical Techniques)

*   **Character Consistency (Nhất quán nhân vật):** Kỹ thuật "Character Reference" (tại `lib/prompts/character-reference`). AI phân tích ảnh gốc thành mô tả văn bản chi tiết, sau đó dùng mô tả này làm "anchor" cho tất cả các phân cảnh sau đó để giữ khuôn mặt/trang phục nhân vật không bị thay đổi (một bài toán khó trong AI Video).
*   **Smart Text Splitting:** Sử dụng AI để nhận diện cấu trúc chương hồi của tiểu thuyết, tự động chia nhỏ thành các `Episodes` và `Clips` dựa trên ngữ cảnh thay vì chỉ đếm số chữ.
*   **Cinematography Planning (Quy hoạch phối cảnh):** Có một Agent riêng (`agent_cinematographer`) chuyên trách việc tạo ra các quy tắc về góc máy, ánh sáng và bố cục cho từng khung hình dựa trên nội dung kịch bản.
*   **Optimistic UI:** Khi người dùng thay đổi hoặc tạo mới, hệ thống sử dụng các mutation hooks để cập nhật giao diện ngay lập tức trong khi chờ API phản hồi, tạo cảm giác mượt mà.

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Hệ thống vận hành theo một dây chuyền sản xuất công nghiệp:

1.  **Giai đoạn Novel Input:** Người dùng nhập văn bản thô hoặc upload tài liệu. AI Agent thực hiện phân tích cấu trúc kịch bản.
2.  **Giai đoạn Asset Analysis:** Hệ thống tự động trích xuất danh sách nhân vật và bối cảnh. Người dùng xác nhận "hồ sơ" (Profile) cho từng tài sản.
3.  **Giai đoạn Storyboarding:**
    *   Chuyển kịch bản thành các khung hình (Panels).
    *   Tự động sinh Prompt hình ảnh và Prompt chuyển động (Video Prompt).
    *   Tạo chỉ dẫn diễn xuất (Acting direction) và quy tắc nhiếp ảnh.
4.  **Giai đoạn Generation:**
    *   Sinh ảnh hàng loạt cho các Panels.
    *   Hỗ trợ "Image Edit" để sửa đổi cục bộ các chi tiết không ưng ý.
    *   Sinh video từ ảnh (Image-to-Video).
5.  **Giai đoạn Audio & Sync:** Tổng hợp giọng nói (TTS) -> Thực hiện khớp khẩu hình (Lip Sync) giữa âm thanh và video nhân vật.
6.  **Giai đoạn Editor:** Sử dụng AI Editor để ghép nối các đoạn clip, thêm hiệu ứng chuyển cảnh và xuất thành phẩm cuối cùng.

### Kết luận
**waoowaoo** là một hệ thống cực kỳ phức tạp và chuyên nghiệp. Nó biến quy trình sáng tạo vốn phụ thuộc vào kỹ năng thủ công thành một quy trình có thể kiểm soát được (Controllable AI Production). Đây là mô hình tiêu biểu cho các ứng dụng **SaaS AI** thế hệ mới: tập trung vào quy trình (workflow) hơn là chỉ đơn thuần là một khung chat AI.