Dựa trên cấu trúc mã nguồn và các tệp tin tài liệu của **GitButler**, dưới đây là phân tích chuyên sâu về hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)

GitButler là một ứng dụng desktop hiện đại được xây dựng trên nền tảng **Tauri**, kết hợp sức mạnh xử lý hệ thống của Rust và tính linh hoạt giao diện của Web:

*   **Backend (Rust):** Chiếm ~66% mã nguồn. Sử dụng `git2` (libgit2) và đang dần chuyển sang `gix` (Gitoxide - một bản triển khai Git thuần Rust hiệu năng cao). 
    *   **SQLite (`rusqlite`):** Được sử dụng để lưu trữ trạng thái của ứng dụng, cấu hình các nhánh ảo (virtual branches) và lịch sử hoạt động (undo timeline).
    *   **Tokio:** Runtime cho các tác vụ bất đồng bộ (async), xử lý file watcher và giao tiếp mạng.
*   **Frontend (Svelte & TypeScript):** Sử dụng **Svelte 5** (với các tính năng mới như runes `.svelte.ts`) và **Vite**. 
    *   **Tailwind CSS / PostCSS:** Xử lý giao diện với các kỹ thuật nesting tiên tiến.
    *   **IPC (Inter-Process Communication):** Giao tiếp giữa Svelte và Rust thông qua các lệnh `tauri::command`.
*   **AI Integration:** Tích hợp trực tiếp với **Anthropic (Claude)**, **OpenAI** và các mô hình cục bộ như **Ollama**, **LM Studio**.
*   **Build System:** Sử dụng **Turborepo** để quản lý monorepo và **pnpm** làm package manager.

### 2. Tư duy Kiến trúc (Architectural Thinking)

GitButler không chỉ là một GUI cho Git; nó là một **lớp trừu tượng (abstraction layer)** mới bên trên Git:

*   **Virtual Branches (Nhánh ảo):** Đây là tư duy chủ đạo. Thay vì `git checkout` vật lý làm thay đổi file trên đĩa, GitButler cho phép người dùng làm việc trên nhiều nhánh đồng thời trong cùng một thư mục làm việc (working directory) bằng cách quản lý các tập hợp thay đổi (changesets) độc lập.
*   **Modular Monorepo:** Cấu trúc chia nhỏ thành hàng chục crate nhỏ trong `crates/`. 
    *   `but-core`: Chứa logic cốt lõi.
    *   `but-graph`: Phân tích và phân loại đồ thị commit.
    *   `but-db`: Lớp truy xuất dữ liệu SQLite.
    *   `but-api`: Cung cấp API chung cho các client (GUI, CLI).
*   **Project-Centric:** Kiến trúc dựa trên `ProjectHandle`, quản lý mọi thứ theo từng dự án riêng biệt với các cấu hình và database độc lập.

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Type-Safe Bridge:** Sử dụng `ts-rs` hoặc các kỹ thuật macro trong `but-api-macros` để tự động tạo định nghĩa TypeScript từ các struct của Rust, đảm bảo tính nhất nhất quán dữ liệu giữa Backend và Frontend.
*   **Snapshotting & Oplog:** Kỹ thuật ghi lại mọi thay đổi vào một "Operation Log". Điều này cho phép tính năng "Unlimited Undo" (Hoàn tác vô hạn) - một điều cực kỳ khó thực hiện với Git truyền thống.
*   **Hunk Management:** Khả năng tách nhỏ các thay đổi trong cùng một file thành các "hunk" và gán chúng vào các nhánh khác nhau (nhờ logic trong `but-hunk-assignment`).
*   **Reactive State Management:** Ở frontend, dự án tận dụng triệt để kiến trúc **Compound Components** (Controller - Provider - Consumers). Controller là các class `.svelte.ts` giữ logic, Provider cung cấp context, giúp giảm thiểu "prop drilling".

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Quét và Giám sát (Watching):** Khi mở dự án, `gitbutler-watcher` (Rust) sẽ theo dõi mọi thay đổi tệp tin trong thư mục làm việc.
2.  **Đưa vào Trạng thái Tạm thời (Staging/Virtualize):** Khi file thay đổi, thay vì `git add`, GitButler tính toán diff và lưu trữ thông tin hunk vào SQLite dưới dạng các nhánh ảo.
3.  **Chiếu (Projection):** Backend Rust thực hiện việc "chiếu" (projecting) các thay đổi từ database lên giao diện Svelte. Người dùng có thể kéo thả các commit/hunk giữa các "lane" (nhánh ảo) một cách mượt mà.
4.  **Tương tác AI:** Khi người dùng yêu cầu tạo commit message, hệ thống thu thập ngữ cảnh diff, gửi qua `but-llm` đến các provider, và trả kết quả về UI.
5.  **Đẩy dữ liệu (Syncing):** Khi người dùng nhấn "Push", GitButler mới thực hiện các lệnh Git vật lý thực sự để đồng bộ với GitHub/GitLab, chuyển đổi trạng thái ảo thành các commit Git chuẩn.

### Tổng kết
Kiến trúc của GitButler rất phức tạp nhưng có tính module cao. Nó kết hợp khả năng **quản lý trạng thái bằng SQLite** và **xử lý Git bằng Rust** để tạo ra một trải nghiệm người dùng vượt xa Git CLI truyền thống, đặc biệt tối ưu cho các quy trình làm việc có sự hỗ trợ của AI (Agentic workflows).