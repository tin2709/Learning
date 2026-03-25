Dưới đây là phân tích chi tiết về dự án **OmniSearch** — một công cụ tìm kiếm tệp tin hiệu năng cao dành cho Windows, kết hợp sức mạnh của C++, Rust và React.

---

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án sử dụng mô hình "kiềng ba chân" để tối ưu hóa từng phần của ứng dụng:

*   **C++ (Engine):** Là "trái tim" xử lý dữ liệu. OmniSearch sử dụng các API cấp thấp của Windows (`Win32 API`) để can thiệp trực tiếp vào cấu trúc hệ thống tệp **NTFS**. Cụ thể là truy cập **MFT (Master File Table)** và **USN (Update Sequence Number) Journal** thông qua lệnh `DeviceIoControl`. Đây là cách nhanh nhất để lấy danh mục tệp tin mà không cần quét từng thư mục theo cách thông thường.
*   **Rust (Bridge/Backend):** Đóng vai trò là lớp bảo mật và quản lý tài nguyên. Rust sử dụng **FFI (Foreign Function Interface)** để gọi mã C++, sau đó đóng gói thành các **Tauri Commands**. Rust cũng xử lý các tác vụ hệ thống như Tray Icon, phím tắt toàn cục (Global Shortcuts) và quản lý cửa sổ thông qua thư viện `windows-rs`.
*   **React 19 & TypeScript (Frontend):** Cung cấp giao diện người dùng hiện đại, phản ứng nhanh. Tận dụng sức mạnh của Vite để đóng gói và giao tiếp với Backend qua giao thức RPC của Tauri.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của OmniSearch được thiết kế theo kiểu phân tầng chức năng (Layered Architecture) cực kỳ rõ ràng:

*   **Tách biệt bối cảnh xử lý:** Chia làm 3 lớp:
    1.  *Lớp giao tiếp (UI Layer):* Chạy trong WebView2, xử lý trải nghiệm người dùng.
    2.  *Lớp logic hệ thống (System Layer):* Viết bằng Rust, điều phối các luồng dữ liệu và sự kiện hệ thống.
    3.  *Lớp phần cứng (Hardware Access Layer):* Viết bằng C++, làm việc trực tiếp với ổ đĩa cứng.
*   **Administrator Privileges (Quyền quản trị):** Tư duy kiến trúc ở đây là chấp nhận đánh đổi bảo mật để lấy hiệu năng. Việc yêu cầu quyền Admin qua `windows-app-manifest.xml` là bắt buộc để ứng dụng có quyền đọc thô (`raw volume access`) các phân vùng NTFS.
*   **Incremental Update (Cập nhật gia tăng):** Thay vì index lại từ đầu mỗi khi mở máy, kiến trúc sử dụng "USN watcher" để theo dõi các thay đổi tệp tin theo thời gian thực từ nhật ký của Windows.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **FFI Bridge (Rust <-> C++):** Trong `build.rs`, mã C++ được biên dịch bằng `cc` crate và liên kết trực tiếp vào Rust. Các hàm được khai báo `extern "C"` để đảm bảo tính tương thích bộ nhớ giữa hai ngôn ngữ.
*   **Native Drag-and-Drop:** Sử dụng các giao diện COM của Windows như `IDataObject` và `IDropSource` (trong `lib.rs`) để cho phép người dùng kéo tệp trực tiếp từ ứng dụng ra File Explorer.
*   **Memory Optimization:** Một kỹ thuật thú vị được tìm thấy trong `desktop.rs` là gọi `K32EmptyWorkingSet`. Khi cửa sổ ứng dụng bị ẩn, nó sẽ "ép" Windows giải phóng bộ nhớ RAM không cần thiết của tiến trình để giảm thiểu tài nguyên chiếm dụng.
*   **Multithreaded Hashing:** Khi tìm kiếm tệp trùng lặp (Duplicate Finder), ứng dụng sử dụng kỹ thuật băm (hashing) đa luồng để tận dụng tối đa CPU khi xử lý các tệp tin dung lượng lớn.
*   **DWM Styling:** Sử dụng `DwmSetWindowAttribute` để tùy chỉnh màu sắc thanh tiêu đề (Title Bar) theo theme của ứng dụng, tạo cảm giác "native" hoàn toàn trên Windows 11.

---

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Khởi động:**
    *   Ứng dụng khởi chạy và yêu cầu quyền Admin.
    *   Lớp Rust đăng ký các phím tắt (Alt+Shift+S) và tạo Tray Icon.
2.  **Lập chỉ mục (Indexing):**
    *   React gọi lệnh `start_indexing`.
    *   Rust chuyển yêu cầu xuống C++.
    *   C++ mở Volume (`\\.\C:`) -> Đọc MFT -> Phân tích cấu trúc thư mục -> Xây dựng Map chỉ mục trong bộ nhớ -> Trả về JSON cho Frontend.
3.  **Tìm kiếm (Searching):**
    *   Người dùng nhập từ khóa.
    *   Frontend gọi `search_files` (asynchronous).
    *   Backend Rust chạy tác vụ này trong `spawn_blocking` để không làm treo UI.
    *   C++ thực hiện tìm kiếm trên Map trong bộ nhớ và trả về kết quả gần như tức thì.
4.  **Xem trước (Preview):**
    *   Khi chọn một tệp, ứng dụng gọi `load_preview_data_url`.
    *   Rust đọc tệp tin, chuyển đổi sang định dạng Base64 và gửi lên Frontend để hiển thị ảnh/video/PDF mà không cần đường dẫn tệp thực tế trong HTML (tăng tính bảo mật).

### Tổng kết
**OmniSearch** là một ví dụ điển hình về việc sử dụng đúng ngôn ngữ cho đúng mục đích: **C++** để tối ưu tốc độ phần cứng, **Rust** để đảm bảo an toàn hệ thống và **React** để tạo ra giao diện người dùng tinh tế. Đây là một ứng dụng Desktop hiện đại, giải quyết triệt để vấn đề tốc độ tìm kiếm trên hệ điều hành Windows.