Tuyệt vời, đây là nội dung bài viết được viết lại dưới dạng file README Markdown bằng tiếng Việt:

# 1)  8 Mẹo hàng đầu khi sử dụng Prisma ORM với MongoDB

README này tổng hợp những mẹo quan trọng nhất để giúp bạn sử dụng Prisma ORM một cách hiệu quả khi làm việc với cơ sở dữ liệu MongoDB. MongoDB là cơ sở dữ liệu dạng tài liệu (document-based) và có những đặc thù riêng so với các cơ sở dữ liệu quan hệ truyền thống. Hiểu rõ những khác biệt này và cách Prisma xử lý chúng là chìa khóa để xây dựng ứng dụng ổn định và hiệu năng cao.

Các mẹo dưới đây được đúc kết từ kinh nghiệm thực tế, giúp bạn tránh được những lỗi phổ biến và tận dụng tối đa sức mạnh của cả Prisma và MongoDB.

## 1. Cấu hình MongoDB với Replica Set

**Quan trọng:** MongoDB khi sử dụng với Prisma ORM *phải* được cấu hình thành một **replica set**. Đây là yêu cầu bắt buộc.

*   **Lý do:** Prisma sử dụng các transaction nội bộ để đảm bảo tính nhất quán của dữ liệu, đặc biệt với các truy vấn lồng nhau. MongoDB chỉ hỗ trợ transaction trong môi trường replica set.
*   **Vấn đề nếu không dùng:** Nếu bạn cố gắng sử dụng Prisma với MongoDB dạng standalone (độc lập), bạn sẽ gặp lỗi: `"Transactions are not supported by this deployment."` ngay cả với các thao tác đơn giản, vì Prisma bọc chúng trong transaction.
*   **Giải pháp:**
    *   Sử dụng **MongoDB Atlas**: Đây là cách dễ nhất vì Atlas hỗ trợ replica set mặc định (ngay cả gói miễn phí).
    *   **Phát triển local**: Chuyển MongoDB standalone sang replica set. Tham khảo hướng dẫn chi tiết trên trang chủ MongoDB.

## 2. Làm việc với các Trường ObjectId

Trong MongoDB, khóa chính (primary key) mặc định nằm trong trường `_id` và thường có kiểu dữ liệu `ObjectId`. Bạn cần ánh xạ chính xác trường này trong schema Prisma:

*   Trường ObjectId trong Prisma phải là kiểu **`String`** hoặc **`Bytes`**. Kiểu `String` phổ biến hơn.
*   Phải có thuộc tính **`@db.ObjectId`**.
*   Nên sử dụng **`@default(auto())`** để Prisma tự động sinh ID mới khi tạo bản ghi.

**Ví dụ mô hình `User` với `ObjectId`:**

```prisma
model User {
  id    String @id @default(auto()) @map("_id") @db.ObjectId
  email String
  name  String?
}
```

Khi cần tạo `ObjectId` thủ công (ví dụ: để test), bạn có thể sử dụng package `bson`:

```javascript
import { ObjectId from 'bson';
const id = new ObjectId();
```

## 3. Hiểu sự khác biệt giữa `null` và Trường không tồn tại

MongoDB phân biệt giữa các trường có giá trị `null` và các trường đơn giản là không tồn tại trong tài liệu. Đây là một phần của tính linh hoạt (polymorphism) của MongoDB.

*   **Vấn đề:** Nếu bạn tạo một bản ghi mà không cung cấp giá trị cho một trường tùy chọn, MongoDB sẽ không lưu trường đó. Tuy nhiên, khi bạn truy vấn qua Prisma, Prisma ORM sẽ trả về `null` cho trường đó, khiến bạn nhầm tưởng rằng trường này tồn tại và được gán giá trị `null`.
*   **Khi lọc dữ liệu:** Lọc chỉ với `fieldName: null` sẽ chỉ tìm thấy các bản ghi mà trường `fieldName` thực sự được set là `null`. Để bao gồm cả các bản ghi không có trường đó, bạn cần sử dụng toán tử **`isSet: false`**.

**Ví dụ:** Tìm các bản ghi `User` mà trường `name` hoặc là `null` hoặc không tồn tại:

```javascript
const users = await prisma.user.findMany({
  where: {
    OR: [
      { name: null },
      { name: { isSet: false } }
    ]
  }
});
```

## 4. Xử lý Quan hệ đúng cách

MongoDB xử lý quan hệ thông qua tham chiếu tài liệu (document references) hoặc nhúng tài liệu (embedded documents), khác với khái niệm khóa ngoại (foreign keys) trong cơ sở dữ liệu quan hệ. Khi introspect một cơ sở dữ liệu MongoDB hiện có, Prisma có thể không tự động nhận diện được các quan hệ này.

*   **Giải pháp:** Bạn cần tự định nghĩa các trường quan hệ và thuộc tính `@relation` trong schema Prisma.

**Ví dụ quan hệ một-nhiều giữa `User` và `Post`:**

```prisma
model Post {
  id     String @id @default(auto()) @map("_id") @db.ObjectId
  title  String
  // Trường userId lưu ID của User liên quan
  userId String @db.ObjectId // <-- Quan trọng: Cần @db.ObjectId cho ID tham chiếu

  // Định nghĩa quan hệ
  user   User   @relation(fields: [userId], references: [id])
}

model User {
  id    String @id @default(auto()) @map("_id") @db.ObjectId
  email String
  name  String?
  // Định nghĩa trường quan hệ ngược lại (một User có nhiều Post)
  posts Post[]
}
```

Hãy nhớ rằng các trường dùng để lưu trữ ID tham chiếu đến tài liệu khác luôn cần thuộc tính **`@db.ObjectId`**.

## 5. Mô hình hóa Tài liệu nhúng bằng `type`

MongoDB cho phép nhúng dữ liệu có cấu trúc (sub-documents) vào trong một tài liệu duy nhất – giúp giảm nhu cầu join dữ liệu ở phía ứng dụng. Prisma ORM hỗ trợ mô hình này bằng cách sử dụng từ khóa `type`.

*   **`type` trong Prisma:** Không tạo ra một collection riêng trong MongoDB. Nó chỉ định nghĩa cấu trúc của dữ liệu nhúng bên trong một tài liệu khác.

**Ví dụ định nghĩa kiểu `Address` và sử dụng trong mô hình `Customer`:**

```prisma
// Định nghĩa kiểu Address
type Address {
  street  String
  city    String
  state   String
  zipCode String
}

model Customer {
  id        String   @id @default(auto()) @map("_id") @db.ObjectId
  name      String
  email     String   @unique
  address   Address?    // <-- Tài liệu Address nhúng đơn lẻ (tùy chọn)
  addresses Address[] // <-- Mảng các tài liệu Address nhúng
}
```

**Sử dụng trong code:**

```javascript
// Tạo bản ghi Customer với tài liệu Address nhúng
const customer = await prisma.customer.create({
  data: {
    name: 'Jane Smith',
    email: 'jane@example.com',
    address: { // Dữ liệu nhúng
      street: '123 Main St',
      city: 'Anytown',
      state: 'CA',
      zipCode: '12345'
    }
  }
});

// Truy vấn dựa trên các trường trong tài liệu nhúng
const californiaCustomers = await prisma.customer.findMany({
  where: {
    address: { // Lọc theo trường trong tài liệu nhúng
      state: 'CA'
    }
  }
});
```

Tài liệu nhúng thích hợp khi:
*   Dữ liệu nhúng luôn được truy xuất cùng với tài liệu cha.
*   Có mối quan hệ sở hữu rõ ràng (dữ liệu nhúng chỉ thuộc về một tài liệu cha duy nhất).
*   Không cần truy vấn tài liệu nhúng một cách độc lập hoặc tạo quan hệ với các tài liệu khác.

## 6. Quản lý Schema đơn giản với MongoDB

MongoDB có schema linh hoạt, do đó không yêu cầu các bước migrate schema phức tạp như cơ sở dữ liệu quan hệ.

*   **Lệnh `prisma db push`:** Là công cụ chính để đồng bộ hóa schema Prisma với cơ sở dữ liệu MongoDB. Lệnh này sẽ:
    *   Tạo collection nếu chúng chưa tồn tại.
    *   Thiết lập các index cho các trường được đánh dấu `@unique`.
    *   Cập nhật Prisma Client để phản ánh những thay đổi trong schema.
*   Bạn có thể thay đổi mô hình dữ liệu trong file `schema.prisma` và chạy `prisma db push` mà thường không cần downtime (trừ khi có những thay đổi cấu trúc lớn cần xử lý thủ công).

## 7. Một số Lưu ý quan trọng khi Thiết kế

Khi thiết kế schema Prisma cho MongoDB, hãy ghi nhớ những điểm đặc thù sau:

*   MongoDB chỉ sử dụng một trường `_id` làm khóa chính. Bạn không thể dùng `@@id([])` với nhiều trường như trong cơ sở dữ liệu quan hệ.
*   Luôn sử dụng **`@default(auto())`** với trường `id` kiểu `String` và thuộc tính `@db.ObjectId` để Prisma tự động sinh `ObjectId`. Không sử dụng `autoincrement()`.
*   Với các quan hệ có vòng lặp hoặc phức tạp, cân nhắc sử dụng **`onDelete: NoAction`** trên thuộc tính `@relation` để tránh các lỗi hoặc hành vi không mong muốn liên quan đến việc xóa dữ liệu.
*   Kiểu dữ liệu **`Decimal128`** trong MongoDB hiện chỉ được Prisma hỗ trợ một phần. Cần kiểm tra tài liệu chính thức của Prisma để biết mức độ hỗ trợ hiện tại.
*   Nắm vững sự khác biệt cơ bản giữa cơ sở dữ liệu dạng tài liệu (MongoDB) và cơ sở dữ liệu quan hệ (như PostgreSQL, MySQL) là nền tảng để thiết kế mô hình dữ liệu hiệu quả.

## 8. Tối ưu hóa Hiệu năng với Collections lớn

Hiệu suất khi truy vấn MongoDB thông qua Prisma ORM có thể suy giảm khi làm việc với các collection có kích thước lớn nếu không được tối ưu hóa.

*   **Thêm Index:** Sử dụng thuộc tính **`@index`** trên các trường mà bạn thường xuyên sử dụng trong các mệnh đề `where` hoặc `orderBy` để tăng tốc độ truy vấn.
*   **Sử dụng `select`:** Chỉ lấy về các trường dữ liệu thực sự cần thiết bằng cách sử dụng tùy chọn `select` trong các truy vấn `findMany`, `findFirst`, `findUnique`.
*   **Phân trang:** Sử dụng các tùy chọn **`skip`** và **`take`** để thực hiện phân trang, tránh tải về toàn bộ dữ liệu của một collection lớn cùng lúc.
*   **Sử dụng `$runCommandRaw` hoặc `$runCommandRawUnsafe`:** Đối với các truy vấn phức tạp yêu cầu sử dụng MongoDB aggregation pipeline hoặc các lệnh MongoDB đặc thù mà Prisma Client không hỗ trợ trực tiếp, bạn có thể dùng các phương thức raw command này.

**Ví dụ tối ưu hóa truy vấn:**

```javascript
const users = await prisma.user.findMany({
  where: { role: "ADMIN" }, // Giả sử có index trên trường role
  select: { id: true, email: true }, // Chỉ lấy id và email
  take: 100, // Giới hạn 100 kết quả
  skip: page * 100, // Phân trang (ví dụ: lấy trang thứ 'page')
});
```

## Kết luận

Sử dụng Prisma ORM với MongoDB mang lại nhiều lợi ích về hiệu suất làm việc cho nhà phát triển, nhưng đòi hỏi sự hiểu biết về những khác biệt cơ bản của MongoDB so với cơ sở dữ liệu quan hệ. Nắm vững 8 mẹo hàng đầu này sẽ giúp bạn xây dựng ứng dụng mạnh mẽ, hiệu quả và tránh được các lỗi phổ biến.

Hãy nhớ rằng, chìa khóa là luôn dùng replica set, xử lý `ObjectId` và sự khác biệt `null`/thiếu trường chính xác, định nghĩa quan hệ và tài liệu nhúng đúng cách, tận dụng `prisma db push`, và tối ưu hóa truy vấn khi cần thiết.

*(Để hiểu thêm, bạn có thể tham khảo các hướng dẫn chi tiết hơn về xây dựng ứng dụng full-stack với Next.js + Prisma + MongoDB.)*
