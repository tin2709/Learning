Dựa trên mã nguồn bạn cung cấp, đây là phân tích chi tiết về kiến trúc và kỹ thuật của công cụ **create-gas-app**:

### 1. Công nghệ cốt lõi (Core Tech Stack)
*   **CLI Engine:** Sử dụng `@clack/prompts` để tạo giao diện dòng lệnh tương tác (interactive UI) đẹp mắt và `picocolors` để định dạng màu sắc.
*   **Build Tooling (CLI):** Sử dụng `tsup` để đóng gói mã nguồn TypeScript của chính CLI thành mã thực thi Node.js (ESM).
*   **Build Tooling (Generated Project):** 
    *   **Vite:** Trái tim của quá trình phát triển, xử lý HMR (Hot Module Replacement) và bundling.
    *   **Vite-plugin-singlefile:** Kỹ thuật then chốt để gộp toàn bộ CSS/JS vào một file HTML duy nhất — yêu cầu bắt buộc của Google Apps Script (GAS) cho các dialog/sidebar.
*   **Runtime Libraries:** 
    *   `gas-client`: Wrapper giúp gọi hàm server GAS bằng Promise và hỗ trợ TypeScript.
    *   `clasp`: Công cụ chính thức của Google để push/pull code giữa máy cục bộ và server Google.

### 2. Tư duy Kiến trúc (Architectural Thinking)
**create-gas-app** áp dụng kiến trúc **Monorepo (Workspaces)** cho các dự án được tạo ra, giúp giải quyết sự phức tạp của việc phát triển Web trong môi trường hạn chế của GAS:

*   **Chia tách Logic (Separation of Concerns):**
    *   `packages/server`: Chứa mã TypeScript chạy trên server GAS (với các global như `SpreadsheetApp`).
    *   `apps/`: Chứa các ứng dụng frontend (Sidebars, Dialogs).
    *   `packages/shared`: Nơi quan trọng nhất, chứa các định nghĩa Type chung để frontend có thể "biết" các hàm trên server có tham số và kiểu trả về là gì (Full-stack Type Safety).
    *   `packages/ui`: Thư viện component dùng chung.
*   **Tính linh hoạt (Framework Agnostic):** Kiến trúc generator được thiết kế theo dạng template pattern, cho phép hỗ trợ React, Vue, Svelte, SolidJS với cùng một cấu trúc thư mục và quy trình build.

### 3. Các kỹ thuật chính (Key Techniques)

#### A. Local Dev Bridge (Kỹ thuật "Cầu nối")
Đây là kỹ thuật sáng tạo nhất của project. GAS không hỗ trợ chạy local server trực tiếp trong iframe của nó.
*   **Giải pháp:** Project push một file HTML "wrapper" (`dev-dialog-bridge.html`) lên server Google. File này chứa một `<iframe>` trỏ về `https://localhost:3000`.
*   **PostMessage:** Các lệnh gọi `google.script.run` từ app local sẽ được gửi qua `postMessage` tới file wrapper, file wrapper thực thi lệnh thật trên GAS và gửi kết quả ngược lại. Điều này cho phép lập trình viên dùng **Live Reload** mà không cần push code liên tục.

#### B. Import Maps & Externalization
Để giữ file HTML inlined nhỏ gọn và tận dụng bộ nhớ đệm (cache):
*   Project sử dụng `<script type="importmap">` để tải các thư viện lớn như React, Vue từ **esm.sh** (CDN).
*   Trong `vite.config.ts`, các thư viện này được đánh dấu là `external`, giúp tốc độ build cực nhanh và file `.html` cuối cùng chỉ chứa code nghiệp vụ của bạn.

#### C. Strip GAS Exports
GAS yêu cầu các hàm server-side phải là hàm global (`function myFunction()`) và không hỗ trợ từ khóa `export` trong file bundle cuối cùng.
*   **Kỹ thuật:** Project sử dụng một plugin Vite tùy chỉnh (`strip-gas-exports` trong `vite.config.ts`) để dùng Regex xóa bỏ câu lệnh `export { ... }` ở cuối file bundle sau khi Rollup đã đóng gói xong.

#### D. Granular OAuth & Type Inference
*   Hỗ trợ quản lý Scope OAuth minh bạch qua file `appsscript.json`.
*   Tự động map kiểu dữ liệu từ `packages/server` sang frontend thông qua `import type * as ServerFunctions`.

### 4. Tóm tắt luồng hoạt động (Operational Flow)

1.  **Scaffolding (Khởi tạo):** CLI thu thập cấu hình từ người dùng $\rightarrow$ Chạy các generator (`generateBase`, `generateServer`, `generateClient`,...) để ghi file xuống đĩa $\rightarrow$ Cài đặt dependencies.
2.  **Development (Phát triển):** 
    *   Chạy `npm run dev`.
    *   Vite chạy HTTPS server tại máy local.
    *   `clasp` đẩy bản build "bridge" lên Google.
    *   Người dùng mở Google Sheets/Docs, thấy giao diện local với HMR.
3.  **Deployment (Triển khai):**
    *   Chạy `npm run deploy`.
    *   Vite build server thành `code.js` (xóa exports).
    *   Vite build từng dialog thành các file `.html` độc lập (inlined mọi thứ).
    *   `clasp push` toàn bộ folder `dist/` lên Google.

### Đánh giá:
Đây là một boilerplate cực kỳ hiện đại, giải quyết được hầu hết các "nỗi đau" (pain points) khi làm việc với Google Apps Script như: thiếu Type Safety, quy trình debug chậm chạp, và khó khăn khi sử dụng các framework UI hiện đại. Công nghệ này biến GAS từ một môi trường scripting đơn giản thành một nền tảng phát triển ứng dụng chuyên nghiệp.