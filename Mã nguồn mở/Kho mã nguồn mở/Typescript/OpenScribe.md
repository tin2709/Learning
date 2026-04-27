Dựa trên mã nguồn bạn cung cấp, dưới đây là phân tích chi tiết về dự án **OpenScribe**:

### 1. Công nghệ cốt lõi (Core Technologies)

OpenScribe là một hệ thống hybrid phức tạp kết hợp giữa Web, Desktop và AI cục bộ:

*   **Frontend Framework:** **Next.js 16 (App Router)** và **React 19**. Đây là phiên bản rất mới, cho thấy dự án sử dụng các tính năng tiên tiến nhất của React như Server Actions.
*   **Desktop Wrapper:** **Electron**, cho phép chạy ứng dụng trên macOS, Windows và Linux với khả năng tương tác sâu vào phần cứng (micro, hệ thống tệp).
*   **AI Transcription (Chuyển đổi tiếng nói):**
    *   **Local:** `whisper.cpp` (thông qua `pywhispercpp`) để tối ưu hiệu suất trên CPU/GPU cục bộ.
    *   **Cloud:** OpenAI Whisper API (dùng làm fallback).
*   **AI Note Generation (Tạo văn bản lâm sàng):**
    *   **Local:** **Ollama** (chạy các model như Llama 3.2, Gemma 3) để đảm bảo quyền riêng tư tuyệt đối.
    *   **Cloud:** Anthropic **Claude 3.5/4.5 Sonnet** (mặc định cho bản web).
*   **Backend (Local Services):** **Python 3.11** với **FastAPI** và **Uvicorn** để tạo các server API nội bộ xử lý âm thanh và AI.
*   **Ngôn ngữ & Tooling:** **TypeScript** (chiếm đa số), **Tailwind CSS v4** (thiết kế monochrome tối giản), và quản lý monorepo bằng **pnpm workspaces**.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án đi theo triết lý **"Local-first & Privacy-conscious"**:

*   **Kiến trúc Monorepo:** Chia nhỏ hệ thống thành các `packages/` dùng chung và `apps/` riêng biệt.
    *   `apps/web`: Ứng dụng chính.
    *   `packages/pipeline`: Chia nhỏ quy trình xử lý thành các công đoạn (ingest -> transcribe -> assemble -> note-core).
    *   `packages/storage`: Tách biệt logic lưu trữ.
*   **Pipeline-as-a-Service:** Quy trình xử lý từ âm thanh đến ghi chú lâm sàng được thiết kế như một đường ống (pipeline). Mỗi giai đoạn có một hợp đồng (API/Interface) rõ ràng, cho phép thay thế nhà cung cấp (ví dụ: đổi từ Whisper sang MedASR) mà không ảnh hưởng đến UI.
*   **Hybrid Runtime:** Hỗ trợ 3 chế độ chạy: **Mixed Web** (Local Transcription + Cloud LLM), **Fully Local Desktop** (Cục bộ 100%), và **Cloud Fallback**.
*   **Security by Design:** Dữ liệu PHI (Thông tin sức khỏe cá nhân) được mã hóa bằng **AES-GCM** ngay tại trình duyệt trước khi lưu vào LocalStorage. Âm thanh chỉ xử lý trong bộ nhớ (in-memory), không lưu xuống đĩa nếu không cần thiết.

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)

*   **Real-time Audio Processing:** Sử dụng **Audio Worklets** (`pcm-processor.js`) để thu thập dữ liệu âm thanh PCM thô từ trình duyệt với độ trễ thấp nhất.
*   **Transcript Stitching (Ghép bản ghi):** Kỹ thuật xử lý chồng lấn (overlap trimming) trong `session-store.ts`. Khi âm thanh được gửi đi theo từng đoạn (segment), hệ thống sẽ so sánh các token cuối của đoạn cũ và đầu của đoạn mới để loại bỏ phần trùng lặp khi ghép thành văn bản hoàn chỉnh.
*   **Markdown-Driven Templates:** Thay vì dùng JSON Schema cứng nhắc, dự án sử dụng các file `.md` làm template cho ghi chú lâm sàng. Kỹ thuật này giúp các bác sĩ hoặc người đóng góp không cần biết lập trình vẫn có thể tùy chỉnh định dạng ghi chú (SOAP, HPI...).
*   **Provider Abstraction:** Sử dụng mẫu thiết kế **Strategy Pattern** trong `provider-resolver.ts` để quyết định xem sẽ dùng bộ công cụ AI nào dựa trên cấu hình `.env` của người dùng.
*   **Desktop-Backend Bridge:** Kỹ thuật đóng gói ứng dụng Python vào trong Electron bằng `PyInstaller`, sau đó giao tiếp qua IPC (Inter-Process Communication) hoặc các cổng localhost nội bộ.

### 4. Luồng hoạt động hệ thống (System Workflow)

Quy trình hoạt động từ lúc bác sĩ bắt đầu khám đến khi có ghi chú:

1.  **Giai đoạn Thu nhận (Audio Ingest):**
    *   Người dùng nhấn Record. `MediaRecorder` API bắt đầu thu âm.
    *   Âm thanh được chia nhỏ thành các đoạn (segments) dài khoảng 10-12 giây để xử lý song song.
2.  **Giai đoạn Chuyển ngữ (Transcription):**
    *   Các đoạn âm thanh được gửi đến API `/api/transcription/segment`.
    *   Hệ thống gọi Whisper (Local Server hoặc OpenAI).
    *   **Server-Sent Events (SSE)** được sử dụng để đẩy văn bản thô vừa dịch được về lại giao diện người dùng theo thời gian thực.
3.  **Giai đoạn Lắp ghép (Assembly):**
    *   `TranscriptionSessionStore` quản lý các đoạn văn bản, thực hiện ghép nối và xử lý lỗi nếu một đoạn âm thanh bị mất hoặc lỗi.
4.  **Giai đoạn Tạo ghi chú (Note Core):**
    *   Sau khi kết thúc buổi khám, toàn bộ bản ghi văn bản (transcript) được gửi đến LLM (Claude hoặc Ollama).
    *   LLM áp dụng Template (ví dụ: SOAP) để trích xuất thông tin bệnh nhân và kế hoạch điều trị.
5.  **Giai đoạn Lưu trữ & Hiển thị:**
    *   Văn bản cuối cùng được mã hóa và lưu vào LocalStorage.
    *   Người dùng có thể chỉnh sửa ghi chú trong `NoteEditor` hoặc gửi sang các hệ thống khác như OpenClaw.

### Tổng kết
OpenScribe là một ví dụ điển hình về việc xây dựng ứng dụng AI **quyền riêng tư cao** trong y tế. Sự kết hợp giữa khả năng xử lý mạnh mẽ của Cloud và tính an toàn của Local giúp nó trở thành một công cụ linh hoạt cho các chuyên gia y tế.