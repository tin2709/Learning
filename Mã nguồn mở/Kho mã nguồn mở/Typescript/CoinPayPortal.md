Dựa trên mã nguồn và tài liệu của dự án **CoinPayPortal**, dưới đây là phân tích chi tiết về dự án hạ tầng thanh toán đa chuỗi này:

---

### 1. Công nghệ cốt lõi (Core Tech Stack)

Dự án được xây dựng theo mô hình Fullstack hiện đại, tối ưu cho cả người dùng cá nhân và các tác nhân AI (AI Agents):

*   **Framework chính:** **Next.js (v15/16)** với App Router và **TypeScript**. Sử dụng cả Server Components và API Routes để xử lý logic backend.
*   **Cơ sở dữ liệu & Auth:** **Supabase (PostgreSQL)**. Tận dụng triệt để tính năng **Row Level Security (RLS)** để bảo mật dữ liệu ở mức database.
*   **Blockchain Libraries:**
    *   **EVM (ETH/POL):** `viem`, `ethers.js v6`.
    *   **Solana:** `@solana/web3.js`.
    *   **Bitcoin/BCH:** `bitcoinjs-lib`, `ecpair`, `tiny-secp256k1`.
*   **Lightning Network:** **Greenlight (CLN)** - cung cấp các nút Lightning được quản lý cho merchant, hỗ trợ giao thức **BOLT12 (Offers)**.
*   **Thanh toán truyền thống:** Tích hợp **Stripe Connect** để hỗ trợ thanh toán qua thẻ tín dụng song song với crypto.
*   **Quản lý ví:** Sử dụng **BIP32/BIP39/BIP44** để tạo ví HD (Hierarchical Deterministic) và **AES-256-GCM** để mã hóa khóa bí mật tại client/server.
*   **Testing:** **Vitest** với hơn 2.800 test cases, đảm bảo độ tin cậy cực cao cho các giao dịch tài chính.

---

### 2. Tư duy Kiến trúc (Architectural Thinking)

Kiến trúc của CoinPayPortal tập trung vào tính **Phi tập trung (Non-custodial)** và **Khả năng tương tác máy-với-máy**:

*   **Triết lý Non-custodial:** Hệ thống không giữ tiền của người dùng. Đối với ví web, khóa bí mật được tạo và mã hóa hoàn toàn ở phía Client. Đối với cổng thanh toán, tiền từ khách hàng được gửi vào một địa chỉ tạm thời và tự động chuyển tiếp (Forwarding) về ví của Merchant sau khi trừ phí platform.
*   **Giao thức x402:** Đây là một điểm sáng kiến trúc, triển khai tiêu chuẩn HTTP 402 (Payment Required). Nó cho phép các AI Agent tự động thanh toán để truy cập API thông qua cơ chế `x402fetch`.
*   **Hệ thống uy tín DID (CPTL):** Xây dựng một lớp uy tín di động dựa trên **Decentralized Identifiers (DIDs)**. Uy tín không dựa trên đánh giá chủ quan mà dựa trên các "biên lai hành động" (ActionReceipts) từ các giao dịch ký quỹ (escrow) thực tế.
*   **Cấu trúc Monorepo:** Quản lý cả mã nguồn Portal (Dashboard) và gói SDK (`packages/sdk`) trong một kho lưu trữ để đồng bộ hóa logic nghiệp vụ.

---

### 3. Kỹ thuật lập trình chính (Key Programming Techniques)

*   **HD Wallet Derivation:** Sử dụng kỹ thuật phái sinh địa chỉ duy nhất cho mỗi giao dịch thanh toán để dễ dàng theo dõi trạng thái mà không cần yêu cầu người dùng nhập "Memo" hay "Tag".
*   **Secure Forwarding & Fee Splitting:** Kỹ thuật chia tách thanh toán ngay khi giao dịch được xác nhận trên chuỗi (ví dụ: 99.5% cho merchant và 0.5% cho platform), thực hiện qua các script tự động có cơ chế retry (exponential backoff).
*   **Webhook Signature Verification:** Sử dụng **HMAC-SHA256** để ký và xác thực các thông báo webhook, đảm bảo merchant nhận đúng dữ liệu từ CoinPay.
*   **Replay Protection:** Sử dụng cơ chế Nonce-based cho các thanh toán x402 để ngăn chặn việc gửi lại cùng một giao dịch nhiều lần.
*   **Client-side Encryption:** Mọi dữ liệu nhạy cảm của ví web được xử lý bằng Web Crypto API trong trình duyệt, đảm bảo "Your keys, your coins".

---

### 4. Luồng hoạt động hệ thống (Operational Workflow)

Quy trình thanh toán tiêu chuẩn diễn ra qua 6 bước:

1.  **Khởi tạo:** Merchant gọi API (hoặc SDK) để tạo một yêu cầu thanh toán (ví dụ: $100 qua ETH).
2.  **Tạo địa chỉ:** Hệ thống dựa vào Master Public Key của merchant để phái sinh một địa chỉ ví ETH duy nhất cho đơn hàng đó.
3.  **Hiển thị:** Trình duyệt hiển thị mã QR (BIP21/EIP-681) cho khách hàng.
4.  **Giám sát (Monitoring):** Một worker chạy ngầm (Vercel Crons/Railway) liên tục quét blockchain qua các RPC provider (Alchemy/Infura) để phát hiện giao dịch vào địa chỉ đó.
5.  **Xác nhận & Chuyển tiếp:** Khi giao dịch đạt đủ số lượng xác nhận (Confirmations), hệ thống thực hiện 2 việc:
    *   Gửi lệnh chuyển tiền (minus fee) từ ví tạm về ví chính của merchant.
    *   Kích hoạt Webhook thông báo cho hệ thống của merchant đơn hàng đã hoàn tất.
6.  **Cập nhật uy tín:** Sau khi giao dịch kết thúc (đặc biệt là qua Escrow), hệ thống tự động cập nhật Trust Score cho DID của các bên tham gia.

### Tổng kết
**CoinPayPortal** là một hệ thống hạ tầng thanh toán toàn diện, không chỉ giải quyết bài toán chấp nhận tiền điện tử cho e-commerce mà còn tiên phong trong việc xây dựng nền kinh tế cho AI Agents thông qua các giao thức như x402 và uy tín phi tập trung (CPTL).