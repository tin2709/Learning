
# 1 DynamoDB vs. MySQL & Single Table Design: Hướng dẫn cho Pet Shop

## Phần 1 - DynamoDB là gì và tại sao nó khác MySQL?

Tưởng tượng bạn đang xây một app bán thú cưng online, kiểu như “Pet Shop” made in Vietnam. Đột nhiên, một influencer livestream quảng bá, cả triệu người ùa vào xem mèo con, server MySQL của bạn bắt đầu “khóc thét”, truy vấn chậm như rùa bò. Lúc này, AWS thì thầm: “Thử DynamoDB đi, nó sinh ra để xử lý drama này đó!”. Vậy DynamoDB là gì, và nó khác MySQL ở chỗ nào? Hãy cùng tìm hiểu xem nhé!

### Kèo căng DynamoDB vs MySQL

DynamoDB là cơ sở dữ liệu NoSQL của AWS, được thiết kế serverless (không cần lo server) và mở rộng vô hạn (AWS nói vậy chứ mình cũng k biết thật ko =)) ). Nó cho phép lưu mọi loại dữ liệu mà không cần bạn định nghĩa trước cấu trúc. Thêm nữa là bạn chỉ trả tiền cho những gì dùng (lưu nhiêu trả nhiêu, read/write nhiêu trả nhiêu, không có thì khỏi trả).

Để ví dụ đơn giản, dễ hiểu nhất thì DynamoDB nó giống như một thằng bạn "chill guy", nói gì với khứa đó cũng được, sẽ ko bị cấm, bị phàn nàn (tào lao quá thì nó đấm thôi =)) ).

MySQL thì ngược lại, là "cô giáo" nghiêm khắc của thế giới RDBMS. Nó yêu cầu bảng, cột, schema rõ ràng, không thì “trừ điểm”. Để hiểu rõ hơn, hãy so sánh chúng qua các khía cạnh cụ thể, với ví dụ minh họa từ app Pet Shop của chúng ta.

#### 1. Data Model: Schema cố định hay tự do bay nhảy?

*   **MySQL:** Yêu cầu bạn định nghĩa schema trước, như tạo bảng `Pets` với các cột cố định: `id`, `name`, `type`, `price`. Nếu bạn đột nhiên muốn thêm `color` cho mèo, phải sửa schema, chạy migration.
    *   **Ví dụ:** Bạn lưu một chú mèo như `{id: 1, name: "Bun", type: "cat", price: 500000}`. Muốn thêm `color: "orange"`? --> Cần `ALTER TABLE` để add nguyên cột mới.

*   **DynamoDB:** " Schema là cái gì?! " --> Ko cần define schema gì cả. Mỗi item trong bảng giống như một JSON, bạn muốn thêm gì thì thêm. Trong bảng `Pets`, bạn lưu chú mèo như `{id: 1, name: "Bun", type: "cat", price: 500000}`. Muốn thêm `color`? Cứ thêm `{id: 2, name: "Tom", type: "cat", price: 600000, color: "orange"}`, bảng vẫn vui vẻ chấp nhận.
    *   **Ví dụ:** Nếu PetShop muốn lưu thêm thông tin “chứng nhận tiêm phòng” cho một số thú cưng, DynamoDB cho phép thêm thuộc tính này chỉ cho những item cần, không ảnh hưởng item khác.

#### 2. Scalability: Ai là “siêu nhân” chịu tải?

*   **MySQL:** Muốn mở rộng? Thường phải nâng cấp server (scale vertically) hoặc chia dữ liệu ra nhiều server (sharding). Với PetShop, nếu triệu người vào xem mèo cùng lúc, bạn phải thêm RAM, CPU, hoặc chia bảng `Pets` ra nhiều server. Sharding còn đòi hỏi bạn tự quản lý logic, dễ sai sót. Mình cũng chỉ mới nghe về Sharding ra nhiều server chứ chưa được động vào bao giờ nhưng chắc chắn là rất phức tạp.

*   **DynamoDB:** Sinh ra để scale tự động! AWS tự chia dữ liệu thành các partition để lưu trữ. Traffic tự nhiêu nhiều quá hả -> bạn chỉ cần hét lên “tăng nhiệt” là xong (tăng RCU/WCU – Read/Write Capacity Units). Hoặc thậm chí là Auto Scale hoàn toàn luôn với On-demand.
    *   **Ví dụ:** Trong livestream PetShop, DynamoDB tự điều chỉnh để xử lý triệu request mà không cần bạn thức đêm. Bạn có thể chọn on-demand capacity (auto hoàn toàn) hoặc provisioned capacity (tiết kiệm hơn, nhưng cần set up để auto scale), tiện như gọi Grab.

#### 3. Query: JOIN vs Speed

*   **MySQL:** SQL là bố! Bạn có thể `JOIN`, `GROUP BY`, `ORDER BY` thoải mái. Với PetShop, muốn tìm tất cả đơn hàng của khách hàng “Nhi”? Dùng query: `SELECT * FROM Orders JOIN Users ON Orders.user_id = Users.id WHERE Users.name = 'Nhi'`. Mạnh mẽ, nhưng nếu bảng lớn, JOIN có thể chậm như rùa luôn.

*   **DynamoDB:** Ếu có JOIN luôn =)) , chỉ tập trung vào truy vấn nhanh dựa trên khóa chính (partition key, sort key) hoặc Index (GSI, LSI). Để lấy đơn hàng của “Nhi”, bạn phải denormalize, tức là lưu thông tin user trực tiếp trong item đơn hàng, như `{order_id: 1, user_name: "Nhi", pet_id: 1}`. Truy vấn sẽ nhanh như chớp, nhưng cần tư duy thiết kế khác.
    *   Trong ví dụ trên, nếu `order_id` là PK, thì chỉ cần đánh GSI cho `user_name` và query `user_name = "Nhi"` trên GSI đó ta sẽ lấy được đơn hàng của Nhi với tốc độ O(1), ko join jiếc gì hết he he.

#### 4. Consistency: Ai đáng tin hơn?

*   **MySQL:** Đảm bảo strong consistency với ACID transactions. Khi bạn thêm một đơn hàng vào PetShop, dữ liệu được cập nhật ngay, không sợ sai lệch. Rất hợp cho hệ thống thanh toán, ví dụ: đảm bảo tiền trừ đúng khi khách mua mèo.

*   **DynamoDB:** Mặc định là eventual consistency, nghĩa là có thể mất vài giây (thường dưới 1s) để dữ liệu đồng bộ giữa các bản sao, giúp tăng tốc độ. Nhưng bạn có thể chọn strong consistency nếu cần.
    *   Với Pet Shop, nếu khách xem số lượng mèo còn lại, eventual consistency đủ dùng vì độ trễ nhỏ không ảnh hưởng. Nhưng khi đặt hàng, bạn phải bật strong consistency để đảm bảo không bán trùng mèo nhé @@.

#### 5. Chi phí và quản lý: Ai dễ thở hơn?

*   **MySQL:** Nếu tự host, bạn lo backup, scaling, vá lỗi – như chăm cún vậy. Dùng AWS RDS thì nhẹ hơn, nhưng vẫn trả tiền cho instance, kể cả khi không dùng.
    *   Với Pet Shop, nếu khách ít, bạn vẫn tốn tiền duy trì server. Đây là một lý do mình quyết định dùng DynamoDb cho Givables vì mình có user nào quái đâu =)))

*   **DynamoDB:** Fully managed, AWS lo hết từ A-Z. Bạn chỉ focus vào code và thiết kế dữ liệu. Chi phí dựa trên RCU/WCU và storage, rất linh hoạt.
    *   Pet Shop chỉ đông khách vào cuối tuần? Dùng on-demand capacity, bạn chỉ trả tiền khi có người vào xem để mua mèo.

### Vậy chọn cái nào bây giờ?

*   **DynamoDB:** Lý tưởng nếu Pet Shop cần xử lý triệu lượt xem, muốn serverless (không quản lý server, hoặc xài nhiêu trả nhiêu), hoặc dữ liệu linh hoạt (như thêm thuộc tính “chứng nhận tiêm phòng” bất kỳ lúc nào). Hiệu suất cao, ổn định, latency thấp (single-digit milliseconds), hợp cho app chat, e-commerce, IoT.

*   **MySQL:** Latency thấp hơn cả Dynamo nếu db không quá lớn, phù hợp cho app cần báo cáo phức tạp, như phân tích doanh thu theo loại thú cưng, hoặc dữ liệu có quan hệ rõ ràng (bảng Users, Orders, Pets). Cũng hợp nếu team bạn yêu SQL và ngại học NoSQL.

### Kết luận: DynamoDB có đáng để “đổi gió”?

DynamoDB không phải Silver bullet, nếu bạn cần tốc độ, scale tự động, và không muốn đau đầu quản lý server, hoặc chỉ đơn giản là ko muốn mất tiền duy trì server khi ít user (giống mình) thì nó là lựa chọn sáng giá. MySQL thì đã quá quen thuộc, đáng tin, cho những thứ cần cấu trúc rõ ràng. Hiểu rõ nhu cầu của bạn – như Pet Shop, muốn bán mèo nhanh hay báo cáo chi tiết – sẽ giúp bạn chọn đúng.

Phần sau, chúng ta sẽ cùng tìm hiểu Single Table Design, bí kíp biến DynamoDB thành “võ công thượng thừa” nhá.

---

## Phần 2 - Single Table Design - Kỹ thuật tối ưu thiết kế DynamoDB
`#MayFest2025`

Chúng ta đã biết ưu điểm của DynamoDB qua bài viết trước, tuy nhiên nếu kết hợp sử dụng Single Table Design (STD) nó như "hổ mọc thêm cánh" vậy. Đại khái là: “Dùng một bảng DynamoDB để lưu cả thế giới” yes he he, đó là STD! Thay vì chia dữ liệu thành chục bảng như MySQL, STD nhét mọi thứ vào một bảng duy nhất, vừa tiết kiệm, vừa siêu nhanh. Nghe điên vãi =))) Đúng, nhưng điên kiểu thiên tài! Cùng “mổ xẻ” để xem bí mật này là gì nhé.

### Single Table Design là gì?

Trong MySQL, bạn quen chia dữ liệu thành nhiều bảng: `Users`, `Orders`, `Pets`, mỗi bảng một nhiệm vụ. STD với Dynamo thì bảo: “Nhiều bảng làm gì cho mệt? Một bảng là đủ!”. STD là cách thiết kế để tất cả dữ liệu liên quan – khách hàng, đơn hàng, thú cưng – sống chung trong một bảng, được truy vấn siêu tốc nhờ Primary Key (partition key, sort key) và Index (GSI, LSI).

### Tại sao nên sử dụng STD?

#### 1. Tối ưu hiệu suất:
DynamoDB truy vấn nhanh nhất khi dùng khóa chính. STD đảm bảo mọi thứ bạn cần đều nằm trong một bảng, giảm số lần gọi API.
*   Với app PetShop, bạn muốn lấy thông tin khách hàng và tất cả đơn hàng của họ. Nếu dùng nhiều bảng, bạn phải query riêng bảng `Users`, rồi bảng `Orders` – tốn 2 API calls, chậm hơn chút. Với STD, một Query duy nhất vào bảng `PetShop` lấy hết cả hai.
*   Gọi bánh tráng + trà sữa 2 nơi thì bạn tốn 2 lần phí ship. Gọi cùng 1 chổ thì chỉ tốn 1 lần thôi =))

#### 2. Tiết kiệm chi phí:
Một bảng tốn ít storage và RCU/WCU hơn nhiều bảng. Ai mà không thích tiết kiệm tiền?
*   Trong PetShop, nếu dùng ba bảng (`Users`, `Orders`, `Pets`), bạn phải trả tiền cho storage và throughput của cả ba. Với STD, chỉ một bảng `PetShop`, nhìn chung thì centralize lại sẽ ít tiết kiệm hơn.

#### 3. Linh hoạt:
STD xử lý ngon lành dữ liệu phức tạp, kể cả khi app phát triển thêm tính năng.
*   PetShop ban đầu chỉ bán mèo, giờ muốn thêm dịch vụ “thuê thú cưng”? Với STD, bạn chỉ cần thêm item mới như `{PK: "USER#U001", SK: "RENTAL#R001", pet_id: "P001"}` vào bảng `PetShop`, không cần tạo bảng mới. Nếu dùng MySQL, bạn phải tạo bảng `Rentals`, lại đau đầu migration.
*   Với PetShop, STD giúp bạn lưu mọi thứ – từ thông tin khách, đơn hàng, đến thú cưng – trong bảng `PetShop`. Muốn lấy dữ liệu? Chỉ cần một truy vấn, nhanh gọn lẹ! Nhưng để đạt được mức 1 truy vấn lấy đầy đủ data cũng không phải đơn giản nhé.

### Làm sao để thiết kế một bảng “bao sân”?

STD không phải kiểu nhét bừa mọi thứ vào một “sọt rác” dữ liệu. Bạn cần thiết kế dựa trên mô hình truy vấn (Access pattern) – tức là nghĩ trước xem người dùng sẽ hỏi gì và tối ưu bảng để trả lời nhanh nhất. Các bước cơ bản:

1.  **Xác định truy vấn chính** (ví dụ: lấy đơn hàng của khách, lấy thú cưng theo loại).
2.  **Chọn partition key và sort key** để hỗ trợ các truy vấn đó.
3.  **Dùng chỉ mục (GSI)** để xử lý các truy vấn phụ nếu cần.
4.  **Denormalize dữ liệu** – tức là lặp lại thông tin để tránh phải “JOIN kiểu NoSQL”.

Ví dụ với PetShop, bạn muốn hỗ trợ các truy vấn:

*   Lấy thông tin khách hàng theo `userid`.
*   Lấy tất cả đơn hàng của một khách hàng.
*   Lấy danh sách thú cưng theo loại (mèo, chó).

Thay vì ba bảng, bạn tạo bảng `PetShop` với:

*   Partition key: `PK` (ví dụ: `USER#<user_id>` cho khách, `PET#<pet_id>` cho thú cưng).
*   Sort key: `SK` (ví dụ: `INFO` cho thông tin chính, `ORDER#<order_id>` cho đơn hàng).

Mỗi item trong bảng có thể trông như sau:

| PK        | SK           | name | email           | order_id | user_id | total  | type | price  |
| :-------- | :----------- | :--- | :-------------- | :------- | :------ | :----- | :--- | :----- |
| USER#U001 | INFO         | Nam  | nam@gmail.com   |          |         |        |      |        |
| USER#U001 | ORDER#O001   |      |                 | O001     | U001    | 500000 |      |        |
| PET#P001  | INFO         | Miu  |                 |          |         |        | cat  | 500000 |

Giờ muốn lấy đơn hàng của Nam? Chỉ cần Query: `PK = "USER#U001" AND SK begins_with "ORDER#"`. Muốn query info của Nam? `PK = "USER#U001" AND SK begins_with "INFO"`.

### Bí kíp denormalization: Sao chép "thông minh"

MySQL dạy bạn tránh lặp dữ liệu để tiết kiệm chỗ, nhưng DynamoDB thì bảo: “Sao chép đi, đừng ngại!”. Denormalization là cách lặp lại thông tin để truy vấn nhanh hơn, vì DynamoDB không có JOIN.

Ở PetShop, bạn muốn hiển thị danh sách đơn hàng kèm tên khách hàng mà không cần query thêm. Thay vì chỉ lưu `user_id` trong đơn hàng, bạn sao chép cả tên khách luôn:

| PK        | SK         | order_id | user_id | user_name | total  |
| :-------- | :--------- | :------- | :------ | :-------- | :----- |
| USER#U001 | ORDER#O001 | O001     | U001    | Nam       | 500000 |

Giờ lấy đơn hàng, bạn có ngay tên “Nam” mà không cần chạy Query đến item user với id "U001" nữa. Nhược điểm? Nếu tên user đổi, bạn phải cập nhật cả ở đơn hàng của user nhé – nhưng truy vấn nhanh hơn. Có đáng để trade-off không thì bạn là người quyết =)))

### Dùng GSI để “hack” truy vấn linh hoạt

Partition key và sort key đôi khi không đủ. Global Secondary Index (GSI) là cứu tinh, giống như một “góc nhìn khác” của bảng chính, cho phép truy vấn theo cách mới.

PetShop muốn tìm tất cả thú cưng là mèo. Bạn thêm `GSI1` cho bảng và thêm 2 field `GSI1PK` + `GSI1SK` như sau:

| PK       | SK   | name | type | price  | GSI1PK   | GSI1SK |
| :------- | :--- | :--- | :--- | :----- | :------- | :----- |
| PET#P001 | INFO | Miu  | cat  | 500000 | TYPE#CAT | INFO   |

Muốn lấy tất cả mèo? Query GSI1: `GSI1PK = "TYPE#CAT"`, `GSI1SK = "INFO"`. Kết quả trả về danh sách mèo ngay.

### STD có phải silver bullet không?

STD nghe đỉnh, nhưng không phải lúc nào cũng hoàn hảo.

*   **Ưu điểm:** Tiết kiệm chi phí, truy vấn nhanh, quản lý dễ. Với PetShop, một bảng xử lý từ khách hàng đến đơn hàng, bớt đau đầu.
*   **Nhược điểm:** Thiết kế ban đầu cần tư duy kỹ, thay đổi sau hơi mệt. Nếu cần báo cáo phức tạp như “doanh thu mèo so với chó theo tháng”, STD có thể khiến bạn toát mồ hôi hơn MySQL. Đây cũng là lúc tận dụng DynamoDB Stream, nhưng đó là 1 câu chuyện khác =))

PetShop chạy ngon với STD cho truy vấn như xem đơn hàng, tìm thú cưng. Nhưng nếu muốn phân tích “tỷ lệ mua mèo vào thứ Bảy”, có thể cần export dữ liệu sang Redshift, vì STD không mạnh về báo cáo phức tạp.

### Kết luận: STD – Một thiết kế đáng để thử

Single Table Design là một trong những lý do khiến DynamoDB phổ biến – nhanh, rẻ, mạnh. Với PetShop, thay vì đau đầu với ba bảng, bạn dùng một bảng để xử lý mọi thứ, từ tìm mèo đến xem đơn hàng. Nhưng nhớ, STD đòi hỏi bạn nghĩ trước, thiết kế kỹ, và sẵn sàng denormalize một cách thông minh.
```