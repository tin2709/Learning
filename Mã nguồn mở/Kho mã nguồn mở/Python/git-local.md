Dựa trên mã nguồn và tài liệu của dự án **git-local**, dưới đây là phân tích chi tiết về các khía cạnh kỹ thuật và kiến trúc của công cụ này:

### 1. Công nghệ cốt lõi (Core Tech Stack)
*   **Ngôn ngữ:** Python 3.6+. Đây là lựa chọn tối ưu cho các công cụ CLI hệ thống vì tính phổ biến, dễ đọc và thư viện tiêu chuẩn mạnh mẽ.
*   **Git Integration:** Hệ thống phụ thuộc hoàn toàn vào Git CLI. Thay vì sử dụng các thư viện như `GitPython`, tác giả chọn gọi trực tiếp lệnh Git thông qua module `subprocess`. Điều này đảm bảo tính tương thích tối đa với phiên bản Git mà người dùng đang cài đặt.
*   **Patching/Diffing Engine:** Sử dụng định dạng **Unified Diff**. Đây là công nghệ nền tảng của Git, cho phép mô tả các thay đổi mã nguồn dưới dạng các "hunks" (đoạn mã thay đổi). Công cụ tận dụng lệnh `git apply` và `git apply --reverse` để thực hiện việc áp dụng hoặc gỡ bỏ các thay đổi cục bộ.

### 2. Tư duy Kiến trúc (Architectural Thinking)
*   **Vị trí lưu trữ dữ liệu thông minh:** Dự án lưu trữ các bản vá (patches) trong thư mục `.git/local-patches/`. 
    *   *Ưu điểm:* Thư mục `.git/` nằm ngoài cây thư mục làm việc được Git theo dõi, nên các bản vá này sẽ **không bao giờ** bị commit hoặc push lên server, đồng thời không cần phải thêm vào `.gitignore`.
*   **Độ mịn cấp độ "Hunk" (Hunk-level granularity):** Khác với các giải pháp truyền thống như `git update-index --skip-worktree` (chỉ áp dụng cho toàn bộ file), `git-local` cho phép người dùng chọn lọc từng đoạn mã nhỏ trong một file để giữ lại cục bộ.
*   **Cơ chế "Vòng đời tự động" (Automation Lifecycle):** Kiến trúc dựa trên việc can thiệp vào quy trình commit của Git thông qua Hooks:
    *   **Pre-commit:** Tạm thời gỡ bỏ (strip) các thay đổi cục bộ để đảm bảo chúng không lọt vào commit.
    *   **Post-commit:** Ngay lập tức áp dụng lại (re-apply) các thay đổi đó sau khi commit thành công.
*   **Tính bền vững (Persistence):** Vì patches được lưu trong `.git/`, chúng tồn tại xuyên suốt qua các lần chuyển nhánh (checkout) hoặc rebase, giúp duy trì môi trường phát triển cá nhân ổn định.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)
*   **Xử lý luồng lệnh (Subprocess Orchestration):** Sử dụng `subprocess.run` với các tham số `capture_output=True` và `text=True` để đọc kết quả từ lệnh Git và xử lý logic dựa trên mã lỗi (returncode).
*   **Phân tách Diff (Diff Parsing):** Tác giả viết logic thủ công để parse (phân tích) kết quả từ `git diff`. Kỹ thuật này chia nhỏ file diff dựa trên các dòng bắt đầu bằng `@@`, biến chúng thành các đối tượng hunk riêng biệt để người dùng tương tác.
*   **Bảo vệ đường dẫn (Path Normalization):** Kỹ thuật thay thế dấu gạch chéo `/` bằng dấu gạch dưới kép `__` trong tên file bản vá (`patch_file`) để có thể lưu trữ cấu trúc thư mục phức tạp trong một thư mục phẳng mà không vi phạm quy tắc đặt tên file của hệ điều hành.
*   **Tiêm mã vào Hooks (Hook Injection):** Mã nguồn có khả năng nhận diện thư mục `.husky` (nếu có) hoặc thư mục hooks mặc định của Git để cài đặt tự động. Kỹ thuật này bao gồm việc kiểm tra xem đoạn mã đã tồn tại chưa để tránh trùng lặp.

### 4. Luồng hoạt động hệ thống (System Workflow)
Hệ thống vận hành theo một chu trình khép kín:

1.  **Giai đoạn Lưu (Save):** 
    *   Công cụ lấy diff của file.
    *   Hiển thị giao diện tương tác để người dùng chọn Hunk (y/n/a/q).
    *   Lưu các hunk đã chọn vào `.git/local-patches/`.
2.  **Giai đoạn Commit (Pre-commit Hook):**
    *   Chạy lệnh `strip`: Sử dụng `git apply --reverse --index` để xóa thay đổi khỏi cả thư mục làm việc và index (staging area).
    *   **Kiểm tra an toàn:** Nếu sau khi xóa, không còn gì để commit (chỉ có thay đổi cục bộ), hệ thống sẽ apply lại và hủy bỏ lệnh commit để tránh tạo commit trống hoặc mất trạng thái.
3.  **Giai đoạn Hoàn tất (Post-commit Hook):**
    *   Chạy lệnh `apply`: Sử dụng `git apply` để đưa các thay đổi từ file bản vá trở lại mã nguồn.
4.  **Giai đoạn Kiểm tra (Status/List):**
    *   Thực hiện "dry-run" bằng `git apply --check` để xác định trạng thái thực tế (đã áp dụng, đã gỡ bỏ hay bị xung đột do code gốc thay đổi quá nhiều).

**Tổng kết:** `git-local` là một giải pháp tinh tế, giải quyết triệt để vấn đề "config cục bộ" bằng cách tận dụng chính các công cụ lõi của Git và kết hợp với cơ chế Hook để tạo ra trải nghiệm người dùng liền mạch.