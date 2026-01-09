Dựa trên mã nguồn và cấu trúc thư mục của dự án **Streamdal**, dưới đây là bản phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động dưới dạng file `README.md` bằng tiếng Việt.

---

# Phân Tích Kỹ Thuật Dự Án Streamdal

Streamdal là một giải pháp **Code-Native Data Privacy** (Bảo mật dữ liệu gốc từ mã nguồn), cho phép chạy các tác vụ xử lý dữ liệu trực tiếp bên trong ứng dụng thông qua WebAssembly.

## 1. Công Nghệ Cốt Lõi (Core Technology Stack)

Dự án được xây dựng dựa trên các công nghệ hiện đại nhằm tối ưu hiệu suất và khả năng mở rộng:

*   **WebAssembly (Wasm):** Đây là "trái tim" của Streamdal. Các pipeline xử lý dữ liệu (masking, validation, transformation) được biên dịch thành Wasm. Điều này cho phép thực thi logic xử lý với tốc độ gần như bản địa (near-native speed) trên nhiều ngôn ngữ lập trình khác nhau (Go, Python, Node.js) mà không cần thay đổi hạ tầng.
*   **Protocol Buffers (Protobuf) & gRPC:** Sử dụng để định nghĩa cấu trúc dữ liệu và giao tiếp giữa các thành phần. Protobuf đảm bảo tính nhất quán của dữ liệu (Data Contracts) giữa Server, Console và các SDK.
*   **Ngôn ngữ lập trình:**
    *   **Go:** Dùng cho thành phần Server và CLI (tối ưu hóa xử lý song song và hiệu suất hệ thống).
    *   **Rust:** Dùng để viết các thư viện xử lý Wasm (wasm-detective, wasm-transformer) vì tính an toàn bộ nhớ và hiệu suất cực cao.
    *   **TypeScript/Deno (Fresh Framework):** Dùng cho thành phần Console (UI) để quản lý luồng dữ liệu thời gian thực.
*   **Redis:** Sử dụng làm lớp lưu trữ trạng thái (state storage) cho Server.

## 2. Kỹ Thuật và Tư Duy Kiến Trúc (Architecture Design)

Streamdal sử dụng kiến trúc **Control Plane - Data Plane** tách biệt, nhưng với một cách tiếp cận khác biệt:

*   **Kiến trúc "Sidecar-less" (Không cần Sidecar):** Thay vì đẩy dữ liệu ra một proxy hoặc dịch vụ bên ngoài để xử lý (gây độ trễ), Streamdal tích hợp trực tiếp vào mã nguồn ứng dụng thông qua SDK.
*   **Client-Side Execution (Thực thi tại máy khách):** Pipeline dữ liệu chạy ngay trên CPU của chính ứng dụng. Server chỉ đóng vai trò quản lý (Control Plane), gửi các chỉ thị và bytecode Wasm xuống SDK.
*   **Tư duy Code-Native:** Dữ liệu được xử lý ngay khi ứng dụng đọc (`read`) hoặc ghi (`write`). Điều này giúp loại bỏ nhu cầu xây dựng các hệ thống ETL phức tạp bên ngoài.
*   **Monorepo:** Dự án quản lý tất cả SDK, Server, CLI và thư viện Wasm trong một repo duy nhất, giúp đồng bộ hóa phiên bản Protobuf và logic xử lý dễ dàng hơn.

## 3. Các Kỹ Thuật Chính Nổi Bật

*   **PII Detection & Masking:** Sử dụng các thư viện Rust trong Wasm để nhận diện dữ liệu nhạy cảm (email, số điện thoại, thẻ tín dụng) bằng Regex hoặc Pattern Matching và ẩn chúng đi trước khi dữ liệu rời khỏi bộ nhớ ứng dụng.
*   **Dynamic Pipeline Updates:** Người dùng có thể thay đổi luật xử lý dữ liệu trên Dashboard (Console), Server sẽ đẩy cấu hình mới xuống SDK ngay lập tức mà không cần khởi động lại ứng dụng.
*   **Schema Inference:** Tự động suy luận cấu trúc dữ liệu (JSON) để người dùng có thể áp dụng các quy tắc kiểm chứng (Validation) mà không cần cấu hình thủ công từng trường.
*   **Tail -f for Data:** Kỹ thuật stream ngược dữ liệu đang xử lý trong ứng dụng lên Console để debug thời gian thực (giống như lệnh `tail -f` trong log).

## 4. Tóm Tắt Luồng Hoạt Động (Workflow Summary)

Luồng đi của dữ liệu và chỉ thị trong dự án diễn ra như sau:

1.  **Khởi tạo (Registration):** Ứng dụng tích hợp SDK khởi chạy và kết nối tới **Streamdal Server** qua gRPC. Nó khai báo "Audience" (tên dịch vụ, thành phần, và loại thao tác - Producer/Consumer).
2.  **Nhận cấu hình (Discovery):** Server kiểm tra xem có Pipeline nào được gán cho Audience này không. Nếu có, Server gửi cấu hình và file `.wasm` tương ứng xuống SDK.
3.  **Xử lý tại chỗ (Local Processing):**
    *   Khi ứng dụng gọi hàm `.Process()`, SDK nạp dữ liệu vào runtime WebAssembly.
    *   Các hàm Wasm thực hiện kiểm tra hoặc biến đổi dữ liệu (ví dụ: cắt tỉa dữ liệu, ẩn thông tin cá nhân).
    *   Kết quả trả về cho ứng dụng để tiếp tục xử lý logic nghiệp vụ.
4.  **Giám sát (Monitoring):** SDK gửi các thông tin đo lường (metrics) như số lượng bản tin thành công/thất bại và các mẫu dữ liệu (nếu được yêu cầu) ngược về Server để hiển thị trên **Console**.
5.  **Điều khiển (Command & Control):** Người dùng thông qua **Console** hoặc **CLI** thực hiện thay đổi Pipeline. Lệnh này được Server gửi xuống SDK qua một luồng gRPC duy trì liên tục (long-lived stream) để cập nhật logic xử lý ngay lập tức.

---
*Phân tích này dựa trên phiên bản mã nguồn hiện tại của dự án Streamdal.*