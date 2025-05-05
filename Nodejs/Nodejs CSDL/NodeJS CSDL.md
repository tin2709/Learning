

# 1 So sánh Raw SQL, Query Builders và ORMs trong Node.js
Sponsor by https://blog.appsignal.com/2025/03/26/how-to-choose-between-sql-query-builders-and-orms-in-nodejs.html?ref=dailydev

Khi xây dựng các ứng dụng Node.js tương tác với cơ sở dữ liệu quan hệ, bạn có rất nhiều công cụ để quản lý và thực thi các truy vấn.

Ba phương pháp phổ biến nhất — Raw SQL (SQL thô), Query Builders (Trình xây dựng truy vấn), và Object-Relational Mappers (ORMs) — đều mang lại những ưu điểm và thách thức riêng, khiến việc quyết định lựa chọn phương pháp nào là tốt nhất trở nên khó khăn.

Trong hướng dẫn này, chúng ta sẽ so sánh điểm mạnh, sự đánh đổi và các trường hợp sử dụng của cả ba phương pháp. Điều này sẽ giúp bạn hiểu rõ sắc thái của từng lựa chọn và xác định phương pháp nào phù hợp nhất với nhu cầu của mình.

Hãy cùng tìm hiểu!

## Tìm hiểu về Raw SQL

Raw SQL đề cập đến việc viết và thực thi trực tiếp các truy vấn SQL vào cơ sở dữ liệu mà không cần sử dụng bất kỳ lớp trừu tượng nào. Trong cách tiếp cận này, bạn tự tay tạo các truy vấn SQL dưới dạng chuỗi văn bản thuần túy và gửi chúng trực tiếp đến cơ sở dữ liệu để thực thi.

<!-- (Tùy chọn: Có thể thêm ảnh chụp màn hình mã Raw SQL nếu cần) -->

Trước khi bạn có thể thực thi truy vấn Raw SQL từ ứng dụng Node.js, bạn cần thiết lập kết nối giữa ứng dụng Node và loại SQL bạn chọn thông qua driver cơ sở dữ liệu thích hợp. Các lựa chọn phổ biến bao gồm:

*   `mysql2` cho MySQL
*   `pg` cho PostgreSQL
*   `better-sqlite3` cho SQLite
*   Và nhiều loại khác!

Sau khi kết nối với cơ sở dữ liệu, bạn có thể thực thi trực tiếp các truy vấn SQL bằng cách sử dụng đối tượng kết nối được cung cấp. Bạn sẽ xây dựng các truy vấn dưới dạng chuỗi, kết hợp các placeholder (trình giữ chỗ) cho bất kỳ giá trị động nào để ngăn chặn các cuộc tấn công SQL injection. Sau đó, bạn sẽ chuyển truy vấn của mình, cùng với bất kỳ tham số cần thiết nào, đến phương thức thực thi truy vấn của driver cơ sở dữ liệu.

Driver sẽ gửi truy vấn đến cơ sở dữ liệu, nhận kết quả và trả về cho ứng dụng của bạn, thường ở dạng một mảng các đối tượng đại diện cho dữ liệu được truy xuất.

Đây là một ví dụ cơ bản sử dụng driver `better-sqlite3` với tệp cơ sở dữ liệu SQLite:

```javascript
import Database from "better-sqlite3";
const db = new Database("chinook.sqlite");

const selectAlbumByID = "SELECT * FROM Album WHERE AlbumId = ?";

// Chuẩn bị câu lệnh và thực thi với tham số
const row = db.prepare(selectAlbumByID).get(1);
console.log(row.AlbumId, row.Title, row.ArtistId);
```

Bây giờ bạn đã hiểu cách Raw SQL hoạt động, hãy cùng đi sâu vào ưu và nhược điểm của việc chỉ sử dụng nó để giao tiếp với cơ sở dữ liệu SQL.

### 👍 Ưu điểm của Raw SQL

*   **Minh bạch và Kiểm soát:** Một trong những ưu điểm chính của việc viết truy vấn Raw SQL là mức độ minh bạch và kiểm soát mà nó cung cấp. Bạn có toàn quyền xem xét từng hoạt động cơ sở dữ liệu, cho phép bạn thấy chính xác cách dữ liệu được lưu trữ, cấu trúc và truy xuất. Cách tiếp cận trực tiếp này giúp giảm bớt những bất ngờ thường đi kèm với các lớp trừu tượng.
*   **Tối ưu hóa Hiệu năng:** Cho phép bạn tạo ra các truy vấn được tối ưu hóa cao, tránh được sự kém hiệu quả có thể xảy ra từ các công cụ tạo truy vấn tự động. Khả năng tinh chỉnh truy vấn đặc biệt hữu ích trong các tình huống yêu cầu truy xuất hoặc thao tác dữ liệu phức tạp.
*   **Linh hoạt Tối đa:** Raw SQL cung cấp sự linh hoạt không giới hạn vì bạn không bị ràng buộc bởi giới hạn của bất kỳ lớp trừu tượng nào. Bạn có thể khai thác toàn bộ khả năng của công cụ cơ sở dữ liệu và chạy các truy vấn phức tạp, dành riêng cho cơ sở dữ liệu mà có thể không được hỗ trợ hoặc khó truy cập thông qua các lớp trừu tượng cấp cao hơn.
*   **Hiểu biết sâu sắc về SQL:** Làm việc độc quyền với Raw SQL sẽ giúp bạn hiểu sâu hơn về SQL và cách cơ sở dữ liệu hoạt động - kiến thức quý giá cho bất kỳ nhà phát triển nào, đặc biệt là khi hiệu năng và tối ưu hóa là mối quan tâm hàng đầu.

### 👎 Nhược điểm của Raw SQL

*   **Phức tạp và Khó bảo trì:** Các truy vấn Raw SQL thường có thể trở nên dài dòng và khó quản lý, đặc biệt khi xử lý các mối quan hệ phức tạp, truy vấn lồng nhau hoặc truy xuất dữ liệu từ nhiều bảng.
*   **Đường cong học tập cao:** Đối với các nhà phát triển không có nền tảng vững chắc về SQL, việc học và sử dụng Raw SQL có thể khá khó khăn. Trong hệ sinh thái Node.js, có xu hướng ưa chuộng ORM, query builder và các lớp trừu tượng tương tự, khiến việc tìm kiếm tài nguyên và hỗ trợ cho các mẫu Raw SQL trở nên thách thức hơn.
*   **Rủi ro Bảo mật:** Viết Raw SQL có thể khiến bạn dễ gặp lỗi và lỗ hổng bảo mật, chẳng hạn như SQL injection, nếu các truy vấn không được làm sạch (sanitize) đúng cách. Bạn cần xử lý thủ công bằng cách sử dụng các truy vấn tham số hóa (parameterized queries) hoặc câu lệnh chuẩn bị (prepared statements) và đảm bảo rằng tất cả đầu vào của người dùng được làm sạch cẩn thận.
    <!-- (Tùy chọn: Có thể thêm ảnh Bobby Tables XCKD nếu cần) -->
    *Nguồn: Bobby Tables XCKD*
*   **Dễ phát sinh lỗi:** Làm việc với Raw SQL thường liên quan đến việc thao tác các truy vấn dưới dạng chuỗi thuần túy, điều này có thể dẫn đến các lỗi tinh vi (như lỗi chính tả tên cột hoặc kiểu dữ liệu không chính xác) mà có thể không được chú ý cho đến khi chạy ứng dụng. Nếu bạn đang sử dụng TypeScript, bạn có thể xem xét các công cụ như [PgTyped](https://github.com/adelsz/pgtyped) để sử dụng Raw SQL trong ứng dụng của mình một cách an toàn về kiểu.

### Ai nên sử dụng Raw SQL?

Raw SQL phù hợp nhất trong các tình huống mà việc tối ưu hóa hiệu năng và kiểm soát chi tiết là tối quan trọng, hoặc khi xử lý các truy vấn phức tạp, không theo tiêu chuẩn mà ORM không thể xử lý dễ dàng.

---

## Tìm hiểu về Query Builders (Trình xây dựng truy vấn)

Thay vì viết Raw SQL, bạn có thể chọn các query builder để tương tác với cơ sở dữ liệu của mình.

<!-- (Tùy chọn: Có thể thêm ảnh chụp màn hình mã Knex.js nếu cần) -->

Chúng cung cấp một cách có cấu trúc và an toàn hơn để soạn thảo các truy vấn, đồng thời loại bỏ một số phức tạp của việc xây dựng chuỗi SQL thủ công.

Query builders thường cung cấp một API nơi bạn có thể kết nối các phương thức (method chaining) để xây dựng các truy vấn phức tạp từng bước. Cách tiếp cận này giúp ngăn chặn các lỗi phổ biến như lỗ hổng SQL injection và đơn giản hóa quá trình kết hợp dữ liệu động vào truy vấn của bạn.

Vẻ đẹp thực sự của query builders là chúng tạo ra sự cân bằng giữa trừu tượng và kiểm soát. Bạn vẫn tương tác với các khái niệm cơ sở dữ liệu quen thuộc như bảng, cột và mối quan hệ, nhưng theo cách thân thiện hơn với JavaScript. Điều này chuyển thành sự an toàn và tiện lợi tăng lên mà không làm mất đi sự hiểu biết rõ ràng về các hoạt động cơ sở dữ liệu cơ bản.

Một query builder nổi bật trong hệ sinh thái Node.js là [Knex.js](https://knexjs.org/). Bạn cần cài đặt gói `knex` và driver cơ sở dữ liệu liên quan cho cơ sở dữ liệu cụ thể mà bạn đang làm việc (`pg`, `mysql`, v.v.).

```shell
npm install knex sqlite3
```

Bây giờ bạn có thể viết các truy vấn như sau:

```javascript
import knex from "knex";

const Database = knex({
  client: "sqlite3",
  connection: {
    filename: "./chinook.sqlite",
  },
  useNullAsDefault: true, // Cần thiết cho SQLite
});

// Sử dụng phương thức của Knex để xây dựng truy vấn
const selectedRow = await Database("Album")
  .where({
    AlbumId: 1,
  })
  .select("*");

console.log(selectedRow);
```

Giá trị của query builders so với Raw SQL có thể không rõ ràng ngay lập tức nếu bạn chủ yếu viết các truy vấn đơn giản, tĩnh như trên. Nhưng chúng có thể nhanh chóng chứng minh giá trị của mình khi cần xây dựng các truy vấn với các điều kiện động:

```javascript
let query = Database("users"); // Sử dụng 'Database' đã khởi tạo ở trên

// Giả sử searchCriteria là một đối tượng chứa các tiêu chí tìm kiếm
if (searchCriteria.name) {
  query = query.where("name", "like", `%${searchCriteria.name}%`);
}

if (searchCriteria.email) {
  query = query.where("email", searchCriteria.email);
}

if (searchCriteria.minAge) {
  query = query.where("age", ">=", searchCriteria.minAge);
}

const results = await query.select("*");
console.log(results);

// Đừng quên đóng kết nối khi hoàn tất
await Database.destroy();
```

Các phương thức có thể kết nối chuỗi của Knex cho phép bạn dễ dàng xây dựng các truy vấn phức tạp dựa trên các điều kiện thời gian chạy. So sánh điều này với Raw SQL, nơi việc đạt được kết quả tương tự thông qua việc nối chuỗi không chỉ kém tiện lợi hơn mà còn dễ bị tấn công bảo mật.

### 👍 Ưu điểm của Query Builders

*   **Xây dựng truy vấn động dễ dàng:** Đơn giản hóa việc xây dựng các truy vấn phức tạp dựa trên điều kiện thời gian chạy.
*   **Giảm thiểu SQL Injection:** Giảm thiểu rủi ro tấn công SQL injection thông qua việc sử dụng các truy vấn tham số hóa tích hợp sẵn.
*   **Khả năng bảo trì tốt hơn:** Dễ bảo trì hơn trong dài hạn so với chuỗi Raw SQL. Bằng cách sử dụng các cấu trúc lập trình quen thuộc như kết nối phương thức, việc soạn thảo các truy vấn phức tạp thành các phần dễ quản lý và phân biệt giữa toán tử và dữ liệu trở nên dễ dàng hơn, trong khi vẫn giữ đúng ngữ nghĩa SQL.
    ```javascript
    Database("users") // Sử dụng 'Database' đã khởi tạo
      .select("users.id", "users.name", "posts.title")
      .join("posts", "users.id", "posts.author_id")
      .where("posts.published", true)
      .orderBy("posts.created_at", "desc")
      .then(rows => console.log(rows))
      .catch(err => console.error(err))
      .finally(() => Database.destroy()); // Đóng kết nối
    ```
*   **Minh bạch:** Không giống như ORM, query builders cung cấp sự minh bạch về truy vấn SQL cơ bản. Mặc dù chúng sử dụng các phương thức để đại diện cho các nguyên tắc SQL, các hoạt động cơ sở dữ liệu cơ bản không bị che khuất, vì vậy bất kỳ ai quen thuộc với SQL vẫn có thể hiểu được mục đích và các tác động hiệu năng tiềm ẩn của truy vấn.
*   **Hỗ trợ nhiều CSDL:** Query builders thường hỗ trợ nhiều backend, cho phép bạn viết mã dễ di chuyển hơn giữa các hệ thống cơ sở dữ liệu khác nhau. Mặc dù backend cơ sở dữ liệu hiếm khi thay đổi khi ứng dụng đã sản xuất, tính năng này cho phép các nhà phát triển làm việc với các cơ sở dữ liệu khác nhau tránh phải học một mô hình mới để viết SQL cho từng loại.
*   **Khuyến khích học SQL:** Mặc dù một số người chỉ trích query builders vì không trừu tượng hóa đủ sự phức tạp của SQL, nhưng đây lại là một điểm mạnh. Việc chỉ dựa vào các công cụ bỏ qua việc học SQL sẽ gây hại về lâu dài. Query builders, khi được sử dụng hiệu quả, vẫn đòi hỏi sự hiểu biết cơ bản về các nguyên tắc SQL. Chúng cung cấp một môi trường có cấu trúc, an toàn để viết SQL nhằm cải thiện khả năng bảo trì, mà không làm mất đi ngữ nghĩa cốt lõi của SQL.

### 👎 Nhược điểm của Query Builders

*   **Ít nhược điểm so với Raw SQL:** Ngay cả khi xử lý các hoạt động đòi hỏi truy vấn mà builder không cung cấp lớp trừu tượng, thường có chế độ "raw" để gửi truy vấn trực tiếp đến backend, bỏ qua giao diện thông thường của query builder.
    ```javascript
    Database("users") // Sử dụng 'Database' đã khởi tạo
      .select("*")
      .where(Database.raw("(age > ? OR email LIKE ?)", [18, "%@gmail.com"]))
      .then(rows => console.log(rows))
      .catch(err => console.error(err))
      .finally(() => Database.destroy()); // Đóng kết nối
    ```
*   **Hiệu năng:** Về mặt hiệu năng, query builders thường theo kịp Raw SQL, mặc dù có thể có những tình huống mà các truy vấn Raw SQL được tối ưu hóa tỉ mỉ bằng tay có thể hiệu quả hơn một chút.
*   **So với ORM:** Sự đánh đổi chính nằm ở việc so sánh với ORM. Query builders, với lớp trừu tượng ít hơn, đòi hỏi sự hiểu biết sâu hơn về các khái niệm SQL và quản lý lược đồ. Bạn sẽ bỏ lỡ các tiện ích như ánh xạ quan hệ đối tượng tự động, di chuyển lược đồ (schema migrations) và giảm mã soạn sẵn (boilerplate code) mà ORM thường cung cấp.

### Ai nên sử dụng Query Builders?

Query builders phù hợp với bất kỳ ai tìm kiếm sự cân bằng giữa khả năng kiểm soát của Raw SQL và sự tiện lợi của ORM. Chúng là một lựa chọn tuyệt vời nếu bạn muốn duy trì sự minh bạch về các khái niệm SQL cơ bản trong khi theo đuổi một phương pháp có cấu trúc và dễ bảo trì hơn.

Nếu bạn thường xuyên làm việc với nhiều cơ sở dữ liệu, query builder cũng cung cấp một giao diện nhất quán để xây dựng các truy vấn, bất kể hệ thống cơ bản là gì. Điều này làm giảm chi phí chuyển đổi ngữ cảnh khi bạn chuyển từ dự án này sang dự án khác. Và bạn luôn có thể sử dụng `raw()` khi cần truy cập các khả năng dành riêng cho cơ sở dữ liệu.

---

## Tìm hiểu về Object Relational Mappers (ORMs)

ORMs bắc cầu khoảng cách giữa lập trình hướng đối tượng và cơ sở dữ liệu quan hệ bằng cách cung cấp một mức độ trừu tượng cao.

<!-- (Tùy chọn: Có thể thêm ảnh chụp màn hình mã Sequelize ORM nếu cần) -->

Chúng trình bày dữ liệu theo mô hình hướng đối tượng, giảm đáng kể lượng mã soạn sẵn cần thiết và do đó tăng tốc độ phát triển.

Bằng cách truy cập và thao tác dữ liệu dưới dạng đối tượng, ORM giúp giảm bớt nhu cầu viết SQL thủ công. Chúng dịch các hoạt động hướng đối tượng thành các lệnh SQL mà cơ sở dữ liệu có thể hiểu, cho phép bạn tập trung nhiều hơn vào logic nghiệp vụ thay vì sự phức tạp của cơ sở dữ liệu.

Nhiều ORM cũng cung cấp các tính năng tích hợp để quản lý lược đồ cơ sở dữ liệu của bạn, chẳng hạn như tạo bảng, định nghĩa mối quan hệ và xử lý di chuyển lược đồ (schema migrations) khi ứng dụng của bạn phát triển và thay đổi.

Có rất nhiều ORM có sẵn cho Node.js. [Sequelize](https://sequelize.org/) là một lựa chọn lâu đời, nhưng các lựa chọn thay thế mới hơn như [Prisma](https://www.prisma.io/), [MikroORM](https://mikro-orm.io/) và [Drizzle ORM](https://orm.drizzle.team/) đang ngày càng phổ biến do nhấn mạnh vào trải nghiệm nhà phát triển, an toàn kiểu và hiệu năng.

Giống như query builders, bạn cần cài đặt cả ORM và driver cơ sở dữ liệu thích hợp:

```shell
npm install sequelize sqlite3
```

Sau khi cài đặt, bạn cần định nghĩa các `model` đại diện cho các bảng và mối quan hệ trong cơ sở dữ liệu của bạn. Sau đó, bạn có thể sử dụng các phương thức của ORM để tương tác với cơ sở dữ liệu — thực hiện truy vấn, tạo, cập nhật hoặc xóa dữ liệu — tất cả trong khi quản lý lược đồ một cách dễ dàng.

```javascript
import { DataTypes, Sequelize } from "sequelize";

const sequelize = new Sequelize({
  dialect: "sqlite",
  storage: "chinook.sqlite", // Đường dẫn đến file SQLite
  logging: false // Tắt logging cho ví dụ ngắn gọn
});

// Định nghĩa model Album tương ứng với bảng Album
const Album = sequelize.define(
  "Album",
  {
    AlbumId: {
      type: DataTypes.INTEGER,
      primaryKey: true,
      autoIncrement: true,
    },
    Title: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    ArtistId: {
      type: DataTypes.INTEGER,
      allowNull: false,
      // Sequelize tự động hiểu đây là khóa ngoại nếu có model Artist
    },
  },
  {
    timestamps: false, // Không sử dụng cột createdAt, updatedAt
    tableName: "Album", // Chỉ định rõ tên bảng
  }
);

// Sử dụng phương thức của Sequelize để tìm Album theo khóa chính
async function findAlbum() {
  try {
    const album = await Album.findByPk(1);
    if (album) {
      console.log(album.AlbumId, album.Title, album.ArtistId);
    } else {
      console.log("Album không tìm thấy.");
    }
  } catch (error) {
    console.error("Lỗi khi tìm album:", error);
  } finally {
    await sequelize.close(); // Đóng kết nối
  }
}

findAlbum();
```

Tóm lại, ORM cung cấp một lớp trừu tượng mạnh mẽ giúp đơn giản hóa tương tác cơ sở dữ liệu. Điều này giúp các nhà phát triển dễ dàng quản lý các mối quan hệ phức tạp và tự động hóa các tác vụ tẻ nhạt như di chuyển lược đồ, trong khi vẫn cung cấp sự linh hoạt và kiểm soát khi cần thiết.

### 👍 Ưu điểm của ORMs

*   **Mức độ trừu tượng cao:** Cho phép tương tác cơ sở dữ liệu bằng các khái niệm hướng đối tượng quen thuộc thay vì Raw SQL.
*   **Tăng tốc độ phát triển:** Giảm đáng kể lượng mã soạn sẵn cần thiết, đơn giản hóa các hoạt động cơ sở dữ liệu phổ biến (như CRUD - Create, Read, Update, Delete).
*   **Quản lý lược đồ và Migrations:** Các công cụ tích hợp giúp quản lý phiên bản cơ sở dữ liệu và di chuyển lược đồ, giảm thiểu rủi ro lỗi.
*   **Quản lý mối quan hệ phức tạp:** Tự động hóa việc ánh xạ và điều hướng các mối quan hệ (một-nhiều, nhiều-nhiều), giúp định nghĩa và sử dụng chúng dễ dàng hơn trong mã.
*   **Bảo mật:** Giảm thiểu các lỗ hổng bảo mật phổ biến như SQL injection theo mặc định, vì chúng tự động xử lý việc xây dựng truy vấn và làm sạch đầu vào người dùng.
*   **Tính di động:** Cung cấp một lớp tương tác không phụ thuộc vào cơ sở dữ liệu, nghĩa là bạn có thể chuyển đổi giữa các hệ thống cơ sở dữ liệu khác nhau với thay đổi mã tối thiểu.
*   **An toàn kiểu và Tối ưu hóa (ORM hiện đại):** Nhiều ORM hiện đại cung cấp độ an toàn kiểu mạnh mẽ và các tối ưu hóa hiệu năng, giúp nhà phát triển phát hiện lỗi sớm và tối ưu hóa tương tác cơ sở dữ liệu (ví dụ: Prisma, MikroORM, Drizzle).

### 👎 Nhược điểm của ORMs

*   **Chi phí hiệu năng (Performance Overhead):** ORM thường có thể tạo ra các truy vấn không hiệu quả (ngay cả đối với các tác vụ đơn giản) do thiết kế rộng rãi của chúng để phù hợp với nhiều trường hợp sử dụng. Điều này có thể dẫn đến hiệu năng chậm hơn đáng kể, đặc biệt trong các ứng dụng có lưu lượng truy cập cao hoặc khi xử lý các hoạt động dữ liệu phức tạp.
*   **Mất kiểm soát chi tiết:** Mặc dù ORM xuất sắc trong việc đơn giản hóa các tác vụ phổ biến, việc tối ưu hóa các truy vấn cụ thể, mô hình hóa các mối quan hệ phức tạp hoặc tận dụng các tính năng nâng cao dành riêng cho cơ sở dữ liệu có thể khó khăn trong giới hạn của ORM.
*   **Bất tương xứng trở kháng đối tượng-quan hệ (Object-Relational Impedance Mismatch):** Những thách thức này thường bắt nguồn từ sự ma sát giữa mô hình hướng đối tượng được sử dụng trong mã ứng dụng và bản chất dựa trên bảng của cơ sở dữ liệu quan hệ. Sự không phù hợp này thường đòi hỏi các giải pháp tạm thời và thỏa hiệp khi quản lý các mối quan hệ phức tạp trong cơ sở dữ liệu.
*   **Đường cong học tập:** Bạn cần hiểu cú pháp, quy ước cụ thể của ORM và cách nó ánh xạ đối tượng vào bảng cơ sở dữ liệu. Nếu bạn đã quen thuộc với SQL, việc thích nghi với mô hình của ORM đôi khi có thể cảm thấy như một lớp phức tạp bổ sung, vì bạn phải quản lý cả logic nội bộ của cơ sở dữ liệu và ORM.

### Ai nên sử dụng ORMs?

ORMs thường phù hợp cho các tình huống mà bạn ưu tiên tốc độ phát triển hơn là hiệu năng thời gian chạy. Chúng cũng là lựa chọn tốt để xây dựng các ứng dụng mà các truy vấn phức tạp là tối thiểu và các hoạt động CRUD là phổ biến.

---

## Khám phá Cách tiếp cận Kết hợp (Hybrid Approach)

Cho đến nay, chúng ta chủ yếu xem xét việc lựa chọn giữa Raw SQL, query builders và ORM như một đề xuất "hoặc là/hoặc là". Tuy nhiên, trên thực tế, bạn có thể áp dụng một cách tiếp cận kết hợp bằng cách kết hợp điểm mạnh của nhiều phương pháp để phù hợp với dự án hiện tại.

*   **ORM + Raw SQL/Query Builder:** Một chiến lược kết hợp phổ biến là dựa vào ORM cho hầu hết các truy cập dữ liệu, trong khi sử dụng Raw SQL hoặc query builder cho các truy vấn quan trọng về hiệu năng hoặc khi tận dụng các tính năng dành riêng cho cơ sở dữ liệu mà ORM không dễ dàng hỗ trợ. Ví dụ, bạn có thể sử dụng ORM cho các hoạt động CRUD tiêu chuẩn và mô hình hóa mối quan hệ, nhưng chuyển sang Raw SQL/query builder cho các phép nối (JOIN), tổng hợp (aggregation) phức tạp hoặc các hoạt động cơ sở dữ liệu chuyên biệt.
*   **Query Builder + Raw SQL:** Một biến thể khác liên quan đến việc sử dụng query builder làm giao diện chính cho các hoạt động cơ sở dữ liệu. Điều này cho phép khả năng bảo trì và soạn thảo dễ dàng hơn trong khi vẫn giữ được ngữ nghĩa SQL và cho phép linh hoạt chuyển sang Raw SQL khi cần thiết.

Cách tiếp cận kết hợp mang lại cho bạn những gì tốt nhất của cả hai thế giới: sự tiện lợi của lớp trừu tượng bạn chọn, cùng với hiệu năng và khả năng kiểm soát của việc truy cập cơ sở dữ liệu trực tiếp khi cần.

---

## Tổng kết

Trong hệ sinh thái Node.js, không có câu trả lời duy nhất phù hợp cho tất cả khi nói đến tương tác cơ sở dữ liệu.

*   **Raw SQL** cung cấp khả năng kiểm soát và hiệu năng vô song nhưng đòi hỏi chuyên môn cao.
*   **Query Builders** cung cấp sự cân bằng giữa tiện lợi và linh hoạt.
*   **ORMs** ưu tiên sự trừu tượng và phát triển nhanh chóng.

Cuối cùng, sự lựa chọn tốt nhất phụ thuộc vào nhu cầu cụ thể của dự án, chuyên môn của nhóm bạn và những đánh đổi mà bạn sẵn sàng chấp nhận. Bất kể con đường bạn chọn là gì, **một sự hiểu biết vững chắc về SQL vẫn là nền tảng** cho việc quản lý cơ sở dữ liệu quan hệ hiệu quả.

