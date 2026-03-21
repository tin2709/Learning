Dựa trên tài liệu và cấu trúc mã nguồn của dự án **SubQuery (subql)**, dưới đây là phân tích chi tiết về các khía cạnh kỹ thuật, kiến trúc và vận hành của hệ thống:

### 1. Công nghệ cốt lõi (Core Technologies)
SubQuery được xây dựng như một bộ công cụ (SDK) toàn diện để xử lý dữ liệu blockchain:

*   **TypeScript (99.4%):** Ngôn ngữ chủ đạo, đảm bảo tính an toàn về kiểu dữ liệu (type-safe) cho toàn bộ hệ thống từ CLI đến Indexer.
*   **oclif Framework:** Sử dụng để xây dựng giao diện dòng lệnh (`@subql/cli`), hỗ trợ quản lý lệnh, plugin và tự động tạo tài liệu.
*   **NestJS:** Framework chính được sử dụng cho `@subql/node` (Indexer) và `@subql/query` (Query Service), tận dụng kiến trúc Module và Dependency Injection để quản lý logic phức tạp.
*   **GraphQL:** Ngôn ngữ truy vấn chính cho người dùng cuối. Dự án sử dụng các plugin PostGraphile để tự động tạo API GraphQL từ cấu trúc cơ sở dữ liệu.
*   **Cơ sở dữ liệu:** Sử dụng **PostgreSQL** kết hợp với **Sequelize** (hoặc TypeORM/Prisma tùy thành phần) để lưu trữ dữ liệu đã index và quản lý metadata.
*   **Blockchain SDKs:** Tích hợp sâu với `@polkadot/api` (cho Substrate) và `ethers.js` (cho Ethereum/EVM) để tương tác với các nút mạng.
*   **MCP (Model Context Protocol):** Một công nghệ mới được tích hợp để cho phép các công cụ AI (như Cursor, VSCode) tương tác trực tiếp với dự án SubQuery.
*   **IPFS:** Sử dụng để lưu trữ và phân phối các gói dự án (Manifest, Mapping, Schema) một cách phi tập trung.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của SubQuery được thiết kế theo hướng **Modular (Mô-đun hóa)** và **Chain-Agnostic (Không phụ thuộc chuỗi)**:

*   **Monorepo Strategy:** Quản lý nhiều gói trong một kho mã nguồn duy nhất (Yarn Workspaces), giúp đồng bộ hóa các thay đổi giữa `cli`, `node-core`, và `common`.
*   **Phân tách Core và Implementation:** 
    *   `node-core`: Chứa logic xử lý cốt lõi, hàng đợi lấy block, và quản lý store không phụ thuộc vào blockchain cụ thể.
    *   `node-substrate` (hoặc các gói mạng khác): Chứa logic đặc thù để parse dữ liệu từ từng mạng cụ thể.
*   **Kiến trúc dựa trên Manifest:** Mọi dự án SubQuery được định nghĩa qua file `project.yaml` (hoặc `project.ts`). Đây là "nguồn chân lý" duy nhất định nghĩa mạng lưới, điểm bắt đầu và các hàm xử lý dữ liệu.
*   **Kiến trúc Plugin/Processor:** Cho phép cộng đồng mở rộng khả năng xử lý các loại dữ liệu mới (ví dụ: Custom Data Sources cho các hợp đồng thông minh phức tạp).

### 3. Kỹ thuật lập trình chính (Main Programming Techniques)
*   **Code Generation (Codegen):** Kỹ thuật quan trọng nhất. Từ file `schema.graphql`, hệ thống tự động tạo ra các class model trong TypeScript. Điều này giúp lập trình viên có trải nghiệm IntelliSense tốt và tránh lỗi khi thao tác với DB.
*   **Sandbox Execution:** Các hàm mapping của người dùng được chạy trong một môi trường sandbox (VM2 hoặc tương đương) để đảm bảo an toàn và tính cô lập, tránh việc mã độc ảnh hưởng đến hệ thống indexer chính.
*   **Versioned Manifests:** Sử dụng các lớp quản lý phiên bản (ví dụ: `v1_0_0/models.ts`) để đảm bảo tính tương thích ngược khi đặc tả dự án thay đổi theo thời gian.
*   **Adapter Pattern:** CLI sử dụng các adapter để chuyển đổi logic giữa chế độ dòng lệnh truyền thống và giao thức MCP cho AI.
*   **Multi-chain Rewind logic:** Kỹ thuật xử lý việc đảo ngược (re-org) block trên nhiều chuỗi cùng lúc để đảm bảo tính nhất quán của dữ liệu.

### 4. Luồng hoạt động hệ thống (System Workflow)

Quy trình hoạt động điển hình của SubQuery chia làm 2 giai đoạn:

#### Giai đoạn Phát triển (Developer Workflow):
1.  **Init:** Người dùng chạy `subql init` để tạo project từ template (EJS templates).
2.  **Codegen:** Chạy `subql codegen` để đồng bộ Schema GraphQL với các thực thể trong code.
3.  **Build:** Chạy `subql build` (sử dụng Webpack/ESBuild) để đóng gói mã nguồn thành một file `.js` duy nhất có thể chạy được.
4.  **Publish:** Đẩy project lên IPFS để mạng lưới SubQuery Network có thể nhận diện.

#### Giai đoạn Vận hành (Indexer Workflow):
1.  **Fetch:** `node-core` lấy danh sách các block/event/transaction từ RPC node của blockchain dựa trên bộ lọc trong Manifest.
2.  **Dispatch:** Các block được đưa vào hàng đợi và điều phối đến các worker process.
3.  **Mapping:** Hệ thống gọi các hàm mapping (do người dùng viết) để biến đổi dữ liệu thô từ block thành các "Entity".
4.  **Store:** Dữ liệu được lưu vào PostgreSQL thông qua lớp Store Operations.
5.  **Query:** Người dùng sử dụng `@subql/query` để truy vấn dữ liệu đã index thông qua endpoint GraphQL.

### Tổng kết
SubQuery là một hệ thống tinh vi kết hợp giữa **công nghệ CLI mạnh mẽ**, **kiến trúc phần mềm phân lớp chặt chẽ** và **khả năng mở rộng đa chuỗi**. Việc chuyển dịch từ cấu hình YAML sang TypeScript Manifest và tích hợp MCP cho thấy dự án đang hướng tới việc tối ưu hóa tối đa trải nghiệm của nhà phát triển và khả năng tự động hóa bằng AI.