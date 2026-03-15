Dưới đây là phân tích chi tiết về dự án **mq** (mqlang) dựa trên cấu trúc mã nguồn và thông tin bạn cung cấp:

### 1. Công nghệ cốt lõi (Core Technologies)

Dự án được xây dựng chủ yếu bằng **Rust (93.5%)**, tận dụng tối đa sức mạnh của ngôn ngữ này về hiệu suất và an toàn bộ nhớ.

*   **Ngôn ngữ lập trình:** Rust (phiên bản 1.93.0+).
*   **Phân tích cú pháp (Parsing):** 
    *   Sử dụng `nom` và `nom_locate` để xây dựng bộ phân tích cú pháp (Parser) cho ngôn ngữ truy vấn `mq`.
    *   Sử dụng crate `markdown` để xử lý cấu trúc văn bản Markdown.
*   **Xử lý lỗi & Chẩn đoán:** `miette` được sử dụng xuyên suốt để cung cấp các thông báo lỗi cực kỳ chi tiết, có định dạng đẹp mắt và hướng dẫn sửa lỗi (giống như trình biên dịch Rust).
*   **Giao diện dòng lệnh (CLI):** `clap` để xử lý các tham số dòng lệnh.
*   **Hệ sinh thái công cụ:** 
    *   **LSP (Language Server Protocol):** `tower-lsp-server` để cung cấp tính năng IntelliSense (completion, hover, diagnostics) cho các trình soạn thảo.
    *   **DAP (Debug Adapter Protocol):** Hỗ trợ trình gỡ lỗi trực quan.
    *   **Wasm (WebAssembly):** `wasm-bindgen` để chạy `mq` trực tiếp trên trình duyệt hoặc các môi trường JavaScript (`mq-web`).

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Dự án tuân theo kiến trúc **Monorepo** và tư duy của một **Trình biên dịch/Thông dịch (Compiler-like architecture)**:

*   **Chia nhỏ Module (Crates):** Hệ thống được tách thành các thành phần chuyên biệt:
    *   `mq-markdown`: Chịu trách nhiệm phân tích và thao tác với Markdown.
    *   `mq-lang`: Trái tim của dự án, chứa bộ thông dịch (Interpreter) và các hàm tích hợp (Built-ins).
    *   `mq-hir`: Biểu diễn mã nguồn ở mức cao (High-level Internal Representation) để phân tích ngữ nghĩa.
    *   `mq-check`: Hệ thống kiểm tra kiểu dữ liệu (Type Checker) riêng biệt.
    *   `mq-run`: Điểm vào (entry point) cho ứng dụng CLI.
*   **Pipeline xử lý:** Luồng dữ liệu đi qua các bước: `Truy vấn (Query) -> CST (Concrete Syntax Tree) -> HIR -> Thực thi (Execution)`. 
*   **Tính mở rộng (Extensibility):** Hỗ trợ "External Subcommands". Nếu bạn đặt một file thực thi có tên bắt đầu bằng `mq-` vào thư mục bin, nó sẽ trở thành một lệnh con của `mq`.
*   **Đa nền tảng:** Kiến trúc cho phép `mq` chạy như một CLI độc lập, thư viện C (FFI), WebAssembly, hoặc tích hợp trực tiếp vào các ngôn ngữ khác (Elixir, Python, Go...).

---

### 3. Các kỹ thuật chính (Key Techniques)

*   **Hệ thống kiểu (Type Inference):** `mq-check` thực hiện suy diễn kiểu kiểu **Hindley-Milner**. Nó có khả năng nhận diện các kiểu cơ bản (`number`, `string`, `bool`) và các kiểu phức hợp (`union types`, `record types` với `row polymorphism`).
*   **Thu hẹp kiểu (Type Narrowing):** Kỹ thuật `narrowing.rs` cho phép trình kiểm tra kiểu hiểu được luồng điều kiện. Ví dụ: sau câu lệnh `if (is_string(x))`, hệ thống sẽ biết `x` chắc chắn là một chuỗi trong nhánh đó.
*   **Truy vấn dạng Pipe (Pipe Chains):** Giống như `jq` hoặc shell, dữ liệu được truyền qua các ống dẫn (`|`). Kỹ thuật này được xử lý trong `constraint/pipe.rs` để đảm bảo tính toàn vẹn dữ liệu khi đi qua các hàm biến đổi.
*   **Phân tích Markdown theo cấu trúc:** Thay vì dùng Regex, `mq` coi Markdown là một cấu trúc dữ liệu gồm các nút (`Heading`, `CodeBlock`, `TableCell`). Điều này cho phép thực hiện các truy vấn phức tạp như: `.h | select(contains("Filter"))`.
*   **Phân tích cú pháp gia tăng (Incremental Parsing):** Hỗ trợ xử lý mã nguồn hiệu quả khi người dùng đang gõ (trong LSP).

---

### 4. Tóm tắt luồng hoạt động (Workflow Summary)

1.  **Giai đoạn nạp dữ liệu:** `mq` nhận đầu vào là Markdown (từ file hoặc stdin) và một chuỗi truy vấn.
2.  **Phân tích truy vấn:** 
    *   Chuỗi truy vấn được chuyển thành CST để hiểu cấu trúc ngữ pháp.
    *   CST được chuyển sang HIR để kiểm tra logic và biến (Scoping).
    *   Type Checker (`mq-check`) chạy ngầm để đảm bảo các phép toán (như cộng chuỗi với số) không gây lỗi.
3.  **Xử lý Markdown:** File Markdown được phân tích thành một cây đối tượng nội bộ.
4.  **Thực thi (Engine):** 
    *   Bộ thông dịch (`mq-lang/src/engine.rs`) sẽ duyệt qua cây Markdown.
    *   Các "Selectors" (như `.h`, `.code`) sẽ lọc ra các nút tương ứng.
    *   Các hàm biến đổi (như `map`, `select`) sẽ thay đổi hoặc lọc dữ liệu.
5.  **Xuất dữ liệu:** Kết quả cuối cùng được định dạng lại theo yêu cầu người dùng (Markdown, JSON, HTML hoặc Text) và in ra terminal.

### Kết luận
`mq` là một công cụ mạnh mẽ, được thiết kế với tư duy của một ngôn ngữ lập trình thực thụ dành riêng cho văn bản. Nó giải quyết bài toán xử lý Markdown có cấu trúc — một nhu cầu cực lớn trong thời đại LLM (AI), nơi Markdown là ngôn ngữ giao tiếp chính giữa người và máy.