Dựa trên mã nguồn của dự án **Gaseous Server**, đây là bản phân tích chi tiết về công nghệ, kiến trúc và kỹ thuật lập trình của hệ thống quản lý ROM và giả lập trò chơi này.

---

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Runtime & Framework:** Sử dụng **.NET 10.0** (phiên bản mới nhất/thử nghiệm), tận dụng các tính năng hiện đại của ngôn ngữ C# và ASP.NET Core MVC cho phần Web API.
*   **Cơ sở dữ liệu:** Hỗ trợ **MariaDB 11.1.2+** và **MySQL 8+**. Hệ thống giao tiếp qua thư viện `MySqlConnector` thay vì sử dụng các ORM nặng nề như Entity Framework (hướng tới hiệu năng xử lý dữ liệu thô).
*   **Xác thực & Bảo mật:** Sử dụng **Microsoft ASP.NET Core Identity** để quản lý người dùng, phân quyền (Player, Gamer, Admin) và hỗ trợ **2FA (Xác thực 2 yếu tố)**, bao gồm cả mã khôi phục (Recovery Codes).
*   **Xử lý hình ảnh & Tệp tin:**
    *   **Magick.NET:** Để xử lý, resize và chuyển đổi các định dạng ảnh bìa (Covers), ảnh chụp màn hình (Screenshots).
    *   **SharpCompress & SevenZipSharp:** Hỗ trợ giải nén và quản lý các định dạng lưu trữ ROM phổ biến (Zip, Rar, 7z).
*   **Giả lập & Frontend:** Tích hợp **EmulatorJS** (Javascript dựa trên RetroArch) để chạy trò chơi ngay trên trình duyệt mà không cần cài đặt thêm phần mềm.
*   **Metadata (Dữ liệu đặc tả):** Tích hợp sâu với **IGDB API** và dịch vụ proxy **Hasheous** để nhận diện trò chơi qua mã băm (hash) của tệp ROM.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án được thiết kế theo mô hình **Multiprocess Architecture** (Kiến trúc đa tiến trình) nhằm tách biệt các mối quan tâm:

*   **gaseous-server (Main Host):** Đóng vai trò là Web Server xử lý các yêu cầu HTTP, cung cấp giao diện người dùng và API.
*   **gaseous-processhost (Background Worker):** Đây là một điểm sáng trong kiến trúc. Các tác vụ nặng như quét thư viện (Library Scan), nén/giải nén tệp, hoặc tải metadata được đẩy ra một tiến trình riêng biệt để tránh làm treo hoặc quá tải Web Server chính.
*   **gaseous-lib (Core Library):** Chứa toàn bộ logic nghiệp vụ, models và các plugin. Đây là "trái tim" dùng chung cho cả Server, CLI và ProcessHost.
*   **Plugin-based Architecture:** Hệ thống sử dụng các **Interface** (như `IMetadataProvider`, `IDecompressPlugin`, `ITaskPlugin`) để cho phép mở rộng tính năng mà không cần sửa đổi mã nguồn cốt lõi. Ví dụ: Có thể dễ dàng thêm một nguồn lấy dữ liệu game mới bằng cách hiện thực `IMetadataProvider`.
*   **Database-first Migration:** Hệ thống quản lý phiên bản DB thông qua các tệp SQL thuần (`Support/Database/MySQL/gaseous-xxxx.sql`). Khi khởi động, server sẽ tự động kiểm tra `schema_version` và chạy các bản cập nhật để đảm bảo tính nhất quán dữ liệu.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Custom Database Wrapper:** Thay vì dùng ORM, dự án xây dựng lớp `Database.cs` để thực thi SQL trực tiếp với các tham số (`Dictionary<string, object>`). Điều này giúp kiểm soát tối đa các truy vấn phức tạp và tối ưu hóa tốc độ.
*   **Memory Caching:** Lớp `MemoryCache.cs` được tích hợp vào tầng dữ liệu để lưu trữ kết quả của các truy vấn metadata lặp đi lặp lại, giảm tải cho ổ đĩa và DB.
*   **Asynchronous Programming:** Sử dụng triệt để `async/await` trong tất cả các tác vụ I/O (giao tiếp DB, gọi API IGDB, đọc ghi file ROM dung lượng lớn).
*   **Reflection & Metadata Mapping:** Kỹ thuật Mapping tự động các thuộc tính giữa các đối tượng khác nhau (ví dụ: chuyển dữ liệu từ IGDB model sang Gaseous model) dựa trên tên thuộc tính.
*   **Localisation Management:** Hệ thống đa ngôn ngữ được xử lý qua tệp JSON. Lớp `Localisation.cs` chịu trách nhiệm nạp động các bản dịch dựa trên Locale của trình duyệt hoặc cấu hình server.
*   **Windows Service Integration:** Hỗ trợ chạy như một Windows Service thông qua thư viện `Microsoft.Extensions.Hosting.WindowsServices`, đi kèm với một công cụ GUI (Configurator) để quản lý trạng thái service.

---

### 4. Luồng hoạt động hệ thống (System Flow)

#### A. Luồng Nhập ROM (Import Flow):
1.  Người dùng chỉ định thư mục chứa ROM.
2.  **LibraryScan** (Background Task) quét các tệp tin.
3.  Tính toán mã băm (CRC32, MD5, SHA1) cho từng tệp.
4.  Tra cứu mã băm trong DB cục bộ hoặc qua Hasheous Proxy để xác định ID trò chơi trên IGDB.
5.  Tải metadata, ảnh bìa, năm phát hành...
6.  Di chuyển/Sắp xếp tệp vào thư mục thư viện chuẩn theo cấu trúc: `Platform/Game/Rom`.

#### B. Luồng Giả lập (Emulation Flow):
1.  Người dùng nhấn "Play" trên Web UI.
2.  Server kiểm tra quyền truy cập của người dùng.
3.  Server cung cấp tệp ROM qua một luồng Stream an toàn (có thể yêu cầu auth).
4.  Frontend khởi tạo EmulatorJS, tải tệp Core (giả lập) tương ứng với Platform của game.
5.  Trò chơi khởi chạy trong vùng Sandbox của trình duyệt.

#### C. Luồng Duy trì (Maintenance Flow):
*   **Daily/Weekly Maintainer:** Các tác vụ tự động dọn dẹp tệp tạm, kiểm tra tính toàn vẹn của tệp ROM trên đĩa và cập nhật lại các metadata bị thiếu hoặc thay đổi.

### Tổng kết
Gaseous Server là một hệ thống **.NET chuyên sâu**, ưu tiên tính module hóa và sự tách biệt giữa xử lý giao diện và xử lý dữ liệu nặng. Việc sử dụng kiến trúc đa tiến trình và hệ thống plugin làm cho nó trở nên rất mạnh mẽ và dễ mở rộng cho những người dùng có thư viện game khổng lồ.