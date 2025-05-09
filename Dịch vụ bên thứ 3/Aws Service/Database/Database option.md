# 1 Hướng dẫn Chọn Dịch vụ Cơ sở dữ liệu trên AWS: RDS, DynamoDB, và Aurora

Các dịch vụ cơ sở dữ liệu đóng vai trò cốt lõi trong hầu hết các ứng dụng và là kiến thức quan trọng cho kỳ thi AWS CCP (Certified Cloud Practitioner). Tài liệu này phân tích sâu về Amazon RDS, Amazon DynamoDB và Amazon Aurora để giúp bạn lựa chọn công cụ lưu trữ dữ liệu phù hợp nhất.

![alt text](image.png)

## 1. Amazon RDS - Relational Database Service

*   **Đặc điểm chính:**
    *   Dịch vụ cơ sở dữ liệu quan hệ (SQL) được quản lý hoàn toàn (Managed).
    *   Hỗ trợ nhiều công cụ database phổ biến: MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, và cả Amazon Aurora.
    *   AWS chịu trách nhiệm các tác vụ quản trị như: cập nhật hệ điều hành (patch OS), sao lưu tự động (automatic backups), và khả năng chuyển đổi dự phòng Multi-AZ (Multi-AZ failover).

*   **Trường hợp sử dụng điển hình:**
    *   Các ứng dụng yêu cầu xử lý giao dịch (transactional) cao.
    *   Các ứng dụng với schema quan hệ chặt chẽ (relational schema), ví dụ: hệ thống thanh toán (billing system), hệ thống quản lý quan hệ khách hàng (CRM).

*   **Khả năng Mở rộng (Scaling) & Tính sẵn sàng cao (High Availability):**
    *   **Vertical Scaling:** Tăng kích thước instance (ví dụ: từ `db.t3` lên `db.m5`) để có thêm CPU, RAM. Yêu cầu downtime ngắn.
    *   **Read Replicas:** Tạo bản sao chỉ đọc để phân tải lưu lượng truy vấn đọc (offload read traffic). Hỗ trợ sao chép liên vùng (cross-Region).
    *   **Multi-AZ Deployment:** Tự động tạo bản sao dự phòng (standby copy) ở Availability Zone khác và tự động chuyển đổi (switchover) khi primary instance gặp sự cố, đảm bảo tính sẵn sàng cao.
    *   **Automated Backups & Snapshots:** Sao lưu tự động và tạo snapshots. Có thể lưu trữ tối đa 35 ngày. Hỗ trợ khôi phục tại một thời điểm cụ thể (point-in-time restore).

*   **Mô hình định giá:**
    *   **On-Demand Instance:** Thanh toán theo giờ sử dụng instance.
    *   **Reserved Instances:** Giảm giá 30–60% nếu cam kết sử dụng trong 1–3 năm.
    *   **Storage:** Tính phí theo dung lượng (USD/GB mỗi tháng) cộng với số lượng yêu cầu I/O (áp dụng cho Provisioned IOPS).
    *   **Data Transfer:** Miễn phí khi truyền dữ liệu trong cùng AZ (intra-AZ), tính phí khi truyền giữa các AZ (cross-AZ) và ra Internet.

## 2. Amazon DynamoDB - NoSQL Key‑Value & Document

*   **Đặc điểm chính:**
    *   Dịch vụ NoSQL (Key-Value & Document) không máy chủ (Serverless), được quản lý hoàn toàn (fully managed).
    *   Tự động mở rộng thông lượng (scale throughput) và dung lượng lưu trữ (storage).

*   **Mô hình dữ liệu:**
    *   Dữ liệu được tổ chức thành Bảng (Table), Mục (Item) và Thuộc tính (Attribute).
    *   Sử dụng Khóa chính (Primary Key) bao gồm Khóa phân vùng (Partition Key) và tùy chọn Khóa sắp xếp (Sort Key).

*   **Trường hợp sử dụng điển hình:**
    *   Lưu trữ thông tin phiên người dùng (session store).
    *   Giỏ hàng mua sắm (shopping cart).
    *   Bảng xếp hạng thời gian thực (real-time leaderboards).
    *   Các ứng dụng yêu cầu độ trễ thấp và khả năng mở rộng cực lớn.

*   **Khả năng (Capacity) & Tính năng:**
    *   **Chế độ Provisioned (Provisioned Mode):** Định trước số lượng Read Capacity Units (RCU) và Write Capacity Units (WCU). Cần dự báo trước nhu cầu.
    *   **Chế độ On-Demand (On-Demand Mode):** Tự động mở rộng quy mô, thanh toán theo số lượng yêu cầu đọc/ghi thực tế. Không cần dự báo trước.
    *   **Global Tables:** Cung cấp khả năng sao chép dữ liệu đa vùng (multi-Region replication) tự động.
    *   **Transactions & Streams:** Hỗ trợ các giao dịch ACID (trong một bảng hoặc giữa các bảng) và cung cấp luồng dữ liệu thay đổi (change data capture) với DynamoDB Streams.

*   **Mô hình định giá:**
    *   **Chế độ Provisioned:** Tính phí dựa trên số lượng RCU/WCU đã định trước (ví dụ: $0.00013 mỗi RCU/WCU mỗi tháng) cộng với phí lưu trữ ($0.25/GB mỗi tháng).
    *   **Chế độ On-Demand:** Tính phí dựa trên số lượng yêu cầu đọc/ghi thực tế (ví dụ: $1.25 cho mỗi triệu yêu cầu ghi + $0.25 cho mỗi triệu yêu cầu đọc).
    *   **Data Transfer & Streams:** Tính thêm phí dựa trên dung lượng truyền (GB mỗi tháng) và số lượng yêu cầu đối với Streams.

## 3. Amazon Aurora - High‑Performance Relational

*   **Đặc điểm chính:**
    *   Cơ sở dữ liệu quan hệ hiệu suất cao, tương thích với MySQL và PostgreSQL nhưng được AWS tối ưu riêng.
    *   Hiệu suất vượt trội: Nhanh hơn gấp 5 lần so với MySQL chuẩn và gấp 3 lần so với PostgreSQL chuẩn nhờ kiến trúc lưu trữ phân tán.
    *   Hỗ trợ Multi-Master (ghi đồng thời vào nhiều instance) và Serverless.

*   **Tính năng nâng cao:**
    *   **Aurora Serverless v2:** Tự động điều chỉnh năng lực tính toán (compute) theo nhu cầu, mở rộng linh hoạt từ mức tối thiểu đến tối đa chỉ trong mili giây.
    *   **Aurora Global Database:** Cung cấp khả năng sao chép dữ liệu đa vùng (cross-Region) với độ trễ thấp cho các ứng dụng toàn cầu.

*   **Khả năng Mở rộng (Scaling) & Định giá:**
    *   **Chế độ Provisioned:** Chọn lớp instance cụ thể. Có thể tạo tối đa 4 bản sao chỉ đọc (read replicas) trong cùng Region và tối đa 16 bản sao chỉ đọc cross-Region (với Aurora Global Database).
    *   **Chế độ Serverless v2:** Thanh toán theo Aurora Capacity Units (ACU) tiêu thụ. Khả năng tự động mở rộng nhanh chóng và chi tiết hơn Serverless v1.
    *   **Storage:** Tính phí theo dung lượng ($0.10/GB mỗi tháng) cộng với phí I/O dựa trên khối lượng công việc (workload).

## 4. So sánh và Hướng dẫn Lựa chọn

*   **Amazon RDS:** Lựa chọn phù hợp cho các cơ sở dữ liệu quan hệ thông thường, các ứng dụng giao dịch với khối lượng công việc có thể dự đoán được (predictable workload).
*   **Amazon DynamoDB:** Lý tưởng cho các ứng dụng cần khả năng mở rộng linh hoạt, mô hình dữ liệu không schema cố định (no-schema) hoặc schema linh hoạt, và yêu cầu độ trễ rất thấp. Thích hợp cho các workload có lưu lượng truy cập thay đổi đáng kể hoặc không thể dự báo.
*   **Amazon Aurora:** Dành cho các ứng dụng quan hệ yêu cầu hiệu suất cực cao và các tính năng nâng cao như Multi-Master, Serverless tự động mở rộng nhanh chóng hoặc cơ sở dữ liệu toàn cầu với độ trễ thấp. Thích hợp cho các doanh nghiệp lớn (enterprise) hoặc các ứng dụng quan trọng (mission-critical).

Việc lựa chọn dịch vụ database phù hợp trên AWS cần dựa trên việc đánh giá kỹ lưỡng tính chất dữ liệu, nhu cầu mở rộng (scale), yêu cầu về độ trễ (latency) và ngân sách chi phí của ứng dụng.

Tuyệt vời! Dưới đây là nội dung bài viết về truy vấn DynamoDB dựa trên Partition Key và Sort Key, được viết lại dưới dạng file `README.md` bằng tiếng Việt, giữ nguyên cấu trúc và ý chính.

# 5 Truy vấn trong DynamoDB dựa trên Partition Key và Sort Key

*Bài viết được dịch và biên soạn lại từ bài viết của Alexandru Borza trên freestar (Cập nhật cuối: 09/05/2025)*

## 1. Giới thiệu

Amazon DynamoDB là một trong những dịch vụ cốt lõi của AWS, được sử dụng rộng rãi để xây dựng các ứng dụng nhanh, có khả năng mở rộng (scalable) và serverless. Đây là giải pháp cơ sở dữ liệu NoSQL được quản lý toàn diện (fully managed), mang lại hiệu suất phản hồi chỉ vài mili giây ở mọi quy mô. Khác với các cơ sở dữ liệu quan hệ truyền thống, DynamoDB sử dụng mô hình dữ liệu key-value và document, khuyến khích việc thiết kế trước các mô hình truy cập dữ liệu (access patterns).

Trong bài viết này, chúng ta sẽ tập trung vào một trong những tính năng mạnh mẽ nhất của DynamoDB: truy vấn dữ liệu bằng cách sử dụng khóa chính tổng hợp (composite primary key), kết hợp giữa **Partition Key** và **Sort Key**. Chúng ta sẽ cùng tìm hiểu cách mô hình khóa tổng hợp này hoạt động và minh họa cách truy vấn dữ liệu hiệu quả bằng cách sử dụng AWS SDK for Java v2.

## 2. Hiểu về Mô hình Composite Key

DynamoDB hỗ trợ hai loại khóa chính (primary keys):
1.  **Simple Key:** Chỉ sử dụng **Partition Key**.
2.  **Composite Key:** Kết hợp **Partition Key** và **Sort Key**.

*   **Partition Key (Hash Key):** Xác định phân vùng (partition) vật lý nơi dữ liệu được lưu trữ. Nó được sử dụng để phân phối dữ liệu trên các máy chủ lưu trữ của DynamoDB.
*   **Sort Key (Range Key):** Cho phép sắp xếp (sorting) và lọc (filtering) các mục (items) **bên trong một phân vùng cụ thể** được xác định bởi Partition Key.

Mô hình khóa tổng hợp này rất phù hợp để tổ chức các dữ liệu liên quan dưới cùng một Partition Key. Ví dụ, trong bảng `UserOrders`:

*   `userId` có thể là **Partition Key** (Hash Key).
*   `orderDate` có thể là **Sort Key** (Range Key).

Thiết lập này giúp việc truy xuất tất cả các đơn hàng của một người dùng trở nên dễ dàng, sắp xếp chúng theo ngày, hoặc lọc theo khoảng thời gian, tất cả chỉ với một truy vấn duy nhất đến một phân vùng cụ thể.

## 3. Dependency Maven

Để tương tác với DynamoDB từ ứng dụng Java, chúng ta sẽ sử dụng AWS SDK for Java v2, cung cấp API hiện đại và non-blocking:

Thêm dependency sau vào file `pom.xml` của bạn:

```xml
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>dynamodb</artifactId>
    <version>2.31.26</version>
</dependency>
```

Dependency này cung cấp cho chúng ta quyền truy cập vào `DynamoDbClient` và các lớp cần thiết khác để thực hiện truy vấn.

## 4. Truy vấn theo Partition Key

Cách đơn giản và phổ biến nhất để truy vấn dữ liệu trong DynamoDB là sử dụng Partition Key. Khi truy vấn theo Partition Key, DynamoDB sẽ trả về tất cả các mục (items) có cùng giá trị Partition Key.

Giả sử bảng `UserOrders` của chúng ta lưu trữ đơn hàng cho nhiều người dùng, và chúng ta muốn lấy tất cả đơn hàng của một người dùng cụ thể (ví dụ: `user1`). Vì `userId` là Partition Key, chúng ta có thể thực hiện điều này bằng một truy vấn cơ bản:

```java
QueryRequest queryRequest = QueryRequest.builder()
  .tableName("UserOrders")
  .keyConditionExpression("userId = :uid") // Điều kiện trên Partition Key
  .expressionAttributeValues(Map.of(
    ":uid", AttributeValue.builder().s("user1").build() // Gán giá trị cho placeholder :uid
  )).build();

QueryResponse response = dynamoDbClient.query(queryRequest);
```

Trong ví dụ trên, chúng ta sử dụng `DynamoDbClient` để thực hiện truy vấn. Chúng ta xây dựng request bằng mẫu builder, giúp việc tạo các request phức tạp trở nên dễ dàng. Sau khi thực hiện truy vấn, kết quả được trả về trong đối tượng `QueryResponse`. Để truy cập dữ liệu thực tế, chúng ta sử dụng phương thức `items()`:

```java
List<Map<String, AttributeValue>> items = response.items();

for (Map<String, AttributeValue> item : items) {
    // Xử lý từng item
    System.out.println("Order item: " + item.get("item").s());
}
```

Danh sách `items` này chứa mỗi mục dưới dạng một Map ánh xạ tên thuộc tính (attribute name) với giá trị thuộc tính (`AttributeValue`). Bạn có thể xử lý thêm hoặc chuyển đổi chúng thành các đối tượng ứng dụng cụ thể.

## 5. Truy vấn kết hợp Partition Key và Sort Key

Mặc dù truy vấn chỉ bằng Partition Key có thể hữu ích, đôi khi chúng ta cần lọc chính xác hơn. Chúng ta có thể đạt được điều này bằng cách kết hợp Partition Key với các điều kiện trên Sort Key. Điều này cho phép chúng ta lọc kết quả bên trong một phân vùng, ví dụ: theo khoảng ngày hoặc theo một tiền tố cụ thể.

Giả sử chúng ta muốn lấy tất cả các đơn hàng được đặt bởi `user1` sau ngày 1 tháng 1 năm 2025. Vì `orderDate` là Sort Key của chúng ta, chúng ta có thể thêm một điều kiện so sánh vào `keyConditionExpression`:

```java
QueryRequest queryRequest = QueryRequest.builder()
  .tableName("UserOrders")
  .keyConditionExpression("userId = :uid AND orderDate > :startDate") // Điều kiện trên cả PK và SK
  .expressionAttributeValues(Map.of(
    ":uid", AttributeValue.builder().s("user1").build(),
    ":startDate", AttributeValue.builder().s("2025-01-01").build() // Gán giá trị cho placeholder :startDate
  )).build();

QueryResponse response = dynamoDbClient.query(queryRequest);
```

Trong truy vấn này, chúng ta sử dụng Partition Key và một điều kiện trên Sort Key để thu hẹp kết quả. DynamoDB sẽ chỉ quét phân vùng cho `user1` và trả về các mục có `orderDate` sau ngày 2025-01-01. Cách tiếp cận này rất hiệu quả vì nó tránh việc quét toàn bộ dữ liệu không liên quan.

**Lưu ý quan trọng:** Tất cả các điều kiện trong `keyConditionExpression` phải áp dụng cho Khóa Chính (Partition Key VÀ/HOẶC Sort Key). Các điều kiện lọc trên các thuộc tính khác phải được thêm vào bằng `filterExpression`. Tuy nhiên, `filterExpression` được áp dụng *sau* khi dữ liệu đã được lấy ra từ phân vùng, do đó ít hiệu quả hơn so với việc lọc bằng Sort Key trong `keyConditionExpression`.

## 6. Các Điều kiện Phổ biến cho Sort Key (Range Key Conditions)

DynamoDB hỗ trợ một số toán tử hữu ích để lọc theo Sort Key. Chúng cho phép chúng ta tinh chỉnh các truy vấn bên trong một phân vùng bằng cách sử dụng logic so sánh tiêu chuẩn.

### 6.1. BETWEEN

Chúng ta có thể sử dụng `BETWEEN` để truy xuất các mục nằm trong một khoảng giá trị cụ thể. Điều này đặc biệt hữu ích khi làm việc với dấu thời gian (timestamps) hoặc ngày tháng:

```java
QueryRequest queryRequest = QueryRequest.builder()
  .tableName("UserOrders")
  .keyConditionExpression("userId = :uid AND orderDate BETWEEN :from AND :to") // Điều kiện BETWEEN trên SK
  .expressionAttributeValues(Map.of(
    ":uid", AttributeValue.builder().s("user1").build(),
    ":from", AttributeValue.builder().s("2024-12-01").build(),
    ":to", AttributeValue.builder().s("2024-12-31").build()
  )).build();
```

Truy vấn này sẽ trả về tất cả các đơn hàng được đặt bởi `user1` trong tháng 12 năm 2024.

### 6.2. BEGINS_WITH

Nếu Sort Key của chúng ta là một chuỗi (ví dụ: một ngày được định dạng `YYYY-MM-DD`), chúng ta có thể truy vấn tất cả các mục bắt đầu bằng một tiền tố cụ thể. Điều này hữu ích để nhóm theo năm, tháng, hoặc bất kỳ tiền tố dựa trên chuỗi nào.

```java
QueryRequest queryRequest = QueryRequest.builder()
  .tableName("UserOrders")
  .keyConditionExpression("userId = :uid AND begins_with(orderDate, :prefix)") // Điều kiện BEGINS_WITH trên SK
  .expressionAttributeValues(Map.of(
    ":uid", AttributeValue.builder().s("user1").build(),
    ":prefix", AttributeValue.builder().s("2025-01").build() // Tiền tố "2025-01" cho tháng 1/2025
  )).build();
```

Truy vấn này sẽ trả về tất cả các đơn hàng được đặt trong tháng 1 năm 2025 (của `user1`).

## 7. Xử lý phân trang (Pagination) trong Truy vấn

DynamoDB giới hạn kích thước của mỗi phản hồi truy vấn là 1 MB dữ liệu. Nếu truy vấn của chúng ta khớp với nhiều hơn 1 MB, DynamoDB sẽ trả về một `LastEvaluatedKey` trong phản hồi, mà chúng ta có thể sử dụng để tiếp tục tìm nạp trang kết quả tiếp theo.

Để xử lý điều này, chúng ta nên lặp qua các trang kết quả:

```java
List<Map<String, AttributeValue>> allItems = new ArrayList<>();
Map<String, AttributeValue> lastKey = null;
String userId = "user1"; // Ví dụ userId

do {
    QueryRequest.Builder requestBuilder = QueryRequest.builder()
      .tableName("UserOrders")
      .keyConditionExpression("userId = :uid")
      .expressionAttributeValues(Map.of(
        ":uid", AttributeValue.fromS(userId)
      ));

    // Nếu có lastKey từ lần truy vấn trước, thêm vào request để bắt đầu từ đó
    if (lastKey != null && !lastKey.isEmpty()) {
        requestBuilder.exclusiveStartKey(lastKey);
    }

    QueryResponse response = dynamoDbClient.query(requestBuilder.build());
    allItems.addAll(response.items()); // Thêm items của trang hiện tại vào danh sách tổng
    lastKey = response.lastEvaluatedKey(); // Lấy lastKey cho trang tiếp theo
} while (lastKey != null && !lastKey.isEmpty()); // Lặp cho đến khi không còn lastKey

return allItems; // Trả về tất cả items đã lấy từ mọi trang
```

Mẫu này đảm bảo chúng ta truy xuất tất cả các mục khớp, bất kể có bao nhiêu mục được trả về trên mỗi trang. Nó hữu ích khi truy vấn các phân vùng lớn hoặc thực hiện các báo cáo/xuất dữ liệu.

## 8. Kết luận

Mô hình khóa tổng hợp của DynamoDB mang đến một cách mạnh mẽ để tổ chức và truy xuất dữ liệu một cách hiệu quả. Việc sử dụng Partition Key cùng với Sort Key cho phép chúng ta tổ chức dữ liệu liên quan và tạo ra các mô hình truy cập hiệu quả, có khả năng mở rộng cho các ứng dụng đòi hỏi hiệu suất cao.

Trong bài viết này, chúng ta đã khám phá cách truy vấn bảng chỉ sử dụng Partition Key và cách cải thiện các truy vấn đó bằng cách thêm Sort Key để lọc kết quả bên trong một phân vùng.

Như thường lệ, mã nguồn ví dụ có sẵn trên GitHub.

---

*Bài viết gốc:* [Link đến bài viết gốc](https://www.freestar.com/articles/query-in-dynamodb-on-the-basis-of-partition-key-and-sort-key/)
*Mã nguồn ví dụ:* [Link đến mã nguồn ví dụ trên GitHub] (Nếu có, hãy thay thế bằng link thực tế từ bài gốc)
```