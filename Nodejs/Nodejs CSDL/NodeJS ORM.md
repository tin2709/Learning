
# 1  Tìm hiểu về MikroORM: ORM mạnh mẽ cho Node.js và TypeScript

**Tác giả:** Tran Thuy Vy (Frontend Developer)
**Ngày đăng:** 26 Oct, 2024 (Lưu ý: Ngày này trong tương lai, có thể bạn muốn điều chỉnh)

ORM (Object-Relational Mapping) đã trở thành công cụ quan trọng giúp lập trình viên dễ dàng làm việc với cơ sở dữ liệu hơn. Trong bài viết này, chúng ta sẽ cùng tìm hiểu chi tiết về MikroORM - một ORM mạnh mẽ dành cho Node.js, dựa trên các mẫu thiết kế Data Mapper, Unit of Work và Identity Map.

## Mục Lục

1.  [MikroORM là gì?](#1-mikroorm-là-gì)
2.  [Các tính năng nổi bật của MikroORM](#2-các-tính-năng-nổi-bật-của-mikroorm)
3.  [Nhược điểm của MikroORM](#3-nhược-điểm-của-mikroorm)
    *   [Độ phức tạp](#31-độ-phức-tạp)
    *   [Tài liệu ít, cộng đồng nhỏ](#32-tài-liệu-ít-cộng-đồng-nhỏ)
4.  [So sánh TypeORM, Sequelize, MikroORM và Prisma](#4-so-sánh-typeorm-sequelize-mikroorm-và-prisma)
5.  [Hướng dẫn cài đặt và sử dụng cơ bản](#5-hướng-dẫn-cài-đặt-và-sử-dụng-cơ-bản)
6.  [Kết luận](#6-kết-luận)

## 1. MikroORM là gì?

**MikroORM** là một ORM (Object-Relational Mapping) cho Node.js được viết bằng TypeScript. Điểm đặc biệt của MikroORM là nó được xây dựng dựa trên các mẫu thiết kế (design patterns) đã được kiểm chứng như:

*   **Data Mapper:** Tách biệt logic truy cập cơ sở dữ liệu khỏi logic nghiệp vụ của ứng dụng. Nó sử dụng một lớp trung gian (Mapper) để ánh xạ dữ liệu giữa đối tượng trong ứng dụng và bảng trong cơ sở dữ liệu. Điều này giúp model không phụ thuộc trực tiếp vào cơ sở dữ liệu, dễ dàng kiểm thử và bảo trì hơn.
*   **Unit of Work:** Quản lý một tập hợp các thay đổi đối với các thực thể (entity) trong một giao dịch (transaction). Nó theo dõi các đối tượng đã được truy xuất từ cơ sở dữ liệu và biết được đối tượng nào đã thay đổi, từ đó cho phép tối ưu hóa số lượng truy vấn xuống cơ sở dữ liệu.
*   **Identity Map:** Đảm bảo rằng trong một phiên làm việc (session), mỗi thực thể chỉ có một đối tượng duy nhất được tạo ra trong bộ nhớ. Điều này ngăn chặn việc tải lại nhiều bản sao của cùng một thực thể, giúp đảm bảo tính nhất quán của dữ liệu và cải thiện hiệu suất.

MikroORM hỗ trợ nhiều loại cơ sở dữ liệu phổ biến, bao gồm PostgreSQL, MySQL, MariaDB, SQLite, MongoDB và SQL Server. Với MikroORM, bạn có thể định nghĩa các model (thực thể) bằng TypeScript, giúp việc tương tác với cơ sở dữ liệu trở nên linh hoạt, an toàn kiểu (type-safe) và dễ quản lý hơn.

## 2. Các tính năng nổi bật của MikroORM

*   **Hỗ trợ đa dạng cơ sở dữ liệu (SQL và NoSQL):** MikroORM không giới hạn bạn ở một loại cơ sở dữ liệu cụ thể.
    *   *Ví dụ cấu hình kết nối với MySQL:*
        ```typescript
        import { Options } from '@mikro-orm/core';
        import { User } from './entities/User'; // Giả sử bạn có entity User

        const config: Options = {
          entities: [User],
          dbName: 'my_database',
          type: 'mysql',
          user: 'root',
          password: 'password',
        };

        export default config;
        ```
    *   *Ví dụ cấu hình sử dụng MongoDB:*
        ```typescript
        import { Options } from '@mikro-orm/core';
        import { User } from './entities/User'; // Giả sử bạn có entity User

        const config: Options = {
          entities: [User],
          clientUrl: 'mongodb://localhost:27017/my_database',
          type: 'mongo',
        };

        export default config;
        ```

*   **An toàn kiểu với TypeScript (Type Safety):** Việc sử dụng TypeScript giúp phát hiện lỗi tiềm ẩn ngay trong quá trình phát triển (compile-time) thay vì lúc chạy (run-time).
    ```typescript
    // Ví dụ về việc tạo một User mới
    import { MikroORM } from '@mikro-orm/core';
    import { User } from './entities/User';
    import config from './mikro-orm.config'; // File cấu hình của bạn

    (async () => {
      const orm = await MikroORM.init(config);
      const em = orm.em.fork(); // EntityManager

      const user = new User();
      user.name = 'John Doe';
      user.email = 'john@example.com';

      // TypeScript sẽ cảnh báo nếu bạn gán sai kiểu dữ liệu
      // user.name = 123; // Lỗi: Type 'number' is not assignable to type 'string'

      await em.persistAndFlush(user); // Lưu user vào CSDL

      await orm.close(true);
    })();
    ```

*   **Query Builder mạnh mẽ:** Cho phép xây dựng các truy vấn phức tạp một cách linh hoạt và dễ đọc.
    ```typescript
    // Ví dụ truy vấn người dùng
    // ... (khởi tạo orm và em như trên)
    const users = await em.find(User, {
      name: { $like: '%200Lab%' }, // Tìm user có tên chứa "200Lab"
    }, {
      orderBy: { createdAt: 'DESC' }, // Sắp xếp theo ngày tạo giảm dần
      limit: 10,                       // Giới hạn 10 kết quả
      offset: 0,                       // Bắt đầu từ kết quả đầu tiên
    });

    console.log(users);
    // ...
    ```

*   **Định nghĩa Schema bằng Decorator:** Sử dụng các decorator của TypeScript để định nghĩa cấu trúc bảng và các mối quan hệ giữa các thực thể một cách trực quan.
    *   *Ví dụ quan hệ One-to-Many giữa `User` và `Post`:*
        ```typescript
        // src/entities/User.ts
        import { Entity, PrimaryKey, Property, OneToMany, Collection } from '@mikro-orm/core';
        import { Post } from './Post';

        @Entity()
        export class User {
          @PrimaryKey()
          id!: number;

          @Property()
          name!: string;

          @OneToMany(() => Post, post => post.author)
          posts = new Collection<Post>(this);
        }
        ```
        ```typescript
        // src/entities/Post.ts
        import { Entity, PrimaryKey, Property, ManyToOne } from '@mikro-orm/core';
        import { User } from './User';

        @Entity()
        export class Post {
          @PrimaryKey()
          id!: number;

          @Property()
          title!: string;

          @ManyToOne(() => User)
          author!: User;
        }
        ```

*   **Dễ dàng tích hợp với các Framework phổ biến:** Có thể tích hợp mượt mà với ExpressJS, Koa, NestJS, v.v.
    *   *Ví dụ tích hợp với ExpressJS:*
        ```typescript
        import express from 'express';
        import { MikroORM, RequestContext } from '@mikro-orm/core';
        import { User } from './entities/User'; // Entity của bạn
        import config from './mikro-orm.config'; // File cấu hình

        (async () => {
          const orm = await MikroORM.init(config);
          const app = express();

          // Đảm bảo mỗi request sử dụng một EntityManager riêng biệt
          app.use((req, res, next) => {
            RequestContext.create(orm.em, next);
          });

          app.get('/users', async (req, res) => {
            const em = orm.em.fork(); // Lấy EntityManager cho request hiện tại
            const users = await em.find(User, {});
            res.json(users);
          });

          app.listen(3000, () => {
            console.log('Server is running on port 3000');
          });
        })();
        ```

## 3. Nhược điểm của MikroORM

### 3.1 Độ phức tạp

*   **Yêu cầu hiểu biết về Design Patterns:** Để tận dụng tối đa sức mạnh của MikroORM, bạn cần có hiểu biết nhất định về Data Mapper, Unit of Work, và Identity Map.
*   **Cấu hình ban đầu:** Một số tùy chọn cấu hình có thể cần sự chú ý và thiết lập chính xác.
*   **Độ trừu tượng cao:** Đối với người mới bắt đầu, mức độ trừu tượng của MikroORM có thể gây khó khăn ban đầu.

### 3.2 Tài liệu ít, cộng đồng nhỏ

*   **Thiếu ví dụ nâng cao:** Đối với các tính năng phức tạp hoặc các trường hợp sử dụng đặc biệt, việc tìm kiếm ví dụ cụ thể có thể khó khăn hơn so với các ORM có cộng đồng lớn hơn.
*   **Hỗ trợ từ cộng đồng:** Do cộng đồng còn đang phát triển, việc tìm kiếm giải pháp cho các vấn đề hoặc lỗi cụ thể có thể mất nhiều thời gian hơn.

## 4. So sánh TypeORM, Sequelize, MikroORM và Prisma

| Tính năng          | TypeORM                      | Sequelize                    | MikroORM                                     | Prisma                         |
| :----------------- | :--------------------------- | :--------------------------- | :------------------------------------------- | :----------------------------- |
| **Ngôn ngữ**       | TypeScript, JavaScript       | JavaScript                   | TypeScript                                   | TypeScript                     |
| **Type-safe**      | Một phần                     | Không                        | **Có**                                       | **Có**                         |
| **Hỗ trợ CSDL**   | MySQL, PostgreSQL, MariaDB,... | MySQL, PostgreSQL, SQLite,... | MySQL, PostgreSQL, MongoDB,...               | MySQL, PostgreSQL, SQLite, SQL Server |
| **Query Builder**  | Có                           | Có                           | **Có (mạnh mẽ)**                             | Sử dụng Prisma Client (fluent API) |
| **Design Pattern** | Active Record & Data Mapper  | Active Record                | **Data Mapper, Unit of Work, Identity Map** | Không tập trung vào pattern cụ thể (thiên về Prisma Client) |
| **Cộng đồng**      | Lớn                          | Rất lớn                      | Đang phát triển                              | Đang phát triển mạnh mẽ         |
| **Độ khó học**    | Trung bình                   | Dễ                           | **Khó (hơn ban đầu)**                       | Dễ                             |
| **Hiệu suất**      | Tốt                          | Tốt                          | **Cao**                                      | **Rất cao**                    |
| **Hỗ trợ TypeScript** | Tốt                          | Hạn chế                      | **Tuyệt vời**                               | **Tuyệt vời**                   |
| **Migration**      | Có                           | Có                           | **Có**                                       | **Có (Prisma Migrate)**        |

## 5. Hướng dẫn cài đặt và sử dụng cơ bản

Trước khi bắt đầu, bạn cần có một dự án TypeScript cơ bản. Nếu chưa có, bạn có thể tham khảo các hướng dẫn tạo dự án Node.js với TypeScript.

**Bước 1: Cài đặt các packages cần thiết**

```bash
npm install @mikro-orm/core @mikro-orm/mysql reflect-metadata
# Hoặc yarn add @mikro-orm/core @mikro-orm/mysql reflect-metadata

# Lưu ý: Thay thế @mikro-orm/mysql bằng driver tương ứng nếu bạn dùng CSDL khác
# Ví dụ: @mikro-orm/postgresql, @mikro-orm/sqlite, @mikro-orm/mongodb
```

**Bước 2: Cấu hình TypeScript (`tsconfig.json`)**

Đảm bảo rằng `tsconfig.json` của bạn có các tùy chọn sau được bật:

```json
{
  "compilerOptions": {
    "target": "ES2017", // Hoặc phiên bản ES mới hơn
    "module": "CommonJS",
    "lib": ["esnext"],
    "strict": true,
    "esModuleInterop": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "outDir": "dist", // Thư mục output
    "rootDir": "./src"  // Thư mục chứa source code
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

**Bước 3: Định nghĩa Thực thể (Entity)**

Tạo thư mục `src/entities`.

*   `src/entities/User.ts`:
    ```typescript
    import { Entity, PrimaryKey, Property, OneToMany, Collection } from '@mikro-orm/core';
    import { Post } from './Post';

    @Entity()
    export class User {
      @PrimaryKey()
      id!: number;

      @Property()
      name!: string;

      @Property({ unique: true })
      email!: string;

      @OneToMany(() => Post, post => post.author)
      posts = new Collection<Post>(this);

      @Property()
      createdAt = new Date();

      @Property({ onUpdate: () => new Date() })
      updatedAt = new Date();
    }
    ```

*   `src/entities/Post.ts`:
    ```typescript
    import { Entity, PrimaryKey, Property, ManyToOne } from '@mikro-orm/core';
    import { User } from './User';

    @Entity()
    export class Post {
      @PrimaryKey()
      id!: number;

      @Property()
      title!: string;

      @Property({ type: 'text' })
      content!: string;

      @ManyToOne(() => User)
      author!: User;

      @Property()
      createdAt = new Date();

      @Property({ onUpdate: () => new Date() })
      updatedAt = new Date();
    }
    ```

**Bước 4: Cấu hình MikroORM**

Tạo file `mikro-orm.config.ts` ở thư mục gốc của dự án:

```typescript
import { Options } from '@mikro-orm/core';
import { User } from './src/entities/User'; // Đường dẫn đúng tới entity
import { Post } from './src/entities/Post'; // Đường dẫn đúng tới entity
import path from 'path';

const config: Options = {
  entities: [User, Post],
  dbName: 'mikroorm_demo', // Tên CSDL của bạn
  type: 'mysql', // Loại CSDL
  user: 'root', // Username CSDL
  password: 'password', // Password CSDL
  debug: process.env.NODE_ENV === 'development', // Bật debug SQL queries ở môi trường dev
  migrations: {
    path: path.join(__dirname, './migrations'), // Thư mục chứa migrations
    pattern: /^[\w-]+\d+\.[tj]s$/, // Pattern cho tên file migration
    transactional: true, // Chạy migration trong transaction
    disableForeignKeys: false, // Không disable foreign key khi chạy migration
    allOrNothing: true, // Nếu một migration lỗi, rollback tất cả
    emit: 'ts', // Tạo file migration bằng TypeScript
  },
};

export default config;
```
*Lưu ý:* Đảm bảo CSDL `mikroorm_demo` đã được tạo và thông tin đăng nhập là chính xác.

**Bước 5: Thiết lập Migration**

Cài đặt CLI cho migrations:
```bash
npm install @mikro-orm/migrations @mikro-orm/cli --save-dev
# Hoặc yarn add @mikro-orm/migrations @mikro-orm/cli -D
```

Thêm script vào `package.json`:
```json
"scripts": {
  "mikro-orm": "mikro-orm"
  // ... các script khác
}
```

Tạo thư mục `migrations` (nếu chưa có):
```bash
mkdir migrations
```

Tạo migration ban đầu để tạo bảng cho các thực thể:
```bash
npx mikro-orm migration:create --initial
# Hoặc yarn mikro-orm migration:create --initial
```

Chạy migration để áp dụng thay đổi vào cơ sở dữ liệu:
```bash
npx mikro-orm migration:up
# Hoặc yarn mikro-orm migration:up
```

**Bước 6: Thêm dữ liệu mẫu (Seeding - Tùy chọn)**

Tạo file `src/seed.ts`:
```typescript
import { MikroORM } from '@mikro-orm/core';
import { User } from './entities/User';
import { Post } from './entities/Post';
import config from '../mikro-orm.config'; // Đường dẫn tới file config

(async () => {
  const orm = await MikroORM.init(config);
  const em = orm.em.fork();

  // Tạo user
  const user = new User();
  user.name = 'Test User';
  user.email = 'test@example.com';

  // Tạo post
  const post1 = new Post();
  post1.title = 'First Seed Post';
  post1.content = 'This is the content of the first seeded post.';
  post1.author = user;

  const post2 = new Post();
  post2.title = 'Second Seed Post';
  post2.content = 'This is the content of the second seeded post.';
  post2.author = user;

  // Lưu vào CSDL
  await em.persistAndFlush([user, post1, post2]);

  console.log('Dữ liệu mẫu đã được thêm thành công.');

  await orm.close(true);
})();
```
Chạy file seed:
```bash
ts-node src/seed.ts
```

**Bước 7: Thực hiện các thao tác CRUD cơ bản**

Tạo file `src/index.ts` (hoặc tên file chính của bạn):
```typescript
import { MikroORM } from '@mikro-orm/core';
import { User } from './entities/User';
import { Post } from './entities/Post';
import config from '../mikro-orm.config';

(async () => {
  const orm = await MikroORM.init(config);
  const em = orm.em.fork();

  // === READ ===
  // Lấy tất cả user và các post liên quan
  const users = await em.find(User, {}, { populate: ['posts'] });
  console.log('Tất cả Users:', JSON.stringify(users, null, 2));

  // === CREATE ===
  const newUser = new User();
  newUser.name = 'Bob The Builder';
  newUser.email = 'bob@example.com';
  await em.persistAndFlush(newUser);
  console.log('User mới được thêm:', newUser);

  // === UPDATE ===
  if (users.length > 0) {
    const firstUser = await em.findOne(User, { id: users[0].id });
    if (firstUser) {
      firstUser.name = 'Alice Wonderland';
      await em.persistAndFlush(firstUser);
      console.log('User đã được cập nhật:', firstUser);
    }
  }

  // === DELETE ===
  const userToDelete = await em.findOne(User, { email: 'bob@example.com' });
  if (userToDelete) {
    await em.removeAndFlush(userToDelete);
    console.log('User đã được xóa:', userToDelete);
  }

  await orm.close(true);
})();
```
Chạy file:
```bash
ts-node src/index.ts
```

**Bước 8: Tích hợp với ExpressJS để tạo API đơn giản**

Cài đặt Express:
```bash
npm install express
npm install @types/express --save-dev
# Hoặc yarn add express && yarn add @types/express -D
```
Tạo file `src/server.ts`:
```typescript
import 'reflect-metadata'; // Cần import ở đầu cho TypeORM/MikroORM
import express from 'express';
import { MikroORM, RequestContext } from '@mikro-orm/core';
import { User } from './entities/User';
import config from '../mikro-orm.config';

const app = express();
app.use(express.json()); // Middleware để parse JSON body

let orm: MikroORM; // Khai báo biến orm ở scope rộng hơn

// Khởi tạo MikroORM một lần khi server bắt đầu
(async () => {
  orm = await MikroORM.init(config);

  // Middleware để tạo RequestContext cho mỗi request
  // Quan trọng: Đặt middleware này sau khi orm đã được khởi tạo
  app.use((req, res, next) => {
    if (!orm) { // Kiểm tra orm đã init chưa
        return res.status(500).send("ORM not initialized");
    }
    RequestContext.create(orm.em, next);
  });

  // Endpoint lấy danh sách người dùng
  app.get('/users', async (req, res) => {
    const em = RequestContext.getEntityManager(); // Lấy EntityManager từ context
    if (!em) return res.status(500).send("EntityManager not found");
    const users = await em.find(User, {}, { populate: ['posts'] });
    res.json(users);
  });

  // Endpoint thêm người dùng mới
  app.post('/users', async (req, res) => {
    const em = RequestContext.getEntityManager();
    if (!em) return res.status(500).send("EntityManager not found");

    try {
      const user = em.create(User, req.body);
      await em.persistAndFlush(user);
      res.status(201).json(user);
    } catch (error) {
      console.error(error);
      res.status(400).json({ message: "Error creating user", error: (error as Error).message });
    }
  });

  // Khởi chạy server
  app.listen(3000, () => {
    console.log('Server is running on http://localhost:3000');
    console.log('Thử GET /users hoặc POST /users');
  });

})().catch(console.error); // Bắt lỗi nếu MikroORM init thất bại
```

Chạy server:
```bash
ts-node src/server.ts
```
Sau đó, bạn có thể truy cập API tại `http://localhost:3000/users`.

*   `GET /users`: Lấy danh sách người dùng.
*   `POST /users`: Thêm người dùng mới (Gửi dữ liệu JSON trong body của request).
    *   Ví dụ sử dụng cURL để thêm người dùng:
        ```bash
        curl -X POST http://localhost:3000/users \
          -H 'Content-Type: application/json' \
          -d '{"name": "Charlie Brown", "email": "charlie@example.com"}'
        ```

**Sử dụng Query Builder để thực hiện truy vấn phức tạp**

Ví dụ, tìm các bài viết có tiêu đề chứa từ "Seed" và sắp xếp theo ngày tạo mới nhất, giới hạn 5 kết quả:
```typescript
// Bên trong một async function có 'em' (EntityManager)
import { Post } from './entities/Post'; // Đảm bảo import Post

// ... (khởi tạo orm và em như trên)
const posts = await em.createQueryBuilder(Post, 'p') // 'p' là alias cho Post
  .select(['p.id', 'p.title', 'p.createdAt']) // Chọn các trường cụ thể
  .where({ title: { $like: '%Seed%' } }) // Điều kiện title chứa 'Seed'
  .orderBy({ createdAt: 'DESC' })        // Sắp xếp
  .limit(5)                               // Giới hạn kết quả
  .getResultList(); // Hoặc .execute() tùy theo phiên bản và nhu cầu

console.log('Các bài viết tìm thấy:', posts);
```

## 6. Kết luận

MikroORM là một ORM mạnh mẽ và hiện đại cho hệ sinh thái Node.js và TypeScript. Mặc dù có thể có một chút đường cong học tập ban đầu do các khái niệm nâng cao mà nó sử dụng, nhưng những lợi ích về hiệu suất, an toàn kiểu và khả năng quản lý code mà nó mang lại là rất đáng kể.

Hy vọng qua bài viết này, bạn đã có cái nhìn tổng quan về MikroORM, cách cài đặt và bắt đầu sử dụng nó trong các dự án TypeScript của mình. Hãy dành thời gian để làm quen, thực hành và khám phá thêm các tính năng của MikroORM để xây dựng các ứng dụng backend hiệu quả và dễ bảo trì.
```

**Lưu ý:**

*   Ngày "26 Oct, 2024" là một ngày trong tương lai. Bạn có thể muốn cập nhật lại ngày này.
*   Đã thêm một số comment và điều chỉnh nhỏ trong các đoạn code ví dụ để rõ ràng hơn.
*   Trong phần tích hợp Express, đã thêm kiểm tra `orm` đã được khởi tạo chưa và sử dụng `RequestContext.getEntityManager()` để lấy `em` một cách an toàn hơn trong các route handler.
*   Đã thêm ví dụ `createQueryBuilder` hoàn chỉnh hơn.
*   Nội dung này được trình bày dưới dạng Markdown, bạn có thể copy trực tiếp vào file `README.md`.