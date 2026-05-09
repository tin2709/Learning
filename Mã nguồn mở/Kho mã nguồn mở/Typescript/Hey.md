Dựa trên cấu trúc thư mục và nội dung các file chiến lược của dự án **Hey** (trước đây là Lenster), dưới đây là phân tích chuyên sâu về hệ thống mạng xã hội phi tập trung này:

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

Dự án này là một đại diện tiêu biểu cho sự kết hợp giữa **Web2 truyền thống hiệu năng cao** và **Web3 tiên tiến**:

*   **Web3 & Blockchain:**
    *   **Lens Protocol:** Trục xương sống của mạng xã hội, quản lý Social Graph (mối quan hệ, bài đăng, follow) dưới dạng on-chain.
    *   **Viem & Wagmi:** Bộ đôi thư viện mạnh mẽ nhất hiện nay để tương tác với Ethereum và các layer 2 (dự án dùng Base và Polygon).
    *   **Zora & Lens SDK:** Tích hợp sâu các tính năng đặc thù như đúc (mint) Creator Coin và quản lý danh tính số.
*   **Frontend Engine:**
    *   **React 19 & Vite 7:** Sử dụng phiên bản mới nhất của React, tận dụng các cải tiến về hiệu năng và Concurrent Mode.
    *   **Tailwind CSS v4:** Dự án đã cập nhật lên phiên bản mới nhất của Tailwind (sử dụng `@tailwindcss/vite`), cho phép xử lý CSS ngay trong quá trình build của Vite.
    *   **Prosekit:** Một framework hiện đại xây dựng trên ProseMirror để xử lý trình soạn thảo văn bản giàu tính năng (Rich Text Editor), hỗ trợ markdown và mention (@).
*   **Data & State Management:**
    *   **Apollo GraphQL:** Dùng để truy vấn dữ liệu từ Lens Indexer (một lượng lớn file `.graphql` và `codegen.ts`).
    *   **TanStack React Query:** Quản lý trạng thái server-side cho các API không phải GraphQL (như Zora SDK).
    *   **Zustand & React Tracked:** Quản lý trạng thái ứng dụng (UI state, auth) một cách tối giản nhưng hiệu quả, tránh re-render thừa.

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của Hey được thiết kế theo hướng **Modular Monorepo** với sự phân cấp rõ ràng:

*   **Phân tách Ứng dụng và Logic (Apps vs. Packages):**
    *   `apps/web`: Toàn bộ giao diện người dùng.
    *   `apps/api`: Sử dụng **Hono** - một framework siêu nhẹ, chạy được trên Edge (Cloudflare Workers), đóng vai trò làm Proxy hoặc xử lý các logic tập trung (như metadata preparation).
    *   `packages/*`: Chứa các cấu hình dùng chung, kiểu dữ liệu (types) và helpers để đảm bảo tính nhất quán giữa Web và API.
*   **Kiến trúc Thành phần dựa trên Tên miền (Domain-Driven Component):** Thay vì chia theo kiểu `atoms/molecules`, các component được gom nhóm theo tính năng mạng xã hội: `Account/`, `Composer/`, `Notification/`, `Post/`. Điều này giúp việc mở rộng tính năng (ví dụ: thêm loại post mới) trở nên rất dễ dàng.
*   **Offline-First & Persistence:** Sử dụng `persisted` stores trong Zustand để lưu trữ phiên đăng nhập và các tùy chỉnh người dùng, giúp trải nghiệm mượt mà ngay cả khi tải lại trang.

### 3. Kỹ thuật Lập trình Đặc sắc (Coding Techniques)

*   **Xử lý Giao dịch On-chain Phức tạp:** Trong `src/hooks/`, các hook như `useTransactionLifecycle` và `useWaitForTransactionToComplete` được thiết kế để xử lý trạng thái bất đồng bộ của blockchain: Gửi giao dịch -> Chờ đào block -> Chờ Indexer cập nhật dữ liệu.
*   **Trình soạn thảo tùy biến (Custom Editor):** Việc sử dụng Prosekit thay vì các input đơn giản cho thấy sự đầu tư vào UX. Hệ thống mention (@account, g/group) được xử lý thông qua các plugin của Prosekit, ánh xạ trực tiếp từ dữ liệu GraphQL.
*   **Type-Safety Tuyệt đối:** Sử dụng GraphQL Codegen để tạo ra các TypeScript types từ schema của Lens. Điều này giúp lập trình viên biết chính xác cấu trúc dữ liệu của một bài đăng hay một profile mà không cần xem tài liệu.
*   **Sử dụng Biome:** Thay thế cho ESLint/Prettier, dự án dùng **Biome** để đạt tốc độ lint/format cực nhanh, phù hợp với quy mô monorepo lớn.

### 4. Luồng Hoạt động Hệ thống (System Flow)

1.  **Luồng Đăng nhập (Authentication):** Người dùng kết nối ví (Wagmi) -> Ký một thông báo (Challenge) để xác thực -> Nhận JWT từ API Lens -> Lưu vào persisted store -> Apollo Client đính kèm token này vào header cho các yêu cầu sau.
2.  **Luồng Tạo Bài đăng (Post Creation):**
    *   Người dùng soạn thảo (Prosekit).
    *   Metadata được chuẩn bị theo chuẩn Lens Metadata.
    *   Tệp tin (ảnh/video) được tải lên **IPFS/Arweave** (thông qua Lens Storage Client).
    *   Giao dịch được gửi lên mạng blockchain (có thể thông qua *Sponsored Transaction* - Hey trả phí gas hộ người dùng).
    *   Sau khi giao dịch thành công, ứng dụng chờ Lens Indexer phản hồi bài đăng mới để cập nhật UI.
3.  **Luồng Hiển thị Feed:** Apollo truy vấn dữ liệu từ Indexer -> Dữ liệu đi qua các `fragments` (ví dụ: `Post.graphql`) -> Component `SinglePost` render dữ liệu dựa trên `MainContentFocus` (ảnh, video, hay text).

### Tổng kết
**bigint-hey** là một ví dụ mẫu mực về **DApp (Decentralized Application)** hiện đại. Nó không cố gắng đưa mọi thứ lên blockchain mà chỉ đưa "Social Graph" và "Content Ownership", trong khi vẫn giữ trải nghiệm người dùng nhanh nhạy bằng cách sử dụng các công nghệ Web2 mạnh mẽ nhất (Vite 7, Hono, React 19).