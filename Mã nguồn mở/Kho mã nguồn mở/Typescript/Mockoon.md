Dựa trên cấu trúc mã nguồn và thông tin từ kho lưu trữ của **Mockoon**, dưới đây là bản phân tích chi tiết về dự án:

### 1. Công nghệ cốt lõi (Core Technology Stack)

*   **Ngôn ngữ lập trình chính:** **TypeScript (chiếm ~88%)**. Dự án sử dụng TypeScript xuyên suốt từ core logic đến giao diện để đảm bảo tính an toàn về kiểu dữ liệu.
*   **Giao diện Desktop (App):**
    *   **Electron:** Framework để xây dựng ứng dụng desktop đa nền tảng (Windows, macOS, Linux).
    *   **Angular:** Framework phía client được sử dụng trong tiến trình renderer của Electron.
*   **Backend & CLI:**
    *   **Node.js:** Môi trường thực thi cho các tác vụ server-side và CLI.
    *   **Express.js:** Thư viện đứng sau các mock server mà người dùng khởi tạo, giúp xử lý các route HTTP một cách linh hoạt.
*   **Công cụ build:**
    *   **Webpack:** Đóng gói mã nguồn cho Electron Main process.
    *   **Angular CLI:** Build phần giao diện người dùng.
    *   **electron-builder:** Đóng gói ứng dụng thành các bản cài đặt (installer) cho từng hệ điều hành.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Mockoon được thiết kế theo mô hình **Monorepo** sử dụng NPM workspaces, phân tách rõ ràng trách nhiệm giữa các thành phần:

*   **@mockoon/commons:** Thư viện chứa logic lõi, các định nghĩa kiểu dữ liệu (Interfaces/Models) và các hàm chuyển đổi (migrations). Thư viện này dùng chung cho cả App Desktop và CLI.
*   **@mockoon/commons-server:** Chứa logic chuyên biệt cho phía server (Node.js), chịu trách nhiệm thực thi việc khởi tạo và chạy các mock server thực tế.
*   **@mockoon/app:** Ứng dụng Desktop chính.
*   **@mockoon/cli:** Công cụ dòng lệnh để chạy các file cấu hình mock mà không cần giao diện đồ họa (thích hợp cho CI/CD).
*   **@mockoon/serverless:** Gói hỗ trợ triển khai Mockoon trên các môi trường cloud function (AWS Lambda, v.v.).

Kiến trúc này giúp Mockoon có khả năng mở rộng cao, cho phép người dùng thiết kế mock API trên UI và chạy chúng ở bất cứ đâu (local, server, cloud).

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **IPC (Inter-Process Communication):** Sử dụng cơ chế truyền tin giữa tiến trình Main (Node.js) và Renderer (Angular) của Electron để quản lý tài nguyên hệ thống (như ghi file, quản lý server) từ UI.
*   **Data Migration System:** Một kỹ thuật quan trọng giúp ứng dụng tương thích ngược. Khi định dạng dữ liệu JSON thay đổi qua các phiên bản, Mockoon tự động chạy các hàm "migration" để cập nhật file cấu hình cũ của người dùng lên bản mới nhất.
*   **Templating Engine:** Tích hợp các thư viện như `faker.js` và `handlebars` để cho phép người dùng tạo dữ liệu giả động (dynamic data) ngay trong câu phản hồi (response body).
*   **Environment Watching:** Sử dụng `chokidar` để theo dõi sự thay đổi của các file cấu hình, cho phép hot-reload server khi người dùng sửa đổi file bên ngoài ứng dụng.

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Thiết kế (Design):** Người dùng thao tác trên giao diện Angular (renderer process), định nghĩa các Route, Method, Status Code và Body.
2.  **Lưu trữ (Persistence):** Các thay đổi được gửi qua IPC xuống Main process để lưu vào file JSON trên ổ đĩa.
3.  **Khởi động (Execution):** Khi người dùng nhấn "Start", Main process sử dụng `@mockoon/commons-server` để khởi tạo một instance của server Express.js.
4.  **Xử lý Request (Request Handling):** 
    *   Khi có request đến, Express server sẽ đối chiếu với các quy tắc (Rules) mà người dùng thiết lập.
    *   Nếu có templating, engine sẽ parse và sinh dữ liệu ngẫu nhiên.
    *   Phản hồi (Response) được gửi về cho client.
5.  **Ghi nhật ký (Logging):** Mọi request/response được ghi lại và gửi ngược lên Renderer process thông qua IPC để hiển thị trong tab "Logs".

### Tổng kết
Mockoon là một ví dụ điển hình về việc kết hợp **Electron + Angular** để tạo ra một công cụ developer tool chuyên nghiệp. Kỹ thuật chia nhỏ monorepo giúp dự án duy trì tính nhất quán giữa bản Desktop và CLI, đồng thời hệ thống migration mạnh mẽ giúp bảo vệ dữ liệu người dùng qua nhiều năm phát triển.