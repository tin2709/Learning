Dựa trên các tệp tin và cấu trúc thư mục của dự án **rrweb** (phiên bản 2.0.0-alpha), dưới đây là phân tích chi tiết về công nghệ, kiến trúc và cơ chế hoạt động của hệ thống này:

### 1. Phân tích Công nghệ Cốt lõi (Tech Stack)

rrweb là một bộ công cụ cực kỳ phức tạp được xây dựng để "ghi lại và phát lại" (record and replay) mọi thứ diễn ra trên trình duyệt.

*   **Ngôn ngữ:** **TypeScript** chiếm ưu thế tuyệt đối (gần 90%), giúp đảm bảo cấu trúc dữ liệu sự kiện (event) luôn chặt chẽ giữa đầu ghi và đầu phát.
*   **Kiến trúc Monorepo:** Sử dụng **Yarn Workspaces** kết hợp với **Turborepo** để quản lý nhiều gói (packages) trong một kho mã nguồn duy nhất.
*   **Build Tool:** Chuyển đổi từ các công cụ cũ sang **Vite** và **Rollup** để đóng gói (bundling).
*   **DOM Manipulation:** Tận dụng tối đa các Web API hiện đại như **MutationObserver** (theo dõi thay đổi DOM), **requestAnimationFrame** (cho bộ định thời chính xác cao).
*   **Nén dữ liệu:** Sử dụng gói `@rrweb/packer` dựa trên thư viện **fflate** để nén các luồng sự kiện JSON thành chuỗi nhị phân ngắn gọn, giảm băng thông truyền tải.
*   **UI Playback:** **Svelte** được sử dụng để xây dựng `rrweb-player`, cung cấp giao diện điều khiển (play/pause/speed).

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của rrweb được chia thành các lớp chức năng riêng biệt nhằm tách rời quá trình thu thập dữ liệu và quá trình tái tạo:

1.  **Lớp Snapshot (`rrweb-snapshot`):** Đây là "trái tim" của hệ thống. Nó thực hiện hai nhiệm vụ:
    *   **Serialization:** Chuyển đổi cây DOM (một đối tượng phức tạp và có tính tham chiếu vòng) thành một cấu trúc dữ liệu JSON đơn giản, gán cho mỗi Node một `id` duy nhất.
    *   **Rebuilding:** Nhận dữ liệu JSON đó và xây dựng lại thành một cây DOM thật.
2.  **Lớp Quan sát (`@rrweb/record`):** Theo dõi các thay đổi gia tăng (incremental changes) sau bản snapshot đầu tiên. Nó không ghi lại toàn bộ trang web mỗi giây mà chỉ ghi lại các "vi biến động" (Mutation, Mouse Move, Scroll, Input).
3.  **Lớp Phát lại (`@rrweb/replay`):** Sử dụng một "máy ảo" (Virtual Machine/Timer) để áp dụng các thay đổi gia tăng vào cây DOM đã dựng lại theo đúng trình tự thời gian.
4.  **Lớp Tiện ích & Mở rộng:** Bao gồm các plugin ghi lại Console log, Canvas (qua WebRTC hoặc ảnh snapshot), và đóng gói dữ liệu.

### 3. Các Kỹ thuật Chính (Key Techniques)

rrweb giải quyết những bài toán kỹ thuật rất khó của trình duyệt bằng các phương pháp thông minh:

*   **Custom Serialization (Tuần tự hóa tùy chỉnh):** rrweb chuyển đổi các đường dẫn tương đối (CSS/Ảnh) thành đường dẫn tuyệt đối, chuyển đổi thẻ `script` thành `noscript` để ngăn thực thi JS ngoài ý muốn khi phát lại.
*   **High-Precision Timer:** `setTimeout` thông thường không đủ chính xác khi luồng chính bị nghẽn. rrweb sử dụng một bộ định thời tự hiệu chuẩn (calibrated timer) dựa trên `requestAnimationFrame` để đảm bảo các sự kiện như di chuyển chuột mượt mà như thực tế.
*   **Setter Hijacking:** Để ghi lại các thay đổi do mã JavaScript thực hiện (ví dụ: `input.value = "abc"`), rrweb "đè" (hijack) lên các hàm setter của nguyên mẫu (prototype) trình duyệt để bắt được sự kiện mà `MutationObserver` bỏ sót.
*   **Sandbox Playback:** Khi phát lại, rrweb đặt tài liệu trong một thẻ `iframe` với thuộc tính `sandbox` (loại bỏ thực thi JS, form submission, popup) để bảo mật và tránh xung đột với trang web hiện tại.
*   **Simulation Hover:** Vì không thể kích hoạt trạng thái `:hover` thật sự bằng JS, rrweb duyệt qua các stylesheet, sao chép các rule `:hover` và tạo ra các class CSS giả (ví dụ: `.\:hover`) để mô phỏng hiệu ứng thị giác.

### 4. Luồng Hoạt động của Hệ thống (System Workflow)

#### Giai đoạn Ghi (Recording Phase):
1.  **Full Snapshot:** Ngay khi bắt đầu, rrweb chụp lại toàn bộ trạng thái hiện tại của DOM.
2.  **Incremental Tracking:** 
    *   `MutationObserver` bắt đầu lắng nghe mọi thay đổi về cấu trúc DOM, thuộc tính và văn bản.
    *   Các sự kiện người dùng (chuột, cuộn trang, input) được thu thập và "throttle" (tiết lưu) để giảm dung lượng.
3.  **Event Stream:** Tất cả dữ liệu được đóng gói thành các đối tượng JSON kèm theo `timestamp`.
4.  **Emit:** Dữ liệu được đẩy ra qua hàm `emit` để người dùng lưu trữ vào DB hoặc gửi lên Server.

#### Giai đoạn Phát (Replaying Phase):
1.  **Initialization:** `Replayer` nhận danh sách sự kiện.
2.  **Synchronous Rebuild:** Snapshot đầu tiên được dựng lên ngay lập tức trong Sandbox Iframe.
3.  **Asynchronous Playback:** 
    *   Bộ định thời bắt đầu chạy.
    *   Dựa vào `timestamp`, hệ thống "đắp" các thay đổi gia tăng lên DOM.
    *   Chuột được mô phỏng bằng một phần tử đồ họa di chuyển trên màn hình.
4.  **Interaction:** Nếu người dùng tua (fast-forward), rrweb sẽ thực hiện tính toán gộp các thay đổi DOM một cách đồng bộ để nhanh chóng đạt đến trạng thái tại thời điểm đó.

### Tóm lại
**rrweb** không phải là một trình quay video màn hình, mà là một hệ thống **"Nhật ký biến động DOM"**. Nó cho phép tái tạo lại trải nghiệm người dùng với độ phân giải hoàn hảo (pixel-perfect) nhưng với dung lượng cực thấp so với video, đồng thời cho phép tìm kiếm và phân tích sâu vào dữ liệu cấu trúc bên trong sự kiện.