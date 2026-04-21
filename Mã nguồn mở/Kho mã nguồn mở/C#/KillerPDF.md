Chào bạn, dựa trên mã nguồn và tài liệu của dự án **KillerPDF**, đây là bản phân tích chi tiết về công nghệ, kiến trúc và kỹ thuật lập trình của ứng dụng này:

---

### 1. Công nghệ cốt lõi (Core Technology)

KillerPDF được xây dựng với mục tiêu: **Nhẹ, không cần cài đặt (portable) và không phụ thuộc vào Adobe.**

*   **UI Framework:** **WPF (Windows Presentation Foundation)** trên **.NET Framework 4.8**. Việc chọn .NET 4.8 là một quyết định chiến lược (thay vì .NET 8) vì nó có sẵn trên hầu hết các máy Windows 10/11, giúp người dùng chạy file `.exe` ngay lập tức mà không cần cài thêm Runtime.
*   **Thư viện PDF (The "Triad"):**
    *   **Docnet.Core (PDFium):** Sử dụng engine của Google (giống Chrome) để render PDF chất lượng cao ra bitmap.
    *   **PdfPig:** Dùng để đọc cấu trúc văn bản, hỗ trợ tìm kiếm toàn văn và trích xuất text.
    *   **PdfSharpCore:** Dùng cho các tác vụ thay đổi cấu trúc file như gộp (merge) và chia (split) trang.
*   **Đóng gói (Packaging):** **Costura.Fody**. Đây là kỹ thuật cực kỳ quan trọng giúp nhúng tất cả file DLL (bao gồm cả file native `pdfium.dll`) vào trong duy nhất một file `.exe` dung lượng ~6MB.
*   **PolySharp:** Cho phép sử dụng các tính năng mới nhất của ngôn ngữ C# (như C# 12) trên nền tảng .NET 4.8 cũ.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án đi theo triết lý **"Zero-Footprint & Local-First"**:

*   **Hybrid Rendering Pattern:** Ứng dụng không cố gắng vẽ trực tiếp các đối tượng PDF lên màn hình bằng vector. Thay vào đó, nó render trang PDF thành một hình ảnh (Bitmap) và chồng một lớp **WPF Canvas** lên trên. Mọi thao tác vẽ, highlight, ký tên thực chất là vẽ trên lớp Canvas này, giúp tăng hiệu năng và đơn giản hóa việc quản lý UI.
*   **Tool-based State Machine:** Quản lý tương tác người dùng qua `EditTool` (Select, Text, Draw...). Mỗi mode sẽ thay đổi cách ứng dụng phản ứng với các sự kiện chuột (Mouse Events).
*   **Custom Chrome Architecture:** Ứng dụng sử dụng `WindowStyle="None"` và tự định nghĩa lại thanh tiêu đề (Title Bar), nút đóng/thu nhỏ để tạo giao diện Dark Theme hiện đại, chuyên nghiệp, thoát ly khỏi giao diện mặc định của Windows.
*   **Non-destructive Editing:** Các thay đổi (annotation) được lưu dưới dạng các đối tượng riêng biệt (`PageAnnotation`) trong bộ nhớ. Chúng chỉ thực sự được "làm phẳng" (flatten) vào file PDF khi người dùng thực hiện lệnh In hoặc Lưu.

### 3. Kỹ thuật Lập trình Chính (Key Programming Techniques)

*   **XAML Templating & Styling:** Sử dụng `ControlTemplate` để tùy biến hoàn toàn ScrollBar (siêu mỏng màu xanh), ContextMenu và Button. Điều này tạo nên đặc trưng thẩm mỹ của thương hiệu "Killer Tools".
*   **Serialization (Tuần tự hóa):** Chuyển đổi các tọa độ vẽ từ WPF (vốn khó lưu trữ trực tiếp) sang class `SerializablePoint` để lưu chữ ký người dùng vào AppData dưới dạng JSON.
*   **Win32 Hooking:** Sử dụng hook `WM_GETMINMAXINFO` (qua `HwndSource`) để xử lý lỗi khi phóng to cửa sổ frameless không bị che mất thanh Taskbar của Windows — một kỹ thuật xử lý UI Windows chuyên sâu.
*   **Drag & Drop Logic:** Xử lý kéo thả trong `ListBox` để sắp xếp lại thứ tự các trang PDF một cách trực quan.
*   **Font Matching:** Khi chỉnh sửa văn bản, mã nguồn cố gắng khớp font chữ đang dùng trong PDF với font hệ thống để đảm bảo tính nhất quán của tài liệu sau khi sửa.

### 4. Luồng hoạt động của hệ thống (System Workflow)

1.  **Giai đoạn Load:**
    *   Người dùng chọn file -> `Docnet.Core` mở file.
    *   Hệ thống render trang đầu tiên thành `BitmapSource` và hiển thị vào `Image control`.
    *   `PdfPig` quét tọa độ của tất cả các ký tự trên trang đó để chuẩn bị cho việc bôi đen (select) hoặc tìm kiếm.
2.  **Giai đoạn Tương tác:**
    *   Người dùng chọn tool "Draw" -> Click chuột lên `AnnotationCanvas`.
    *   Hệ thống bắt tọa độ chuột, tạo đối tượng `InkAnnotation` và vẽ các đoạn thẳng nối tiếp nhau (Polyline) trên Canvas.
3.  **Giai đoạn Xử lý trang:**
    *   Khi Merge/Split: `PdfSharpCore` can thiệp vào cấu trúc file ở cấp độ thấp (Low-level PDF Objects) để sao chép các trang từ file này sang file khác mà không làm mất định dạng ban đầu.
4.  **Giai đoạn Output:**
    *   Khi In/Save: Ứng dụng tạo một "Print Document". Nó vẽ hình ảnh trang gốc trước, sau đó vẽ các lớp Annotation từ Canvas đè lên trên theo đúng tỉ lệ (Scaling), sau đó xuất ra máy in hoặc file PDF mới.

### Tổng kết
**KillerPDF** là một ví dụ điển hình về việc kết hợp các thư viện mã nguồn mở một cách thông minh để tạo ra một công cụ chuyên biệt (Niche Tool). Nó không cố gắng thay thế hoàn toàn Adobe Acrobat về mọi tính năng, nhưng nó giải quyết cực tốt các bài toán: **Nhanh, Gọn, Không cần cài đặt và Đủ dùng cho công việc hiện trường.**