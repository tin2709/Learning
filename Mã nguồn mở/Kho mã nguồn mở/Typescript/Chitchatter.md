Dựa trên các tệp tin mã nguồn bạn đã cung cấp, dưới đây là bản phân tích chi tiết về dự án **Chitchatter** theo các khía cạnh: Công nghệ, Kiến trúc, Kỹ thuật và Luồng hoạt động.

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án này là một ứng dụng Web hiện đại tập trung vào quyền riêng tư, sử dụng các công nghệ "không máy chủ" (serverless) và giao thức ngang hàng (P2P):

*   **Frontend Framework:** **React** kết hợp với **Vite** để đóng gói nhanh. Ngôn ngữ chủ đạo là **TypeScript** nhằm đảm bảo an toàn về kiểu dữ liệu.
*   **P2P Networking:**
    *   **Trystero:** Thư viện chính để trừu tượng hóa WebRTC, cho phép tạo phòng chat mà không cần máy chủ API trung gian.
    *   **WebTorrent:** Sử dụng các server Tracker của torrent làm kênh truyền tín hiệu (signaling) để các trình duyệt tìm thấy nhau.
*   **Security (Bảo mật):**
    *   **Web Crypto API:** Sử dụng trực tiếp trên trình duyệt để tạo cặp khóa (Public/Private Key), mã hóa đầu cuối (E2EE).
    *   **secure-file-transfer:** Thư viện chuyên biệt để gửi file mã hóa qua WebRTC.
*   **Giao diện & UI:** **Material UI (MUI)**, hỗ trợ Dark/Light mode, Markdown (`react-markdown`) và Highlight code (`react-syntax-highlighter`).
*   **Persistence (Lưu trữ):** **localforage** (IndexedDB) chỉ dùng để lưu cài đặt người dùng (Settings), tuyệt đối không lưu nội dung tin nhắn.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Chitchatter được xây dựng theo mô hình **Web Mesh**:

*   **Decentralized (Phi tập trung):** Ứng dụng có thể chạy hoàn toàn từ các tài nguyên tĩnh (GitHub Pages). Máy chủ API (nếu có) chỉ phục vụ mục đích cấu hình relay (TURN server) để vượt tường lửa, không tham gia vào luồng dữ liệu chat.
*   **Ephemeral (Tính tạm thời):** Kiến trúc "Memory-only" cho tin nhắn. Khi đóng tab hoặc rời phòng, toàn bộ dữ liệu hội thoại biến mất. Đây là tư duy thiết kế để bảo vệ quyền riêng tư tuyệt đối.
*   **Service-Oriented (Hướng dịch vụ ở Client):** Các logic phức tạp được tách thành các `services` riêng biệt (Encryption, Serialization, FileTransfer) và được quản lý qua **React Context** (`SettingsContext`, `ShellContext`, `RoomContext`).
*   **SDK & Embeddability:** Dự án cung cấp một SDK dựa trên **Web Components** (`<chat-room />`), cho phép nhúng ứng dụng vào bất kỳ trang web nào khác thông qua iframe nhưng vẫn giữ được tính bảo mật.

---

### 3. Kỹ thuật Lập trình chính (Key Programming Techniques)

*   **Functional Programming & Hooks:** Tuân thủ nghiêm ngặt quy tắc sử dụng **Arrow Functions** (như quy định trong `AGENTS.md`). Sử dụng Custom Hooks cực kỳ mạnh mẽ để quản lý logic WebRTC (ví dụ: `usePeerAction`, `useRoom`, `useRoomVideo`).
*   **Cơ chế xác thực Peer (Peer Verification):**
    *   Trong `usePeerVerification.ts`, hệ thống sử dụng cơ chế Challenge-Response. Một token ngẫu nhiên được mã hóa bằng khóa công khai của Peer mới và yêu cầu Peer đó giải mã gửi lại để chứng minh danh tính.
*   **Quản lý luồng Media:** Tách biệt rõ ràng các loại luồng (Stream) như Webcam, Microphone và Screen Share thông qua các hooks chuyên biệt (`useRoomVideo`, `useRoomAudio`, `useRoomScreenShare`).
*   **Xử lý File lớn:** Sử dụng **StreamSaver.js** để ghi dữ liệu trực tiếp vào ổ đĩa từ luồng mạng, tránh việc làm tràn bộ nhớ RAM khi nhận file dung lượng lớn.
*   **Kiểm thử E2E:** Sử dụng **Playwright** với các kịch bản kiểm thử đa trình duyệt, mô phỏng việc nhiều người dùng tham gia vào cùng một phòng để chat.

---

### 4. Luồng hoạt động của hệ thống (System Workflow)

1.  **Khởi tạo (Init Phase):**
    *   Khi người dùng mở trang, `Init.tsx` sẽ kiểm tra môi trường trình duyệt.
    *   Tạo một cặp khóa RSA/AES mới (nếu chưa có) để dùng cho mã hóa.
    *   Tạo một ID người dùng (UUID) ngẫu nhiên.
2.  **Tham gia phòng (Joining Room):**
    *   Người dùng nhập tên phòng hoặc dùng UUID ngẫu nhiên.
    *   Ứng dụng kết nối tới các **WebTorrent Trackers** để tìm kiếm các Peer khác đang có cùng "mã băm" (hash) của tên phòng.
3.  **Thiết lập kết nối (Signaling & Handshake):**
    *   Thông qua Tracker, các Peer trao đổi các gói tin tín hiệu WebRTC (SDP/ICE Candidates).
    *   Trao đổi khóa công khai (Public Key) và Metadata (username).
    *   Thực hiện bước xác thực Peer tự động.
4.  **Trao đổi dữ liệu (Active Session):**
    *   **Tin nhắn văn bản:** Được mã hóa và gửi qua `DataChannel` của WebRTC.
    *   **Media:** Luồng video/audio được thêm trực tiếp vào kết nối P2P.
    *   **File:** Được chia nhỏ, mã hóa từng phần và gửi qua kênh dữ liệu.
5.  **Kết thúc (Termination):**
    *   Khi người dùng rời đi, sự kiện `onPeerLeave` kích hoạt để dọn dẹp bộ nhớ và đóng các luồng media. Dữ liệu trong RAM bị giải phóng.

### Tổng kết
Chitchatter là một ví dụ mẫu mực về việc tận dụng tối đa sức mạnh của trình duyệt hiện đại để loại bỏ hoàn toàn nhu cầu về máy chủ trung tâm. Mã nguồn được tổ chức rất chặt chẽ, coi trọng tính **bảo mật** và **hiệu suất P2P**.