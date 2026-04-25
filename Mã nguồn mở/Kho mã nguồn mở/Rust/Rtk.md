Dựa trên tệp tin bạn cung cấp, đây là phân tích chi tiết về dự án **RTK (Rust Token Killer)** - một công cụ Proxy CLI hiệu suất cao được thiết kế để giảm thiểu tiêu thụ token cho các mô hình ngôn ngữ lớn (LLM):

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án tận dụng hệ sinh thái Rust để đạt được tốc độ và độ an toàn tối đa:
*   **Ngôn ngữ chính:** **Rust (chiếm 92.1%)**, được chọn vì khả năng khởi động cực nhanh (<10ms) và quản lý bộ nhớ hiệu quả (<5MB RAM).
*   **CLI Framework:** `clap` (v4) để xử lý các câu lệnh và tham số dòng lệnh một cách mạnh mẽ.
*   **Xử lý dữ liệu & Regex:** `regex` để lọc văn bản, `serde`/`serde_json` để xử lý cấu trúc dữ liệu JSON từ các công cụ như `gh` hoặc `npm`.
*   **Cơ sở dữ liệu:** `rusqlite` (SQLite) được dùng để lưu trữ lịch sử tiêu thụ token và tính toán mức độ tiết kiệm (`rtk gain`).
*   **Hệ thống Hook:** Sử dụng **Bash shell scripts** và **TypeScript** để tích hợp sâu vào các AI Agent (Claude Code, Copilot, Cursor).
*   **Kiểm thử (Testing):** `insta` để thực hiện snapshot testing (kiểm tra định dạng đầu ra) và các script smoke test (`test-all.sh`) để đảm bảo không có lỗi hồi quy.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của RTK được xây dựng dựa trên nguyên tắc **"Zero-Overhead Proxy"**:
*   **Mô hình Proxy:** RTK đứng giữa AI Agent và hệ điều hành. Thay vì AI gọi trực tiếp `git status`, nó sẽ gọi `rtk git status`. RTK thực thi lệnh thật, thu giữ kết quả, nén lại rồi mới trả về cho AI.
*   **Ưu tiên hiệu suất (Performance First):** Thiết kế đơn luồng (single-threaded), không sử dụng runtime async (như Tokio) để tránh độ trễ khi khởi động. Mục tiêu luôn là giữ overhead dưới 10ms để lập trình viên không cảm nhận được sự chậm trễ.
*   **Phân cấp bộ lọc (Layered Filtering):**
    *   **Rust Modules:** Dành cho các lệnh phức tạp cần phân tích cú pháp sâu (như JSON, NDJSON).
    *   **TOML DSL:** Một ngôn ngữ khai báo đơn giản cho phép người dùng định nghĩa bộ lọc bằng Regex mà không cần biên dịch lại mã nguồn.
*   **Nguyên tắc "Never Block":** Nếu bộ lọc gặp lỗi, RTK sẽ tự động trả về kết quả thô (raw output). Công cụ này đảm bảo không bao giờ làm gián đoạn luồng công việc của người dùng.

### 3. Các kỹ thuật chính (Key Techniques)

RTK sử dụng 4 chiến lược nén nội dung chính:
*   **Smart Filtering:** Loại bỏ nhiễu như comments, khoảng trắng thừa, boilerplate code (sử dụng `src/core/filter.rs`).
*   **Grouping:** Nhóm các mục tương tự lại với nhau (ví dụ: gộp các file lỗi theo thư mục thay vì liệt kê từng file một cách rời rạc).
*   **Truncation (Cắt bỏ thông minh):** Giữ lại ngữ cảnh quan trọng và cắt bớt các phần lặp lại hoặc dư thừa (ví dụ: chỉ hiện 5 commit gần nhất thay vì 50).
*   **Deduplication:** Gộp các dòng log giống hệt nhau và hiển thị số lần xuất hiện (ví dụ: `Error x42`).
*   **Kỹ thuật Hooking nâng cao:** Sử dụng `lexer.rs` để phân tích các câu lệnh shell phức tạp (có pipe `|`, redirect `>`, hoặc `&&`) để ghi đè (rewrite) chính xác từng phần của câu lệnh mà không làm hỏng cú pháp.
*   **Tee Recovery:** Khi một lệnh thất bại, RTK lưu toàn bộ output thô vào một file log cục bộ và cung cấp đường dẫn cho AI, giúp AI có thể đọc chi tiết lỗi nếu cần mà không tốn token trong context chính.

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

Luồng xử lý của một câu lệnh thông qua RTK diễn ra như sau:

1.  **Cài đặt:** Người dùng chạy `rtk init`. RTK cài đặt các script hook vào thư mục cấu hình của AI Agent (như `~/.claude/hooks`).
2.  **Đánh chặn (Interception):** Khi AI Agent chuẩn bị chạy một lệnh (ví dụ: `cargo test`):
    *   Hook sẽ gửi lệnh này đến `rtk rewrite`.
    *   RTK kiểm tra xem lệnh này có bộ lọc hỗ trợ không. Nếu có, nó đổi lệnh thành `rtk cargo test`.
3.  **Thực thi Subprocess:** RTK thực hiện lệnh gốc trong một tiến trình con, thu giữ toàn bộ `stdout` và `stderr`.
4.  **Xử lý bộ lọc (Pipeline):**
    *   Loại bỏ mã ANSI (màu sắc).
    *   Áp dụng các quy tắc nén (Rust logic hoặc TOML Regex).
    *   Cắt ngắn nếu vượt quá giới hạn token.
5.  **Ghi nhận & Trả về:**
    *   RTK tính toán số token tiết kiệm được và lưu vào `history.db`.
    *   Kết quả đã nén được in ra màn hình cho AI Agent đọc.
    *   Mã thoát (Exit code) của lệnh gốc được bảo toàn để AI Agent biết lệnh thành công hay thất bại.
6.  **Phân tích:** Người dùng có thể chạy `rtk gain` bất cứ lúc nào để xem báo cáo tổng quát về số tiền/token đã tiết kiệm được.

**Kết luận:** RTK không chỉ là một trình lọc văn bản đơn thuần, mà là một hệ thống hạ tầng tối ưu hóa chi phí AI, kết hợp giữa kỹ thuật hệ thống (Rust), xử lý ngôn ngữ (Regex/Parsers) và tích hợp quy trình làm việc (AI Agent Hooks).