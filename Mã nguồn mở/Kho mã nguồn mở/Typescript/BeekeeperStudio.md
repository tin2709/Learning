Dựa trên nội dung kho lưu trữ của **Beekeeper Studio**, dưới đây là phân tích chi tiết về công nghệ, kiến trúc và luồng hoạt động của dự án:

### 1. Công nghệ cốt lõi (Tech Stack)

Beekeeper Studio là một ứng dụng desktop hiện đại được xây dựng trên nền tảng Web:

*   **Framework chính**: **Electron** (v31/32) – Cho phép chạy ứng dụng Web trên nền tảng Desktop (Windows, macOS, Linux).
*   **Frontend**: 
    *   **Vue.js 2.7**: Sử dụng để xây dựng giao diện người dùng.
    *   **TypeScript**: Ngôn ngữ lập trình chính cho cả quy trình Main và Renderer để đảm bảo an toàn về kiểu dữ liệu.
    *   **Vuex**: Quản lý trạng thái (state management) cho toàn bộ ứng dụng.
    *   **SCSS**: Xử lý giao diện với hệ thống theme đa dạng (Dark, Light, Solarized).
*   **Build Tools**:
    *   **Vite**: Dùng để build quy trình Renderer (giao diện).
    *   **ESBuild**: Dùng để build quy trình Main (logic hệ thống) và Utility.
    *   **Yarn Workspaces**: Quản lý monorepo (chia nhỏ dự án thành `apps/studio`, `apps/ui-kit`, `shared`).
*   **Cơ sở dữ liệu & Truy vấn**:
    *   **TypeORM**: Quản lý cơ sở dữ liệu nội bộ (SQLite) của ứng dụng để lưu trữ lịch sử, kết nối đã lưu.
    *   **Knex.js**: Trình xây dựng truy vấn (Query Builder) cho nhiều loại database.
    *   **Database Drivers**: Hỗ trợ 15+ loại (PostgreSQL, MySQL, SQLite, SQL Server, Cassandra, MongoDB, Redis, v.v.).

### 2. Kỹ thuật và Tư duy Kiến trúc (Architecture)

Dự án áp dụng mô hình **Monorepo** và kiến trúc đa tiến trình của Electron:

*   **Cấu trúc Monorepo**: 
    *   `apps/studio`: Ứng dụng Electron chính.
    *   `apps/ui-kit`: Thư viện các thành phần giao diện dùng chung (Table, SQL Editor).
    *   `shared/src`: Code logic dùng chung giữa các ứng dụng.
*   **Phân tách tiến trình (Process Separation)**:
    *   **Main Process**: Quản lý cửa sổ, thực đơn hệ thống (Native Menu), và các tác vụ đặc quyền.
    *   **Renderer Process**: Chạy ứng dụng Vue.js, xử lý tương tác người dùng.
    *   **Utility Process**: Thực hiện các tác vụ nặng hoặc kết nối database để tránh làm treo giao diện.
*   **Hệ thống Plugin**: Kiến trúc mở cho phép mở rộng tính năng (ví dụ: `bks-ai-shell`).
*   **Mô hình cấp phép (Licensing Model)**: Sử dụng cấu trúc thư mục đặc biệt (`src` cho Community/GPLv3 và `src-commercial` cho các tính năng trả phí) để quản lý đồng thời hai phiên bản trong cùng một repo.
*   **Trình điều khiển DB thống nhất (Unified DB Client)**: Cung cấp một Interface chung (`BaseDatabaseClient`) để mọi database (từ SQL đến NoSQL) đều có thể tương tác với giao diện theo một cách giống nhau.

### 3. Tóm tắt luồng hoạt động (Operational Flow)

Luồng hoạt động của Beekeeper Studio có thể chia thành 3 giai đoạn chính:

#### Giai đoạn 1: Khởi tạo (Startup)
1.  **Main Process** khởi động, kiểm tra cấu hình (`.ini` files).
2.  **Database nội bộ (SQLite)** chạy migration thông qua TypeORM để cập nhật cấu trúc bảng lưu trữ của ứng dụng.
3.  Electron mở cửa sổ Renderer và tải ứng dụng Vue.js.

#### Giai đoạn 2: Kết nối (Connection)
1.  Người dùng nhập thông tin kết nối trong `ConnectionInterface`.
2.  Renderer gửi yêu cầu kết nối qua **IPC (Inter-Process Communication)** đến Main/Utility Process.
3.  Ứng dụng khởi tạo Tunnel (nếu dùng SSH) và tạo kết nối đến database mục tiêu bằng driver tương ứng.
4.  Nếu thành công, trạng thái `connected` được cập nhật trong Vuex, chuyển người dùng sang `CoreInterface`.

#### Giai đoạn 3: Tương tác & Truy vấn (Query & Interaction)
1.  **Trình soạn thảo (SQL Editor)**: Sử dụng CodeMirror với các cấu hình tùy chỉnh cho từng dialect để hỗ trợ autocomplete.
2.  **Thực thi truy vấn**: Câu lệnh SQL được gửi xuống tiến trình nền -> Thực thi trên DB -> Kết quả trả về dưới dạng stream hoặc mảng dữ liệu.
3.  **Hiển thị**: Dữ liệu được Renderer xử lý và hiển thị qua component `Tabulator` (được đóng gói trong `ui-kit`) cho phép sắp xếp, lọc và chỉnh sửa trực tiếp.
4.  **Lưu trữ**: Các hành động như lưu truy vấn hoặc đánh dấu yêu thích sẽ được ghi lại vào database SQLite nội bộ của ứng dụng.

### 4. Điểm đặc biệt khác
*   **Hỗ trợ đa nền tảng**: Quy trình build được cấu hình rất kỹ cho Windows (NSIS), macOS (Notarize), và Linux (AppImage, Snap, Deb, Rpm).
*   **UX/UI Focus**: Beekeeper ưu tiên sự mượt mà và đơn giản, tránh việc nhồi nhét quá nhiều tính năng làm rối giao diện người dùng.

## Supported Databases

<!-- Don't edit this, it gets built automatically from docs/includes/supported_databases.md -->
<!-- SUPPORT_BEGIN -->

| Database                                                 | Support                      | Community | Paid Editions |                             Beekeeper Links |
| :------------------------------------------------------- | :--------------------------- | :-------: | :------: | -----------------------------------------: |
| [PostgreSQL](https://postgresql.org)                     | ⭐ Full Support              |    ✅     |    ✅    |  [Features](https://beekeeperstudio.io/db/postgres-client) |
| [MySQL](https://www.mysql.com/)                          | ⭐ Full Support              |    ✅     |    ✅    |  [Features](https://beekeeperstudio.io/db/mysql-client)|
| [SQLite](https://sqlite.org)                             | ⭐ Full Support              |    ✅     |    ✅    |   [Features](https://beekeeperstudio.io/db/sqlite-client), [Docs](https://docs.beekeeperstudio.io/user_guide/connecting/sqlite) |
| [SQL Server](https://www.microsoft.com/en-us/sql-server) | ⭐ Full Support              |    ✅     |    ✅    |   [Features](https://beekeeperstudio.io/db/sql-server-client)  |
| [Amazon Redshift](https://aws.amazon.com/redshift/)      | ⭐ Full Support              |    ✅     |    ✅    |    [Features](https://beekeeperstudio.io/db/redshift-client) |
| [CockroachDB](https://www.cockroachlabs.com/)            | ⭐ Full Support              |    ✅     |    ✅    | [Features](https://beekeeperstudio.io/db/cockroachdb-client)|
| [MariaDB](https://mariadb.org/)                          | ⭐ Full Support              |    ✅     |    ✅    |     [Features](https://beekeeperstudio.io/db/mariadb-client) |
| [TiDB](https://pingcap.com/products/tidb/)               | ⭐ Full Support              |    ✅     |    ✅    |        [Features](https://beekeeperstudio.io/db/tidb-client) |
| [Google BigQuery](https://cloud.google.com/bigquery)     | ⭐ Full Support             |    ✅      |    ✅    |    [Features](https://beekeeperstudio.io/db/google-big-query-client), [Docs](https://docs.beekeeperstudio.io/user_guide/connecting/bigquery) |
| [Redis](https://redis.io/)                               | ⭐ Full Support               |    ✅    |    ✅    |       [Features](https://www.beekeeperstudio.io/db/redis-client/) |
| [Oracle Database](https://www.oracle.com/database/)      | ⭐ Full Support              |           |    ✅    |      [Features](https://beekeeperstudio.io/db/oracle-client), [Docs](https://docs.beekeeperstudio.io/user_guide/connecting/oracle) |
| [Cassandra](http://cassandra.apache.org/)                | ⭐ Full Support              |           |    ✅    |   [Features](https://beekeeperstudio.io/db/cassandra-client) |
| [Firebird](https://firebirdsql.org/)                     | ⭐ Full Support              |           |    ✅    |    [Features](https://beekeeperstudio.io/db/firebird-client), [Docs](https://docs.beekeeperstudio.io/user_guide/connecting/firebird) |
| [LibSQL](https://libsql.org/)                            | ⭐ Full Support               |          |    ✅    |      [Features](https://beekeeperstudio.io/db/libsql-client) |
| [ClickHouse](https://clickhouse.tech/)                   | ⭐ Full Support                |         |    ✅    |  [Features](https://www.beekeeperstudio.io/db/clickhouse-client/), [Docs](https://docs.beekeeperstudio.io/user_guide/connecting/clickhouse) |
| [DuckDB](https://duckdb.org/)                            | ⭐ Full Support                |         |    ✅    |      [Features](https://www.beekeeperstudio.io/db/duckdb-client/), [Docs](https://docs.beekeeperstudio.io/user_guide/connecting/duckdb) |
| [SQL Anywhere](https://www.sap.com/products/technology-platform/sql-anywhere.html)  | ⭐ Full Support    |           |    ✅    |      [Features](https://www.beekeeperstudio.io/db/sql-anywhere-client/) |
| [MongoDB](https://www.mongodb.com/)                      | ⭐ Full Support               |          |    ✅    |     [Features](https://www.beekeeperstudio.io/db/mongodb-client/), [Docs](https://docs.beekeeperstudio.io/user_guide/connecting/mongodb) |
| [Trino](https://trino.io/) / [Presto](https://prestodb.io/) | ⭐ Full Support                |           |    ✅    |    [Features](https://www.beekeeperstudio.io/db/trino-client/), [Docs](https://docs.beekeeperstudio.io/user_guide/connecting/trino/) |
| [Snowflake](https://www.snowflake.com/)                  | ⏳ Coming Soon                |           |    ✅    |   -- |
| [DynamoDB](https://aws.amazon.com/dynamodb/)             | 🗓️ Planned               |           |    ✅    |       -- |