Dựa trên mã nguồn của dự án **Jujutsu (jj)**, dưới đây là phân tích chi tiết về kiến trúc và công nghệ của hệ thống quản lý phiên bản (VCS) thế hệ mới này.

### 1. Công nghệ cốt lõi (Core Technologies)

*   **Ngôn ngữ lập trình:** **Rust** (chiếm 99.2%). Jujutsu tận dụng tối đa tính an toàn bộ nhớ, hiệu năng cực cao và khả năng xử lý song song của Rust để giải quyết các thuật toán VCS phức tạp.
*   **Git Interoperability:** Sử dụng thư viện **gitoxide (gix)** — một bản triển khai Git thuần Rust hiện đại, thay thế cho `libgit2` để đạt hiệu suất và tính linh hoạt cao hơn.
*   **Hệ thống xử lý bất đồng bộ:** Sử dụng **Tokio** và **Futures** cho các tác vụ I/O, đặc biệt là khi làm việc với các hệ thống file phân tán hoặc remote backend.
*   **Serialization & Storage:** Sử dụng **Protocol Buffers (Prost)** để lưu trữ cấu trúc dữ liệu nội bộ như Index và Operation Store. Việc này giúp đảm bảo tính tương thích ngược và hiệu suất đọc/ghi cao.
*   **Giao diện người dùng:** 
    *   **Clap**: Xử lý dòng lệnh (CLI).
    *   **Pest**: Parser generator được dùng để xây dựng các ngôn ngữ truy vấn nội bộ như `revsets` (truy vấn commit) và `filesets` (truy vấn file).
    *   **Ratatui**: Dùng cho giao diện đồ họa terminal (TUI).
*   **Hệ thống Build & Nix:** Sử dụng **Nix Flakes** để đảm bảo môi trường phát triển nhất quán và có thể tái lập trên mọi máy tính của các nhà phát triển.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Jujutsu tách biệt hoàn toàn giữa **Mô hình dữ liệu logic** và **Lớp lưu trữ vật lý**:

*   **Tách biệt Backend (Storage Abstraction):** Jujutsu không bị ràng buộc vào định dạng của Git. Nó định nghĩa các `trait Backend` và `trait Store`. Hiện tại Git là backend chính, nhưng kiến trúc này sẵn sàng cho các hệ thống như Mercurial hoặc Google Piper.
*   **Working-copy-as-a-commit:** Đây là tư duy đột phá. Trong Git, working directory là một thực thể nằm ngoài cây commit. Trong `jj`, các thay đổi chưa commit được tự động snapshot thành một "commit ảo". Điều này loại bỏ khái niệm `index/staging area` và `git stash`.
*   **Conflicts as First-class Objects:** Jujutsu coi xung đột (conflict) là một trạng thái hợp lệ của dữ liệu. Thay vì dừng lại và bắt người dùng giải quyết ngay lập tức để tạo commit, `jj` cho phép bạn commit cả những tệp đang xung đột. Thông tin xung đột được lưu trữ trong cây (tree), cho phép truyền bá (propagate) và tự động giải quyết lại khi rebase.
*   **Hệ thống Operation Log:** Jujutsu lưu lại lịch sử của chính nó. Mỗi lệnh bạn chạy (`commit`, `rebase`, `pull`) là một "Operation". Điều này cho phép tính năng **Undo** mạnh mẽ (như một cỗ máy thời gian cho repo).

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **Domain-Specific Languages (DSL):** Jujutsu tự xây dựng ngôn ngữ truy vấn mạnh mẽ (`revsets`). Kỹ thuật này giúp người dùng chọn lọc commit bằng các biểu thức phức tạp như `ancestors(main) & ~immutable()`.
*   **Immutable Data Structures:** Các đối tượng như Commit, Tree, Revset thường được xử lý dưới dạng bất biến, giúp việc tính toán rebase và hiển thị đồ thị (graph) an toàn trong môi trường đa luồng.
*   **Lazy Loading & Caching:** Sử dụng **LRU Cache (clru)** và kỹ thuật nạp dữ liệu lười để xử lý các kho mã nguồn khổng lồ mà không làm tràn bộ nhớ.
*   **Snapshotting:** Jujutsu sử dụng các kỹ thuật quét hệ thống file hiệu quả (dựa trên `ignore` crate) để phát hiện thay đổi và tạo snapshot tự động mà không gây trễ cho người dùng.
*   **Content-addressable Storage:** Tương tự Git nhưng được mở rộng. Jujutsu băm (hash) không chỉ nội dung file mà cả cấu trúc xung đột và lịch sử thao tác.

### 4. Luồng hoạt động hệ thống (System Workflow)

1.  **Giai đoạn Quan sát (Snapshot):** Khi người dùng chạy bất kỳ lệnh nào, `jj` tự động quét thư mục làm việc, phát hiện các file thay đổi và cập nhật vào commit "working copy" hiện tại.
2.  **Giai đoạn Giải quyết Truy vấn (Revset Resolution):** Hệ thống phân tích các đối số dòng lệnh (ví dụ: `@-`, `main`) thông qua DSL parser để xác định các commit mục tiêu trong đồ thị DAG.
3.  **Giai đoạn Thao tác Đồ thị (Graph Manipulation):** Nếu thực hiện lệnh `rebase` hoặc `describe`, Jujutsu sẽ tính toán lại đồ thị commit. Nhờ thiết kế "tự động rebase", các commit con sẽ tự động được di chuyển theo cha của chúng.
4.  **Giai đoạn Đồng bộ vật lý (Backend Sync):** Jujutsu dịch các đối tượng logic của nó sang đối tượng Git (nếu đang dùng Git backend) và ghi xuống thư mục `.git/objects`.
5.  **Giai đoạn Ghi nhật ký (Op Log):** Trạng thái mới của repository sau lệnh được ghi vào `operation log`, cho phép người dùng `jj undo` nếu kết quả không như ý.
6.  **Giai đoạn Hiển thị (Rendering):** Kết quả được định dạng qua một hệ thống template tùy chỉnh và in ra terminal với màu sắc và ký tự đồ họa (Unicode symbols).

### Tổng kết
Jujutsu là một dự án có kiến trúc **đề cao tính trừu tượng và sự an toàn**. Nó giải quyết những "nỗi đau" kinh niên của Git (xung đột khi rebase, mất dữ liệu làm việc, staging area phức tạp) bằng cách áp dụng các nguyên lý khoa học máy tính hiện đại vào cấu trúc dữ liệu VCS. Đây là một ví dụ điển hình về cách Rust được sử dụng để xây dựng các công cụ hệ thống có độ tin cậy cực cao.