Dựa trên nội dung mã nguồn của dự án **prx**, dưới đây là phân tích chi tiết về các khía cạnh công nghệ, kiến trúc và vận hành của hệ thống:

### 1. Công nghệ cốt lõi (Core Tech Stack)

*   **Ngôn ngữ:** Go (Golang) phiên bản 1.26. Đây là một lựa chọn tối ưu cho công cụ CLI nhờ tốc độ thực thi nhanh, khả năng biên dịch thành tệp nhị phân duy nhất và quản lý đa luồng (concurrency) mạnh mẽ.
*   **Giao diện TUI (Terminal User Interface):** Sử dụng hệ sinh thái **Charm CLI**:
    *   `Bubble Tea`: Framework theo mô hình Elm để quản lý luồng dữ liệu và trạng thái UI.
    *   `Lip Gloss`: Thư viện định kiểu (styling) cho terminal.
    *   `Bubbles`: Các thành phần UI có sẵn (viewport, textarea, spinner).
    *   `Glamour`: Render Markdown ngay trong terminal.
*   **Tích hợp AI:** Sử dụng **Claude Code CLI** (`claude`) làm backend xử lý thông minh qua cơ chế subprocess. Thay vì gọi API trực tiếp, dự án điều phối lệnh `claude -p` để tận dụng quyền truy cập codebase và công cụ (tools) của Claude.
*   **Tương tác GitHub:** Phụ thuộc hoàn toàn vào **GitHub CLI** (`gh`). `prx` đóng vai trò là một lớp giao diện (wrapper) cấp cao, thực thi các lệnh `gh pr list`, `gh pr diff`, `gh pr merge` để thao tác với GitHub mà không cần quản lý token trực tiếp.
*   **Giao thức MCP (Model Context Protocol):** Triển khai server JSON-RPC để AI có thể "gọi ngược" lại các chức năng của hệ thống (ví dụ: yêu cầu AI tự thực hiện lệnh Approve PR).

### 2. Tư duy Kiến trúc (Architectural Thinking)

*   **Shared Context (App-Centric):** Cấu trúc `*app.App` chứa đựng toàn bộ thông tin về repo, cấu hình, cache và user hiện tại. Đối tượng này được truyền xuyên suốt để đảm bảo tính nhất quán dữ liệu.
*   **Scene-based UI:** Kiến trúc giao diện được chia thành các "Scene" (Conversation, DiffOverlay, BulkApprove). Mỗi Scene tự quản lý logic cập nhật (`Update`) và hiển thị (`View`) riêng, giúp mã nguồn TUI không bị phình to và dễ bảo trì.
*   **Local-First & Orchestration:** `prx` không cố gắng tái phát minh bánh xe. Nó được thiết kế như một **điều phối viên (Orchestrator)** giữa các công cụ CLI có sẵn. Tư duy này giúp giảm thiểu rủi ro bảo mật (không giữ token) và tận dụng được sức mạnh của các công cụ chính chủ từ GitHub và Anthropic.
*   **Incremental Review (Đánh giá gia tăng):** Hệ thống lưu trữ trạng thái đánh giá qua mã băm (SHA256) của từng đoạn code (hunk). Khi quay lại một PR đã xem, nó chỉ hiển thị những thay đổi mới, giúp giảm tải trí não cho người review.
*   **Cơ chế Caching:** Kết quả phân tích rủi ro của AI được cache dựa trên mã băm của nội dung PR và các tiêu chí đánh giá. Nếu PR không thay đổi, hệ thống sẽ trả kết quả ngay lập tức (< 2s).

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Parallel Fetching:** Sử dụng `sync.WaitGroup` để thực hiện đồng thời nhiều lời gọi CLI (lấy diff, kiểm tra CI, lấy comment, lấy review). Điều này giúp rút ngắn đáng kể thời gian tải PR chi tiết.
*   **Warm Process Pattern:** Trong `internal/ai/warm.go`, hệ thống khởi động trước một tiến trình Claude (`StartWarm`) ngay khi người dùng mở PR. Khi người dùng bắt đầu chat, AI đã sẵn sàng (init xong hooks/MCP), giúp phản hồi gần như tức thì.
*   **Hỗ trợ Graphics trong Terminal:** Sử dụng `go-termimg` để render ảnh thumbnail (Kitty/Sixel/iTerm2 protocol) ngay trong màn hình chat terminal, một tính năng hiếm thấy ở các công cụ CLI truyền thống.
*   **MCP Server Side-car:** Tích hợp một server MCP chạy qua socket Unix, cho phép Claude thực thi các hành động có tính thay đổi (mutation) như `merge_pr` hoặc `set_criterion` sau khi được người dùng xác nhận (Permission system).
*   **Fuzzy Hunk Matching:** Khi áp dụng các ghi chú của AI vào file diff thực tế, hệ thống sử dụng thuật toán so khớp mờ (sai số ±3 dòng) để đối phó với việc dòng code bị xê dịch do rebase hoặc thay đổi nhỏ.

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Khởi động (Bootstrap):** Kiểm tra sự tồn tại của `gh` và `claude`. Quét các repository mục tiêu và thông tin người dùng.
2.  **Quét PR (Triage):**
    *   Lấy danh sách PR đang mở và PR vừa mới merge (post-merge review).
    *   Tải dữ liệu chi tiết PR (Diff, Checks, Comments) theo cơ chế song song.
3.  **Phân tích rủi ro (Assessment):**
    *   Hệ thống gửi nội dung PR sang Claude kèm theo một "System Prompt" được thiết kế kỹ lưỡng.
    *   Claude chấm điểm rủi ro theo 5 tiêu chí (Blast radius, Intent clarity, v.v.).
    *   Kết quả trả về dưới dạng JSON và được hiển thị trong giao diện Chat.
4.  **Tương tác người dùng (Review):**
    *   Người dùng có thể chat với Claude để hỏi về các đoạn code phức tạp.
    *   Người dùng mở Diff Viewer để xem chi tiết code, nơi các đoạn code "rác" (boilerplate, trivial) tự động được thu gọn dựa trên phân tích của AI.
5.  **Thực thi hành động (Action):**
    *   Thông qua Chat hoặc phím tắt, người dùng ra lệnh Approve/Merge/Request Changes.
    *   `prx` gọi `gh CLI` để thực hiện lệnh trên GitHub.
    *   Cập nhật `ReviewState` cục bộ để đánh dấu các đoạn code đã được duyệt.

**Tổng kết:** `prx` là một công cụ "Code Review Terminal" hiện đại, kết hợp giữa triết lý Unix (nối các công cụ nhỏ thành hệ thống lớn) và sức mạnh của AI thế hệ mới để tối ưu hóa quy trình review mã nguồn vốn thường gây mệt mỏi cho lập trình viên.