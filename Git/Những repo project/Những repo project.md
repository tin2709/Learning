Repo project: https://github.com/caotrongtin99/evm-app-boilerplate

Đây là một boilerplate (khuôn mẫu) đầy đủ cho việc xây dựng các ứng dụng phi tập trung (DApp) dựa trên EVM (Ethereum Virtual Machine). Nó được thiết kế như một **monorepo**, nghĩa là nhiều dự án con (ứng dụng và thư viện) được quản lý trong cùng một kho lưu trữ Git.

### 1. Kiến trúc tổng thể (Architecture)

Monorepo này được tổ chức thành hai thư mục chính: `apps/` (các ứng dụng có thể triển khai) và `packages/` (các thư viện dùng chung).

*   **`apps/`**: Chứa các ứng dụng độc lập, mỗi ứng dụng phục vụ một mục đích cụ thể trong DApp:
    *   **`apps/web` (Frontend)**:
        *   Đây là giao diện người dùng chính của DApp.
        *   Sử dụng **Next.js 15** với App Router (cách tổ chức routing và fetching dữ liệu mới của Next.js) và **React 19** cho giao diện.
        *   Tích hợp **Web3**, cho phép người dùng tương tác với blockchain (ví dụ: kết nối ví, gửi giao dịch).
    *   **`apps/api` (Backend)**:
        *   Đây là phần backend API của DApp.
        *   Sử dụng **NestJS** (một framework Node.js mạnh mẽ) để xây dựng API.
        *   Có khả năng tương tác với blockchain (ví dụ: lấy dữ liệu từ blockchain, xử lý logic phức tạp không phù hợp cho frontend).
    *   **`apps/subgraph` (Subgraph)**:
        *   Đây là một giải pháp để **lập chỉ mục (indexing) dữ liệu blockchain**.
        *   Khi blockchain tạo ra rất nhiều dữ liệu, việc truy vấn trực tiếp rất chậm. Subgraph giúp biến dữ liệu blockchain thành dữ liệu có thể truy vấn nhanh chóng bằng **GraphQL**.
        *   Ví dụ, thay vì quét từng block để tìm một sự kiện nào đó, Subgraph đã "nghe" và lưu trữ các sự kiện đó vào một cơ sở dữ liệu có thể truy vấn dễ dàng.

*   **`packages/`**: Chứa các thư viện và cấu hình dùng chung để tránh lặp lại code và đảm bảo tính nhất quán:
    *   **`packages/ui` (UI Package)**:
        *   Một thư viện chứa các thành phần giao diện người dùng (UI components) có thể tái sử dụng.
        *   Dựa trên **`shadcn/ui`**, một bộ sưu tập các component UI được xây dựng trên **Tailwind CSS v4** và **Radix UI** (một thư viện cung cấp các nguyên tắc cơ bản cho việc xây dựng UI có khả năng truy cập cao).
        *   Giúp duy trì giao diện nhất quán và phát triển nhanh hơn cho frontend.
    *   **`packages/web3` (Web3 Package)**:
        *   Một thư viện chứa các tiện ích (utilities), hooks (cho React), và các định nghĩa ABI (Application Binary Interface) dùng chung cho các tương tác Web3.
        *   Hỗ trợ **RainbowKit** (giao diện kết nối ví dễ dùng) và **Wagmi** (bộ sưu tập hooks React để tương tác với Ethereum).
        *   Giúp chuẩn hóa cách tương tác với blockchain trên toàn bộ DApp.
    *   **`packages/eslint-config` (ESLint Configs)**:
        *   Chứa các cấu hình ESLint dùng chung. ESLint là một công cụ giúp nhận diện và báo cáo các vấn đề trong code JavaScript/TypeScript.
        *   Đảm bảo chất lượng code và phong cách code nhất quán trong toàn bộ monorepo.
    *   **`packages/typescript-config` (TypeScript Configs)**:
        *   Chứa các cấu hình TypeScript dùng chung. TypeScript giúp thêm kiểu dữ liệu vào JavaScript, tăng cường độ tin cậy và khả năng bảo trì của code.
        *   Đảm bảo việc kiểm tra kiểu dữ liệu và biên dịch code TypeScript nhất quán giữa các dự án con.
    *   **`packages/ipfs` (IPFS Package)**:
        *   `README.md` đề cập đến một package IPFS tại đây, nhưng như đã giải thích trước đó, trong cấu trúc thư mục được cung cấp, không có thư mục `packages/ipfs` riêng biệt.
        *   Trên thực tế, tính năng IPFS được tích hợp trực tiếp vào `apps/web` bằng cách sử dụng thư viện `pinata-sdk` và API Routes của Next.js để xử lý các hoạt động IPFS một cách an toàn và hiệu quả.

### 2. Bắt đầu nhanh (Quick Start)

Phần này hướng dẫn các bước cơ bản để cài đặt và chạy dự án:
*   **Điều kiện tiên quyết:** Cần Node.js phiên bản 20 trở lên và pnpm phiên bản 8 trở lên.
*   **Cài đặt:** Các lệnh cơ bản để clone repo, di chuyển vào thư mục dự án, và cài đặt tất cả các phụ thuộc của monorepo bằng `pnpm install`.
*   **Biến môi trường:** Hướng dẫn sao chép file `.env.local.example` thành `.env.local` và chỉnh sửa các biến môi trường cần thiết (ví dụ: Project ID của WalletConnect, Port của API, URL của Gateway IPFS, v.v.). Đây là bước quan trọng để cấu hình các dịch vụ bên ngoài.
*   **Phát triển (Development):**
    *   `pnpm dev`: Khởi động cả frontend và backend cùng lúc ở chế độ phát triển (hot-reloading, debugging).
    *   `pnpm dev --filter=web`: Chỉ khởi động frontend.
    *   `pnpm dev --filter=api`: Chỉ khởi động backend.
*   **URLs:** Cung cấp các địa chỉ để truy cập ứng dụng frontend, backend API, tài liệu API Swagger và trang demo IPFS.

### 3. Ngăn xếp công nghệ (Tech Stack)

Liệt kê chi tiết các công nghệ được sử dụng ở từng phần:

*   **Frontend (`apps/web`):**
    *   **Next.js 15 + App Router:** Framework React cho các ứng dụng web tối ưu SEO và hiệu suất.
    *   **React 19:** Thư viện JavaScript để xây dựng giao diện người dùng.
    *   **TypeScript:** Ngôn ngữ lập trình giúp tăng cường sự an toàn và khả năng bảo trì code.
    *   **Tailwind CSS v4:** Framework CSS tiện ích (utility-first) để xây dựng UI nhanh chóng và linh hoạt.
    *   **shadcn/ui:** Bộ sưu tập các component UI có sẵn để tùy chỉnh và tích hợp dễ dàng.
    *   **RainbowKit:** Bộ công cụ giúp kết nối ví tiền điện tử với ứng dụng React một cách dễ dàng và đẹp mắt.
    *   **Wagmi:** Bộ hooks React mạnh mẽ để tương tác với Ethereum.
    *   **Viem:** Thư viện JavaScript nhẹ và hiệu quả để tương tác với blockchain EVM.
    *   **IPFS integration:** Tích hợp IPFS cho lưu trữ phi tập trung (thông qua Pinata như đã phân tích).

*   **Backend (`apps/api`):**
    *   **NestJS framework:** Framework Node.js tiến bộ, hỗ trợ TypeScript, giúp xây dựng các ứng dụng phía server hiệu quả và có cấu trúc.
    *   **TypeScript:** Tương tự như frontend, giúp code backend an toàn và dễ bảo trì hơn.
    *   **Viem:** Dùng cho các tương tác blockchain ở phía server.
    *   **Swagger:** Công cụ tạo tài liệu API tự động, giúp các nhà phát triển dễ dàng hiểu và sử dụng các endpoint API.
    *   **Security (Helmet, CORS, Rate limiting):** Các biện pháp bảo mật cơ bản để bảo vệ API khỏi các cuộc tấn công phổ biến.

*   **Shared Packages:**
    *   **@workspace/ui:** Các component UI có thể tái sử dụng.
    *   **@workspace/web3:** Các tiện ích và hooks liên quan đến Web3.
    *   **@workspace/eslint-config:** Cấu hình ESLint chung.
    *   **@workspace/typescript-config:** Cấu hình TypeScript chung.

### 4. Tính năng Web3 (Web3 Features)

Đi sâu vào các khả năng blockchain của DApp:
*   **Hỗ trợ đa chuỗi (Multi-chain support):** Tương thích với các mạng lưới phổ biến như Ethereum, Polygon, Arbitrum, Optimism, Base.
*   **Kết nối ví (Wallet connection):** Dễ dàng kết nối ví thông qua RainbowKit.
*   **Truy vấn số dư token (Token balance queries):** Lấy số dư của ETH và các token ERC20.
*   **Theo dõi giao dịch (Transaction monitoring):** Giúp theo dõi trạng thái các giao dịch trên blockchain.
*   **Tương tác hợp đồng thông minh (Smart contract interactions):** Gửi giao dịch và đọc dữ liệu từ các smart contract.
*   **Hỗ trợ mạng thử nghiệm (Testnet support):** Cho phép phát triển và thử nghiệm trên các mạng testnet trước khi triển khai lên mainnet.

### 5. Tính năng IPFS (IPFS Features)

Phần này mô tả chi tiết các tính năng IPFS (thực tế được triển khai trong `apps/web`):
*   **Tải file lên (File Upload):** Tải bất kỳ loại file nào lên IPFS.
*   **Truy xuất nội dung (Content Retrieval):** Lấy và hiển thị nội dung từ các hash IPFS (CID).
*   **Metadata NFT (NFT Metadata):** Tạo và tải metadata NFT tương thích với OpenSea.
*   **Quản lý Pin (Pin Management):** Pin/unpin nội dung để lưu trữ dai dẳng (persistent storage) trên các dịch vụ IPFS.
*   **React Hooks:** Các hooks React dễ sử dụng cho các thao tác IPFS (mặc dù thực tế nằm trong `apps/web` chứ không phải package riêng).
*   **UI Components:** Các component UI dựng sẵn để tải file và hiển thị nội dung.
*   **Nhiều nhà cung cấp (Multiple Providers):** Hỗ trợ các nhà cung cấp và gateway IPFS khác nhau (trong boilerplate này là Pinata).

### 6. Cấu trúc dự án (Project Structure)

Phần này lặp lại cấu trúc thư mục đã được đề cập, nhưng có thêm các mô tả ngắn gọn về vai trò của từng thư mục/file quan trọng. Cần lưu ý rằng mô tả về `apps/web/components/ipfs/` và `apps/web/src/app/demo/ipfs-complete/` phản ánh việc IPFS được tích hợp vào frontend, chứ không phải một package riêng biệt.

### 7. Thêm Component UI (Adding UI Components)

Hướng dẫn cách thêm các component `shadcn/ui` mới vào dự án, chỉ rõ rằng chúng sẽ được đặt trong `packages/ui/src/components`.

### 8. Sử dụng Shared Packages (Using Shared Packages)

Ví dụ minh họa cách import và sử dụng các component UI và hooks Web3 từ các package dùng chung, cho thấy cách monorepo giúp tái sử dụng code.

### 9. Triển khai (Deployment)

Hướng dẫn cơ bản cho việc triển khai ứng dụng:
*   **Frontend (Vercel):** Triển khai ứng dụng Next.js lên Vercel.
*   **Backend (Railway/Heroku/Docker):** Triển khai ứng dụng NestJS lên các nền tảng đám mây hoặc qua Docker.

### 10. Biến môi trường (Environment Variables)

Liệt kê các biến môi trường quan trọng cần được cấu hình trong file `.env.local` hoặc `.env.development`.

### 11. Đóng góp (Contributing)

Hướng dẫn quy trình đóng góp cho dự án (fork, tạo branch, commit, push, tạo Pull Request).

### 12. Giấy phép (License)

Thông tin về giấy phép của dự án (MIT License).

**Tóm lại:** "EVM DApp Boilerplate" là một bộ công cụ toàn diện và có cấu trúc tốt để xây dựng DApp. Nó tận dụng monorepo để quản lý nhiều ứng dụng (frontend, backend, subgraph) và các thư viện dùng chung (UI, Web3, cấu hình ESLint/TypeScript), tất cả đều tập trung vào công nghệ hiện đại, khả năng tái sử dụng và quy trình phát triển hiệu quả. Mặc dù có một điểm không nhất quán nhỏ giữa `README.md` và cấu trúc thư mục thực tế về "IPFS Package", nhưng chức năng IPFS vẫn được triển khai mạnh mẽ và an toàn.