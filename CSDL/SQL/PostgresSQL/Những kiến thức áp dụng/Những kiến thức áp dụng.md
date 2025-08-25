# 1  Theo dõi Đơn hàng Theo Thời gian Thực với Estuary, MotherDuck và Hex sử dụng CDC

Dự án này minh họa cách xây dựng một pipeline dữ liệu sử dụng Change Data Capture (CDC) để theo dõi và phân tích toàn bộ lịch sử thay đổi của các đơn hàng trong thời gian thực. Bằng cách ghi lại mọi sự kiện thay đổi thay vì chỉ lưu trạng thái cuối cùng, chúng ta có thể hiểu rõ hơn về hành vi của hệ thống và đưa ra các phân tích sâu sắc hơn.

Tất cả code được sử dụng trong bài viết gốc có sẵn [tại đây](<link-to-code-repo>) và notebook Hex có sẵn [tại đây](<link-to-hex-notebook>). (Lưu ý: Bạn cần thay thế `<link-to-code-repo>` và `<link-to-hex-notebook>` bằng các liên kết thực tế nếu có).

## Vấn đề: Chỉ lưu Trạng thái Hiện tại

Trong hầu hết các pipeline dữ liệu truyền thống, bạn chỉ thấy trạng thái hiện tại của sự vật. Bạn biết đơn hàng trông như thế nào *ngay bây giờ*, nhưng không dễ dàng thấy nó trông như thế nào *trước đây* hoặc nó đã thay đổi *như thế nào* theo thời gian.

Đây là một hạn chế khi bạn cố gắng hiểu hành vi hệ thống, gỡ lỗi sự cố hoặc phân tích các mẫu hoạt động.

## Giải pháp: Sử dụng CDC

Change Data Capture (CDC) cho phép bạn ghi lại *mọi* sự thay đổi. Nó không chỉ cho bạn biết dữ liệu *là gì* — nó cho bạn biết dữ liệu *đã là gì*, *khi nào* nó thay đổi và *làm thế nào*. Khi kết hợp với một nền tảng CDC như Estuary và công cụ phân tích như MotherDuck, bạn có thể lưu trữ và truy vấn *mọi phiên bản* của một bản ghi, không chỉ phiên bản mới nhất.

Điều này hữu ích cho các trường hợp sử dụng mà thứ tự và thời gian của các thay đổi là quan trọng. Ví dụ, trong một hệ thống logistics, bạn muốn biết một đơn hàng đã thay đổi trạng thái bao nhiêu lần, nó đã đi qua những trạng thái nào và nó ở mỗi trạng thái bao lâu. Bạn không thể làm điều đó chỉ từ ảnh chụp (snapshot) dữ liệu mới nhất. Bạn cần lịch sử.

CDC cho phép bạn nắm bắt lịch sử này. Estuary giúp dễ dàng truyền tải (stream) nó vào các hệ thống như MotherDuck, nơi bạn có thể viết SQL để tái tạo lịch sử đó và tính toán các chỉ số phụ thuộc vào nó. Điều này cho phép bạn xây dựng các phân tích hữu ích hơn và hiểu rõ hơn về cách hệ thống của bạn hoạt động theo thời gian.

![alt text](image.png)

## Khái niệm: Theo dõi Lịch sử Thay đổi Trạng thái Đơn hàng bằng CDC

Trong dự án này, chúng tôi mô phỏng một hệ thống logistics cho một cửa hàng vật nuôi trực tuyến. Khách hàng đặt hàng các sản phẩm như đồ chơi chó, máng ăn chim, thức ăn cá. Mỗi đơn hàng trải qua một loạt các trạng thái hoàn thành: `placed` (đã đặt), `packed` (đã đóng gói), `shipped` (đã gửi), `delivered` (đã giao), hoặc `cancelled` (đã hủy).

Dữ liệu đơn hàng được lưu trữ trong bảng PostgreSQL. Một script sẽ liên tục chèn đơn hàng mới và cập nhật các đơn hàng hiện có một cách ngẫu nhiên để mô phỏng các thay đổi trạng thái trong thế giới thực. Một số đơn hàng đi từ `placed` đến `delivered`. Một số khác bị hủy giữa chừng hoặc chuyển đổi qua lại giữa các trạng thái.

Mỗi lần một đơn hàng thay đổi, một sự kiện thay đổi mới sẽ được ghi vào write-ahead log của PostgreSQL. Estuary Flow đọc những thay đổi này bằng cách sử dụng CDC và truyền tải chúng vào MotherDuck. Điều này có nghĩa là MotherDuck nhận được mọi thao tác chèn (insert) và cập nhật (update), theo đúng thứ tự, cho mọi đơn hàng.

Chúng ta không cần thêm cột `updated_at` hoặc quản lý phiên bản thủ công. Hệ thống tự động theo dõi mọi thay đổi. Trong MotherDuck, chúng ta có thể truy vấn toàn bộ lịch sử của từng đơn hàng để xem nó đã thay đổi bao nhiêu lần, nó đã đi qua những trạng thái nào và khi nào những thay đổi đó xảy ra. Điều này cho phép chúng ta phân tích các mẫu hoạt động không thể thấy trong hệ thống dựa trên snapshot.

![alt text](image-1.png)

## Stack Công nghệ

Dự án này sử dụng bốn thành phần chính để mô phỏng dữ liệu đơn hàng theo thời gian thực và phân tích nó bằng CDC:

*   **PostgreSQL:**
    *   Nguồn dữ liệu.
    *   Một script liên tục ghi đơn hàng mới và cập nhật các đơn hàng hiện có trong một bảng `orders`.
    *   Mỗi dòng bao gồm `customer_name`, `product_name`, `status` và dấu thời gian `created_at`.
    *   Các bản cập nhật trạng thái mô phỏng các thay đổi trong quy trình hoàn thành đơn hàng.

    ![alt text](image-2.png)

*   **Estuary Flow:**
    *   Nền tảng CDC.
    *   Bắt mọi thao tác chèn và cập nhật từ PostgreSQL bằng cách sử dụng logical replication.
    *   Trích xuất các thay đổi từ write-ahead log và gửi chúng đi xuống dòng.
    *   Mỗi sự kiện thay đổi được gắn phiên bản và dấu thời gian, bao gồm siêu dữ liệu về thao tác (insert hoặc update).

    ![alt text](image-3.png)

*   **MotherDuck:**
    *   Kho dữ liệu.
    *   Nhận toàn bộ nhật ký thay đổi dưới dạng một bảng material hóa.
    *   Đây *không phải* là một snapshot đã loại bỏ trùng lặp — nó là một bản ghi có thứ tự thời gian của mọi thay đổi đối với mọi đơn hàng.
    *   Tất cả các phiên bản của mỗi dòng được bảo toàn, cho phép phân tích dựa trên thời gian, đếm số lần chuyển trạng thái và tái tạo lại trạng thái đầy đủ bằng SQL.

    ![alt text](image-4.png)

*   **Hex:**
    *   Lớp BI/Phân tích.
    *   Được sử dụng để khám phá dữ liệu trong MotherDuck.
    *   Cho phép chúng ta viết các truy vấn SQL để nhóm và lọc các sự kiện thay đổi, tính toán các chỉ số theo thời gian và trực quan hóa các xu hướng như số lần thay đổi trạng thái trung bình trên mỗi đơn hàng hoặc tỷ lệ hủy đơn hàng theo thời gian.

    ![alt text](image-5.png)

Stack này cung cấp khả năng hiển thị đầy đủ về vòng đời đơn hàng bằng cách sử dụng dữ liệu streaming. Nó không yêu cầu xử lý theo lô (batch processing) hoặc theo dõi sự kiện tùy chỉnh. CDC và việc thu nhận dữ liệu có thứ tự thời gian giúp có thể phân tích toàn bộ sự phát triển của các bản ghi gần như thời gian thực.

## Cách Tái tạo Dòng thời gian Đơn hàng từ Dữ liệu Thay đổi

Thay vì chỉ làm việc với trạng thái mới nhất của mỗi đơn hàng, chúng ta lưu trữ *mọi* thay đổi. Điều này cho phép chúng ta tái tạo hoàn toàn vòng đời của mỗi đơn hàng dựa trên lịch sử thay đổi của nó.

Mỗi lần một đơn hàng được chèn hoặc cập nhật trong PostgreSQL, Estuary sẽ bắt sự thay đổi đó và ghi vào MotherDuck dưới dạng một dòng mới. Bảng kết quả trong MotherDuck chứa một dòng cho mỗi sự kiện, bao gồm cả thao tác chèn và cập nhật. Mỗi dòng có một `order_id`, trạng thái hiện tại (`status`) và một dấu thời gian (`flow_published_at`) hiển thị khi thay đổi xảy ra.

Để tái tạo dòng thời gian của một đơn hàng, chúng ta có thể truy vấn tất cả các sự kiện cho một `order_id` nhất định và sắp xếp chúng theo dấu thời gian của chúng. Điều này cho chúng ta thấy trình tự chính xác các trạng thái mà một đơn hàng đã đi qua. Nó cũng cho phép chúng ta đếm số lần một đơn hàng thay đổi, điều này hữu ích cho việc đo lường mức độ phức tạp của hoạt động.

Chúng ta cũng có thể nhóm các thay đổi này trên nhiều đơn hàng để hiểu các mẫu hình hệ thống. Ví dụ, chúng ta có thể xác định có bao nhiêu đơn hàng hiện đang ở mỗi trạng thái bằng cách chỉ chọn sự kiện thay đổi *gần nhất* cho mỗi `order_id`. Chúng ta có thể tính toán số lần chuyển trạng thái mà các đơn hàng thường trải qua trước khi được giao. Và chúng ta có thể xem những sản phẩm nào có đường dẫn hoàn thành đơn hàng biến động nhất bằng cách tổng hợp số lần chuyển trạng thái theo từng sản phẩm.

Cách tiếp cận này mang lại cho chúng ta các phân tích phản ánh không chỉ *điều gì* đã xảy ra, mà còn *làm thế nào* nó xảy ra. Nhật ký thay đổi đầy đủ cho phép chúng ta tái tạo trạng thái theo thời gian, đo lường các chuyển đổi và phân tích hành vi hoàn thành đơn hàng theo những cách không thể thực hiện được với dữ liệu dựa trên snapshot.

![alt text](image-6.png)

## Các Chỉ số Chỉ có được nhờ CDC

Vì chúng ta lưu trữ mọi sự kiện thay đổi, chúng ta có thể tính toán các chỉ số mà không thể suy ra được từ một snapshot trạng thái hiện tại. Các chỉ số này phụ thuộc vào toàn bộ lịch sử của mỗi đơn hàng, không chỉ trạng thái mới nhất của nó.

Dưới đây là các ví dụ về các chỉ số có thể thực hiện được nhờ CDC (sử dụng cú pháp SQL cho MotherDuck):

1.  **Số lần thay đổi trạng thái trung bình trên mỗi đơn hàng đã giao thành công:**
    Truy vấn này cho biết mức độ phức tạp của quy trình hoàn thành đối với các đơn hàng đã được giao thành công. Chúng ta đếm số sự kiện trên mỗi `order_id`, sau đó lọc để chỉ bao gồm những đơn hàng đạt trạng thái cuối cùng là 'delivered'.

    ```sql
    WITH status_changes AS (
      SELECT order_id, COUNT(*) AS change_count
      FROM orders.main.orders
      GROUP BY order_id
    ),
    delivered_orders AS (
      SELECT DISTINCT order_id
      FROM orders.main.orders
      WHERE status = 'delivered'
    )
    SELECT AVG(change_count) AS avg_status_changes
    FROM status_changes
    WHERE order_id IN (SELECT order_id FROM delivered_orders);
    ```

2.  **Phân bổ trạng thái cuối cùng (snapshot hiện tại):**
    Chúng ta có thể có một snapshot về trạng thái hiện tại của mọi đơn hàng bằng cách chỉ chọn sự kiện gần nhất cho mỗi `order_id`. Điều này cho phép chúng ta xem có bao nhiêu đơn hàng hiện đang ở trạng thái 'delivered', 'cancelled', 'shipped', v.v.

    ```sql
    WITH latest_events AS (
      SELECT *,
             ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY flow_published_at DESC) AS rn
      FROM orders.main.orders
    )
    SELECT status, COUNT(*) AS order_count
    FROM latest_events
    WHERE rn = 1
    GROUP BY status
    ORDER BY order_count DESC;
    ```

3.  **Số lần thay đổi trạng thái trung bình trên mỗi sản phẩm:**
    Chúng ta có thể nhóm tất cả các sự kiện đơn hàng theo `product_name` và `order_id`, đếm số lần thay đổi mà mỗi đơn hàng đã trải qua, và sau đó tính trung bình các lượt đếm đó theo sản phẩm.

    ```sql
    WITH change_counts AS (
      SELECT product_name, order_id, COUNT(*) AS changes
      FROM orders.main.orders
      GROUP BY product_name, order_id
    ),
    avg_per_product AS (
      SELECT product_name, AVG(changes) AS avg_status_changes
      FROM change_counts
      GROUP BY product_name
    )
    SELECT *
    FROM avg_per_product
    ORDER BY avg_status_changes DESC
    LIMIT 10;
    ```

4.  **Tỷ lệ Giao thành công so với Hủy đơn hàng theo thời gian:**
    Bằng cách nhóm vòng đời đơn hàng theo ngày và trạng thái cuối cùng, chúng ta có thể trực quan hóa tỷ lệ giao hàng thành công và xu hướng hủy đơn hàng.

    ```sql
    WITH latest_statuses AS (
      SELECT order_id, status, DATE(flow_published_at) AS day,
             ROW_NUMBER() OVER (
               PARTITION BY order_id
               ORDER BY flow_published_at DESC
             ) AS rn
      FROM orders.main.orders
    )
    SELECT
      day,
      COUNT(*) FILTER (WHERE status = 'delivered') AS delivered,
      COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled,
      COUNT(*) AS total
    FROM latest_statuses
    WHERE rn = 1
    GROUP BY day
    ORDER BY day;
    ```

Những ví dụ này làm nổi bật cách CDC cho phép hiển thị các mẫu hình bị ẩn trong các mô hình truyền thống. Thay vì chỉ phân tích *đơn hàng hiện đang ở đâu*, chúng ta có thể phân tích *nó đã đến đó như thế nào*.

## Tại sao Điều này Quan trọng với các Đội nhóm Thực tế

Nếu bạn đang vận hành một hệ thống logistics, thương mại điện tử hoặc hoàn thành đơn hàng trong thế giới thực, bạn cần nhiều hơn là chỉ giám sát trạng thái hiện tại. Bạn cần hiểu rõ về cách mọi thứ đang thay đổi và tại sao.

Cách tiếp cận này cho phép bạn:

*   Theo dõi số lần đơn hàng thay đổi trạng thái trước khi giao hoặc hủy.
*   Xác định các điểm không hiệu quả trong hoạt động, chẳng hạn như các đơn hàng chuyển đổi qua lại giữa các trạng thái.
*   So sánh hành vi hoàn thành đơn hàng giữa các sản phẩm để tìm các mẫu hình không nhất quán.
*   Xây dựng dòng thời gian để gỡ lỗi các đơn hàng bị kẹt hoặc bị trì hoãn.
*   Kiểm tra toàn bộ vòng đời của một đơn hàng cho mục đích tuân thủ hoặc hỗ trợ khách hàng.

Điều này có giá trị cho các đội vận hành, phân tích và kỹ sư. Nó cung cấp cho mọi người quyền truy cập vào cùng một nguồn sự thật cơ bản — lịch sử hoàn chỉnh về những gì đã xảy ra, khi nào nó xảy ra và hệ thống đã phản hồi như thế nào.

Khả năng phát lại dữ liệu thay đổi cho phép các đội nhóm phát hiện vấn đề sớm hơn, hiểu rõ nguyên nhân gốc rễ nhanh hơn và tối ưu hóa dựa trên hành vi thực tế thay vì giả định.

## Cách Tự chạy Dự án

Bạn có thể tự xây dựng pipeline này bằng cách sử dụng các công cụ tương tự:

1.  **Mô phỏng dữ liệu:** Chạy một script tạo và cập nhật đơn hàng trong PostgreSQL.
2.  **Bật Logical Replication:** Cấu hình PostgreSQL để bật logical replication.
3.  **Thiết lập Estuary Flow:**
    *   Tạo một connector **Postgres CDC** trong Estuary để bắt các thay đổi từ bảng `orders` của bạn.
    *   Thiết lập một **Materialization** trong Estuary để ghi những thay đổi đó vào một bảng trong MotherDuck.
4.  **Phân tích bằng Hex:** Sử dụng Hex để kết nối với MotherDuck và xây dựng các truy vấn như các ví dụ trong bài viết gốc.

Bạn không cần viết các bộ phát sự kiện tùy chỉnh hoặc quản lý cột dấu thời gian thủ công. Estuary và CDC sẽ lo việc theo dõi thay đổi. Công việc của bạn là viết các truy vấn sử dụng dữ liệu đó để trích xuất thông tin chi tiết.

![alt text](image-7.png)
## Kết luận: Lịch sử là Hệ thống

Nhiều hệ thống dữ liệu được thiết kế để hiển thị cho bạn giá trị mới nhất. Nhưng trong hoạt động, lịch sử về cách giá trị đó thay đổi thường quan trọng hơn bản thân giá trị.

CDC mang đến cho bạn lịch sử đó. Kết hợp với các công cụ như Estuary và MotherDuck, nó trở nên khả thi để tái tạo hành vi, theo dõi thay đổi và đo lường sự phát triển của mọi bản ghi. Bạn có thể gỡ lỗi, tối ưu hóa và phân tích dựa trên *những gì thực sự đã xảy ra*, chứ không chỉ *dữ liệu trông như thế nào bây giờ*.

Pipeline này cung cấp cho bạn khả năng hiển thị theo thời gian thực và khả năng truy vết lịch sử trong một hệ thống duy nhất. Nếu bạn đang xây dựng hệ thống hướng tới độ tin cậy, tính minh bạch hoặc trách nhiệm giải trình, sự kết hợp đó là thiết yếu.

---

**Dựa trên bài viết:** [Real-Time Order Tracking with Estuary, MotherDuck, and Hex using CDC](<link-to-original-article>) của Dani Palma.


# 2 Sử dụng PostgreSQL như một Công cụ Tìm kiếm? Vâng, Có Thể Bạn Không Cần Đến Elasticsearch
Sponsor by https://leapcell.io/blog/postgresql-as-search-engine?ref=dailydev

*Bài viết gốc: James Reed, Infrastructure Engineer · Leapcell*
*Ngày: 09 tháng 5 năm 2025*
*(Viết lại từ bài viết tiếng Anh)*

## Nguyên lý Chỉ mục Đảo ngược (Inverted Index)

Chỉ mục đảo ngược bắt nguồn từ công nghệ công cụ tìm kiếm và được coi là nền tảng của chúng. Nhờ công nghệ chỉ mục đảo ngược, công cụ tìm kiếm có thể thực hiện các thao tác như tìm kiếm và xóa dữ liệu một cách hiệu quả. Trước khi đi sâu vào chỉ mục đảo ngược, chúng ta sẽ giới thiệu chỉ mục thuận (forward index) liên quan và so sánh hai loại này.

### Chỉ mục Thuận (Forward Index)

Trong một công cụ tìm kiếm, bảng chỉ mục thuận sử dụng ID tài liệu làm từ khóa, và bảng ghi lại thông tin vị trí của từng từ trong tài liệu. Khi tìm kiếm, hệ thống sẽ quét thông tin từ trong từng tài liệu trong bảng cho đến khi tìm thấy tất cả các tài liệu chứa từ khóa truy vấn.

Cấu trúc của bảng chỉ mục thuận có thể được biểu diễn bằng sơ đồ sau:

```
+---------------------+
|   Bảng Chỉ mục Thuận  |
+---------------------+
|  ID Tài liệu  |  Thông tin Vị trí |
+---------------------+
|   Doc1   |  word1@3 |
|          |  word2@7 |
+---------------------+
|   Doc2   |  word1@2 |
|          |  word3@5 |
+---------------------+
|   Doc3   |  word2@4 |
|          |  word4@6 |
+---------------------+
```

Phương pháp tổ chức này có cấu trúc tương đối đơn giản khi xây dựng chỉ mục, dễ dàng xây dựng và bảo trì. Do chỉ mục được xây dựng dựa trên tài liệu, nếu thêm một tài liệu mới, chỉ cần tạo một khối chỉ mục mới cho tài liệu này và gắn vào cuối tệp chỉ mục gốc; nếu xóa một tài liệu, thông tin chỉ mục tương ứng với ID tài liệu có thể được tìm thấy và xóa trực tiếp. Tuy nhiên, khi truy vấn, cần quét tất cả các tài liệu để đảm bảo không bỏ sót, điều này sẽ kéo dài đáng kể thời gian truy xuất và dẫn đến hiệu quả truy xuất thấp.

Mặc dù nguyên tắc hoạt động của bảng chỉ mục thuận rất đơn giản, hiệu quả truy xuất của nó quá thấp và có ít giá trị thực tế trừ trong các tình huống cụ thể.

### Chỉ mục Đảo ngược (Inverted Index)

Bảng chỉ mục đảo ngược sử dụng các từ hoặc thuật ngữ làm từ khóa cho việc lập chỉ mục, và các mục nhập tương ứng với các từ khóa trong bảng ghi lại tất cả các tài liệu mà từ hoặc thuật ngữ này xuất hiện.

Cấu trúc của bảng chỉ mục đảo ngược có thể được biểu diễn bằng sơ đồ sau:

```
+---------------------+
|   Bảng Chỉ mục Đảo ngược  |
+---------------------+
|  Từ khóa   |  Danh sách Tài liệu |
+---------------------+
|  word1    |  Doc1, Doc2 |
+---------------------+
|  word2    |  Doc1, Doc3 |
+---------------------+
|  word3    |  Doc2      |
+---------------------+
|  word4    |  Doc3      |
+---------------------+
```

Vì số lượng tài liệu tương ứng với mỗi từ hoặc thuật ngữ thay đổi động, việc xây dựng và bảo trì bảng chỉ mục đảo ngược phức tạp hơn. Tuy nhiên, khi truy vấn, vì tất cả các tài liệu tương ứng với từ khóa truy vấn có thể được lấy ngay lập tức, hiệu quả cao hơn so với bảng chỉ mục thuận. Trong truy xuất toàn văn, phản hồi nhanh của truy xuất là một hiệu năng quan trọng. Mặc dù việc xây dựng chỉ mục tương đối chậm vì nó được thực hiện ngầm (trong nền), nó sẽ không ảnh hưởng đến hiệu quả của toàn bộ công cụ tìm kiếm.

## Chỉ mục GIN trong PostgreSQL

### Tổng quan

GIN là viết tắt của **Generalized Inverted Index**, tức là chỉ mục đảo ngược tổng quát. Các giá trị của các kiểu dữ liệu mà nó xử lý không phải là nguyên tử mà được cấu tạo từ các phần tử, mà chúng ta gọi là kiểu dữ liệu phức hợp. Ví dụ, trong (hank, 15:3 21:4), nó có nghĩa là "hank" xuất hiện ở các vị trí 15:3 và 21:4. Phần sau sẽ giúp chúng ta hiểu rõ hơn về chỉ mục GIN thông qua các ví dụ cụ thể.

### Cấu trúc Chỉ mục GIN

#### Cấu trúc Vật lý

Lưu trữ vật lý của chỉ mục GIN chứa các nội dung sau:

*   **Entry (Mục nhập):** Một phần tử trong chỉ mục GIN, có thể coi là một vị trí từ và cũng có thể hiểu là một khóa.
*   **Entry tree (Cây Mục nhập):** Một cây B-tree được xây dựng trên các Entry.
*   **Posting list (Danh sách Vị trí):** Một danh sách liên kết các vị trí vật lý (heap ctid, số dòng trong bảng heap) nơi một Entry xuất hiện.
*   **Posting tree (Cây Vị trí):** Một cây B-tree được xây dựng trên danh sách liên kết các vị trí vật lý (heap ctid, số dòng trong bảng heap) nơi một Entry xuất hiện. Vì vậy, KHÓA của cây vị trí là ctid, còn KHÓA của cây mục nhập là giá trị của cột được lập chỉ mục.
*   **Pending list (Danh sách Chờ xử lý):** Một danh sách liên kết lưu trữ tạm thời các tuple chỉ mục, được sử dụng cho các thao tác chèn trong chế độ `fastupdate`.

Từ những điều trên, có thể thấy chỉ mục GIN chủ yếu bao gồm cây mục nhập (Entry tree) và cây vị trí (posting tree) (hoặc danh sách vị trí - posting list), trong đó cây mục nhập là cấu trúc chính của chỉ mục GIN, và cây vị trí là cây phụ trợ.

Cây mục nhập tương tự như b+tree, và cây vị trí tương tự như b-tree.

Ngoài ra, cả cây mục nhập và cây vị trí đều được tổ chức theo thứ tự dựa trên KHÓA.

#### Cấu trúc Logic

Về mặt logic, chỉ mục GIN có thể được coi là một mối quan hệ, và mối quan hệ này có hai cấu trúc:

**Lập chỉ mục chỉ một cột của bảng gốc**

| key  | value                     |
| :--- | :------------------------ |
| key1 | Danh sách Vị trí (hoặc cây vị trí) |
| key2 | Danh sách Vị trí (hoặc cây vị trí) |
| …    | …                         |

**Lập chỉ mục nhiều cột của bảng gốc (chỉ mục phức hợp, đa cột)**

| column_id   | key  | value                     |
| :---------- | :--- | :------------------------ |
| Số cột 1    | key1 | Danh sách Vị trí (hoặc cây vị trí) |
| Số cột 2    | key1 | Danh sách Vị trí (hoặc cây vị trí) |
| Số cột 3    | key1 | Danh sách Vị trí (hoặc cây vị trí) |
| …           | …    | …                         |

Từ đây có thể thấy, theo cấu trúc này, đối với cùng một khóa trong các cột khác nhau của bảng gốc, nó cũng sẽ được coi là một khóa khác trong chỉ mục GIN.

### Truy xuất Toàn văn (Full-text Retrieval)

Lĩnh vực ứng dụng chính của GIN là tăng tốc tìm kiếm toàn văn. Do đó, ở đây chúng ta sẽ giới thiệu chỉ mục GIN bằng một ví dụ về tìm kiếm toàn văn.

Tạo một bảng, trong đó `doc_tsv` có kiểu dữ liệu tìm kiếm văn bản (`tsvector`), kiểu này có thể tự động sắp xếp và loại bỏ các phần tử trùng lặp:

```sql
pg_study=# create table ts(doc text, doc_tsv tsvector);
CREATE TABLE

pg_study=# insert into ts(doc) values
  ('Can a sheet slitter slit sheets?'),
  ('How many sheets could a sheet slitter slit?'),
  ('I slit a sheet, a sheet I slit.'),
  ('Upon a slitted sheet I sit.'),
  ('Whoever slit the sheets is a good sheet slitter.'),
  ('I am a sheet slitter.'),
  ('I slit sheets.'),
  ('I am the sleekest sheet slitter that ever slit sheets.'),
  ('She slits the sheet she sits on.');
INSERT 0 9

pg_study=# update ts set doc_tsv = to_tsvector(doc);
UPDATE 9

pg_study=# create index on ts using gin(doc_tsv);
CREATE INDEX
```

Cấu trúc của chỉ mục GIN này như sau. Các ô vuông màu đen là số TID (số dòng), và các ô vuông màu trắng là các từ. Lưu ý rằng đây là một danh sách liên kết đơn, khác với danh sách liên kết đôi của B-tree:

```
+--------+     +--------+     +--------+
|  sheet |---->|  slit  |---->| slitter|
+--------+     +--------+     +--------+
   |             |             |
   v             v             v
+--------+   +--------+   +--------+
| (0,10) |   | (0,10) |   | (0,10) |
+--------+   +--------+   +--------+
   |             |             |
   v             v             v
+--------+   +--------+   +--------+
| (0,11) |   | (0,11) |   | (0,11) |
+--------+   +--------+   +--------+
   |             |             |
   v             v             v
   ...           ...           ...
```

Hãy xem một ví dụ khác:

```sql
pg_study=# select ctid,doc, doc_tsv from ts;
  ctid  |                          doc                           |                         doc_tsv
--------+--------------------------------------------------------+---------------------------------------------------------
 (0,10) | Can a sheet slitter slit sheets?                       | 'sheet':3,6 'slit':5 'slitter':4
 (0,11) | How many sheets could a sheet slitter slit?            | 'could':4 'mani':2 'sheet':3,6 'slit':8 'slitter':7
 (0,12) | I slit a sheet, a sheet I slit.                        | 'sheet':4,6 'slit':2,8
 (0,13) | Upon a slitted sheet I sit.                            | 'sheet':4 'sit':6 'slit':3 'upon':1
 (0,14) | Whoever slit the sheets is a good sheet slitter.       | 'good':7 'sheet':4,8 'slit':2 'slitter':9 'whoever':1
 (0,15) | I am a sheet slitter.                                  | 'sheet':4 'slitter':5
 (0,16) | I slit sheets.                                         | 'sheet':3 'slit':2
 (0,17) | I am the sleekest sheet slitter that ever slit sheets. | 'ever':8 'sheet':5,10 'sleekest':4 'slit':9 'slitter':6
 (0,18) | She slits the sheet she sits on.                       | 'sheet':4 'sit':6 'slit':2
(9 rows)
```

Có thể thấy từ trên rằng "sheet", "slit", và "slitter" xuất hiện trong nhiều dòng, vì vậy sẽ có nhiều TID. Trong trường hợp này, một danh sách TID sẽ được tạo ra, và một cây B-tree riêng biệt sẽ được tạo cho nó.

Câu lệnh sau có thể tìm ra từ xuất hiện trong bao nhiêu dòng.

```sql
pg_study=# select (unnest(doc_tsv)).lexeme, count(*) from ts
group by 1 order by 2 desc;
  lexeme  | count
----------+-------
 sheet    |     9
 slit     |     8
 slitter  |     5
 sit      |     2
 upon     |     1
 mani     |     1
 whoever  |     1
 sleekest |     1
 good     |     1
 could    |     1
 ever     |     1
(11 rows)
```

### Ví dụ Truy vấn

Truy vấn sau được thực thi như thế nào?

```sql
-- Vì lượng dữ liệu ở đây nhỏ, chúng ta tắt full table scan
pg_study=# set enable_seqscan TO off;
SET

pg_study=# explain(costs off)
select doc from ts where doc_tsv @@ to_tsquery('many & slitter');
                             QUERY PLAN
---------------------------------------------------------------------
 Bitmap Heap Scan on ts
   Recheck Cond: (doc_tsv @@ to_tsquery('many & slitter'::text))
   ->  Bitmap Index Scan on ts_doc_tsv_idx
         Index Cond: (doc_tsv @@ to_tsquery('many & slitter'::text))
(4 rows)
```

Đầu tiên, trích xuất từng từ (khóa truy vấn) từ truy vấn: `mani` và `slitter`. Điều này được hoàn thành bởi một API đặc biệt, sẽ xem xét các chiến lược của kiểu dữ liệu và toán tử:

```sql
pg_study=# select amop.amopopr::regoperator, amop.amopstrategy
from pg_opclass opc, pg_opfamily opf, pg_am am, pg_amop amop
where opc.opcname = 'tsvector_ops'
and opf.oid = opc.opcfamily
and am.oid = opf.opfmethod
and amop.amopfamily = opc.opcfamily
and am.amname = 'gin'
and amop.amoplefttype = opc.opcintype;
        amopopr        | amopstrategy
-----------------------+--------------
 @@(tsvector,tsquery)  |            1
 @@@(tsvector,tsquery) |            2
(2 rows)
```

Trong cây B-tree chứa các từ, tìm danh sách TID tương ứng với hai khóa:

`mani`: (0,2)
`slitter`: (0,1), (0,2), (1,2), (1,3), (2,2)

Cuối cùng, đối với mỗi TID được tìm thấy, lần lượt gọi hàm kiểm tra tính nhất quán (consistency function). Hàm này có thể xác định xem dòng được trỏ bởi TID có đáp ứng điều kiện truy vấn hay không. Vì các từ trong truy vấn được kết nối bằng toán tử `AND` (`&`), dòng được trả về chỉ là (0,2).

```
       |      |         |  consistency
       |      |         |    function
  TID  | mani | slitter | mani & slitter
-------+------+---------+----------------
 (0,1) |    f |       T |              f
 (0,2) |    T |       T |              T
 (1,2) |    f |       T |              f
 (1,3) |    f |       T |              f
 (2,2) |    f |       T |              f
```

Kết quả là:

```sql
pg_study=# select doc from ts where doc_tsv @@ to_tsquery('many & slitter');
                     doc
---------------------------------------------
 How many sheets could a sheet slitter slit?
(1 row)
```

### Vấn đề Tốc độ Cập nhật Chậm

Thao tác chèn hoặc cập nhật dữ liệu trong chỉ mục GIN rất chậm. Bởi vì mỗi dòng thường chứa nhiều phần tử từ cần được lập chỉ mục. Do đó, khi thêm hoặc cập nhật một dòng, chúng ta phải cập nhật cây chỉ mục rất nhiều lần.

Mặt khác, nếu nhiều dòng được cập nhật cùng lúc, một số phần tử từ của chúng có thể giống nhau, do đó tổng chi phí sẽ ít hơn so với chi phí cập nhật từng tài liệu riêng lẻ.

Chỉ mục GIN có một tham số lưu trữ là `fastupdate`, chúng ta có thể chỉ định nó khi tạo chỉ mục và cập nhật sau này:

```sql
pg_study=# create index on ts using gin(doc_tsv) with (fastupdate = true);
CREATE INDEX
```

Sau khi bật tham số này, các cập nhật sẽ được tích lũy trong một danh sách riêng biệt, không được sắp xếp. Khi danh sách này đủ lớn hoặc trong quá trình `VACUUM` (thu gom rác), tất cả các cập nhật tích lũy sẽ được thao tác ngay lập tức trên chỉ mục. Danh sách "đủ lớn" được xác định bởi tham số cấu hình `gin_pending_list_limit` hoặc tham số lưu trữ cùng tên khi tạo chỉ mục.

### Tìm kiếm Khớp một phần (Partial Match Search)

Truy vấn các tài liệu bắt đầu bằng `slit`:

```sql
pg_study=# select doc from ts where doc_tsv @@ to_tsquery('slit:*');
                          doc
--------------------------------------------------------
 Can a sheet slitter slit sheets?
 How many sheets could a sheet slitter slit?
 I slit a sheet, a sheet I slit.
 Upon a slitted sheet I sit.
 Whoever slit the sheets is a good sheet slitter.
 I am a sheet slitter.
 I slit sheets.
 I am the sleekest sheet slitter that ever slit sheets.
 She slits the sheet she sits on.
(9 rows)
```

Chỉ mục cũng có thể được sử dụng để tăng tốc:

```sql
pg_study=# explain (costs off)
select doc from ts where doc_tsv @@ to_tsquery('slit:*');
                    QUERY PLAN
---------------------------------------------------
 Seq Scan on ts
   Filter: (doc_tsv @@ to_tsquery('slit:*'::text))
(2 rows)
```
*(Ghi chú: Trong ví dụ cụ thể này, planner đã chọn Seq Scan, nhưng trong các trường hợp dữ liệu lớn hơn và truy vấn phù hợp, GIN có thể sử dụng chỉ mục cho tìm kiếm tiền tố).*

### Tần suất Từ khóa

Tạo một số dữ liệu:

```sql
fts=# alter table mail_messages add column tsv tsvector;
fts=# update mail_messages set tsv = to_tsvector(body_plain);
fts=# create index on mail_messages using gin(tsv);
fts=# \timing on

-- Tổng cộng có 356125 dòng dữ liệu (đã cập nhật số lượng từ bài gốc)
fts=# select count(*) from mail_messages;
 count
--------
 356125
(1 row)

-- Ở đây, thay vì sử dụng unnest để đếm số lần một từ xuất hiện trong một dòng, vì lượng dữ liệu tương đối lớn, chúng ta sử dụng hàm ts_stat để tính toán
fts=# select word, ndoc
fts-# from ts_stat('select tsv from mail_messages')
fts-# order by ndoc desc limit 3;
 word  |  ndoc
-------+--------
 wrote | 231174
 use   | 173833
 would | 157169
(3 rows)

Time: 11556.976 ms
```

Ví dụ, chúng ta truy vấn từ hiếm khi xuất hiện trong thông tin email, chẳng hạn như "tattoo":

```sql
fts=# select word, ndoc from ts_stat('select tsv from mail_messages') where word = 'tattoo';
  word  | ndoc
--------+------
 tattoo |    2
(1 row)

Time: 11236.340 ms
```

Số lần hai từ xuất hiện cùng nhau trong cùng một dòng. Chỉ có một dòng mà "wrote" và "tattoo" xuất hiện cùng lúc:

```sql
fts=# select count(*) from mail_messages where tsv @@ to_tsquery('wrote & tattoo');
 count
-------
     1
(1 row)

Time: 0.550 ms
```

Hãy xem nó được thực thi như thế nào. Như đã đề cập ở trên, nếu chúng ta muốn lấy danh sách TID của hai từ bằng cách duyệt qua, hiệu quả tìm kiếm rõ ràng rất thấp: vì chúng ta phải duyệt qua hơn 200.000 giá trị, và chỉ một giá trị được lấy. Tuy nhiên, thông qua thông tin thống kê (`ts_stat`), thuật toán có thể biết rằng "wrote" xuất hiện thường xuyên, trong khi "tattoo" xuất hiện hiếm khi. Do đó, tìm kiếm từ ít được sử dụng ("tattoo") sẽ được thực hiện trước, và sau đó nó sẽ kiểm tra xem "wrote" có tồn tại trong hai dòng được truy xuất đó hay không. Bằng cách này, kết quả truy vấn có thể được lấy nhanh chóng:

```sql
fts=# select count(*) from mail_messages where tsv @@ to_tsquery('wrote & tattoo');
 count
-------
     1
(1 row)

Time: 0.419 ms
```
*(Ghi chú: Thời gian thực thi có thể thay đổi tùy thuộc vào hệ thống và dữ liệu cụ thể).*

### Giới hạn Kết quả Truy vấn

Một đặc điểm của chỉ mục GIN là nó chỉ có thể trả về một bitmap chứ không thể trả về từng TID một. Do đó, tất cả các kế hoạch truy vấn trong bài viết này đều sử dụng bitmap scan.

### Phương pháp Nén

Một trong những ưu điểm của GIN là tính năng nén của nó. Thứ nhất, nếu cùng một từ xuất hiện trong nhiều dòng, nó sẽ chỉ được lưu trữ một lần trong chỉ mục. Thứ hai, các TID được lưu trữ theo thứ tự trong chỉ mục, điều này cho phép chúng ta sử dụng một phương pháp nén đơn giản: TID tiếp theo trong danh sách thực sự khác với TID trước đó; số này thường rất nhỏ, và so với TID sáu byte hoàn chỉnh, số bit yêu cầu ít hơn nhiều.

So sánh kích thước của các chỉ mục khác nhau:

Tạo một chỉ mục B-tree: GIN được xây dựng trên một kiểu dữ liệu khác (tsvector thay vì text), và kiểu dữ liệu này nhỏ hơn. Đồng thời, B-tree sẽ cắt bớt thông điệp đến trong vòng 2K.

```sql
fts=# create index mail_messages_btree on mail_messages(substring(body_plain for 2048));
CREATE INDEX
Time: 32709.807 ms
```

Tạo một chỉ mục GiST:

```sql
fts=# create index mail_messages_gist on mail_messages using gist(tsv);
CREATE INDEX
Time: 14651.884 ms
```

Xem kích thước của gin, gist và btree lần lượt:

```sql
fts=# select pg_size_pretty(pg_relation_size('mail_messages_tsv_idx')) as gin,
fts-#              pg_size_pretty(pg_relation_size('mail_messages_gist')) as gist,
fts-#              pg_size_pretty(pg_relation_size('mail_messages_btree')) as btree;
  gin   |  gist  | btree
--------+--------+--------
 189 MB | 111 MB | 526 MB
(1 row)

Time: 2.961 ms
```

Vì chỉ mục GIN tiết kiệm không gian hơn (trong ví dụ này so với B-tree trên text), chúng ta có thể sử dụng chỉ mục GIN thay vì chỉ mục bitmap trong quá trình di chuyển từ Oracle sang PostgreSQL. Nói chung, chỉ mục bitmap được sử dụng cho các trường có rất ít giá trị duy nhất, điều này cũng rất hiệu quả đối với GIN. Hơn nữa, PostgreSQL có thể tự động xây dựng một bitmap dựa trên bất kỳ chỉ mục nào (bao gồm GIN).

## Kết luận

PostgreSQL với chỉ mục GIN cung cấp một giải pháp mạnh mẽ và hiệu quả cho việc tìm kiếm toàn văn, đặc biệt là khi kết hợp với kiểu dữ liệu `tsvector` và các hàm xử lý văn bản (`to_tsvector`, `to_tsquery`, `ts_stat`). Mặc dù có những thách thức về tốc độ cập nhật so với các công cụ tìm kiếm chuyên dụng, tính năng `fastupdate` giúp cải thiện đáng kể vấn đề này.

Với khả năng lập chỉ mục đảo ngược, tối ưu hóa truy vấn dựa trên tần suất từ khóa, hỗ trợ tìm kiếm phức tạp (AND, OR, NOT, prefix), và lợi thế về khả năng nén dữ liệu, GIN trong PostgreSQL có thể đáp ứng nhu cầu tìm kiếm toàn văn cho nhiều ứng dụng, có thể giảm thiểu hoặc loại bỏ sự cần thiết phải triển khai và quản lý một hệ thống tìm kiếm riêng biệt như Elasticsearch, đặc biệt là đối với các dự án đã sử dụng PostgreSQL làm cơ sở dữ liệu chính.

---
*(Đây là bản viết lại và tóm tắt từ bài viết gốc tiếng Anh)*

# 3 Kết hợp Tìm kiếm Ngữ nghĩa (Semantic Search) và Tìm kiếm Toàn văn (Full-Text Search) trong PostgreSQL
Sponsor by https://www.timescale.com/blog/combining-semantic-search-and-full-text-search-in-postgresql-with-cohere-pgvector-and-pgai?ref=dailydev

*(Với Cohere, Pgvector, và Pgai)*

*Bài viết gốc: James Reed, Infrastructure Engineer · Leapcell*
*Ngày: 09 tháng 5 năm 2025*
*(Viết lại từ bài viết tiếng Anh)*

*Trong trường hợp bạn đã bỏ lỡ buổi ra mắt pgai Vectorizer, hãy tìm hiểu lý do tại sao cơ sở dữ liệu vector là một trừu tượng hóa không phù hợp và cách bạn có thể tạo các vector nhúng AI ngay trong cơ sở dữ liệu PostgreSQL của mình.*

---

## Cấu trúc Tìm kiếm Lai (Hybrid Search) với Cohere, pgvector, và pgai

Công nghệ tìm kiếm là xương sống của các hệ thống truy xuất dữ liệu hiện đại. Qua nhiều năm, chúng đã phát triển từ việc chỉ đơn giản là khớp từ khóa sang các thuật toán tìm kiếm ngữ nghĩa nắm bắt ngữ cảnh và ý định người dùng.

Tuy nhiên, bạn thường gặp phải những tình huống mà một phương pháp tìm kiếm duy nhất không đủ. Hãy tưởng tượng truy vấn tìm kiếm sau: "Các nghiên cứu gần đây về tác động của biến đổi khí hậu đối với động vật hoang dã Bắc Cực." Phương pháp khớp từ khóa sẽ truy xuất các tài liệu chứa các từ cụ thể như "biến đổi khí hậu" và "động vật hoang dã Bắc Cực", trong khi tìm kiếm ngữ nghĩa có thể giúp xác định các nghiên cứu liên quan ngay cả khi không có các từ khóa chính xác, chẳng hạn như các bài viết về "ảnh hưởng của biến đổi khí hậu đối với gấu Bắc Cực".

Do đó, phương pháp mới là kết hợp tìm kiếm toàn văn (full-text search) để truy xuất chính xác với tìm kiếm ngữ nghĩa (semantic search) để có độ liên quan cao hơn. Mô hình này được gọi là tìm kiếm lai (hybrid search), và nó mang lại những ưu điểm tốt nhất của cả hai phương pháp. Trong bài viết này, chúng ta sẽ giải thích cách triển khai tìm kiếm lai trong PostgreSQL bằng cách sử dụng Cohere, pgvector và pgai (cùng với một số đổi mới khác từ Timescale). Điều này sẽ cho phép bạn xây dựng một hệ thống tìm kiếm nhanh, liên quan và chính xác bằng cách sử dụng cơ sở dữ liệu PostgreSQL hiện có của mình.

## 01. Hiểu về Tìm kiếm Toàn văn và Tìm kiếm Ngữ nghĩa

Trước khi đi sâu vào triển khai, hãy xem xét sự khác biệt giữa tìm kiếm toàn văn và tìm kiếm ngữ nghĩa.

### Tìm kiếm Toàn văn (Full-text search)

Tìm kiếm toàn văn, hay tìm kiếm từ khóa, tìm các kết quả khớp chính xác với các từ khóa trong một truy vấn. Các thuật toán tìm kiếm toàn văn lập chỉ mục các tài liệu văn bản bằng cách chia chúng thành các thuật ngữ (terms) và tạo ra một chỉ mục đảo ngược (inverted index) ánh xạ các thuật ngữ đến vị trí của chúng trong các tài liệu. Trong quá trình tìm kiếm, thuật toán truy xuất các tài liệu liên quan bằng cách khớp các thuật ngữ trong truy vấn với chỉ mục và xếp hạng kết quả bằng các thuật toán như TF-IDF hoặc BM25.

Những ngành công nghiệp nào sử dụng tìm kiếm toàn văn? Các trang thương mại điện tử sử dụng nó để giúp khách hàng tìm sản phẩm; các ứng dụng thị trường chứng khoán sử dụng nó để giúp người dùng tìm ký hiệu mã chứng khoán; các trang web tin tức sử dụng nó để hiển thị các bài viết liên quan; các hệ thống chăm sóc sức khỏe sử dụng nó để tìm hồ sơ bệnh nhân.

Mặc dù có tính hữu ích, tìm kiếm toàn văn có một số hạn chế nhất định. Vì nó không nắm bắt được ý định người dùng, nó thường dẫn đến kết quả không liên quan. Ví dụ, một tìm kiếm toàn văn với truy vấn “apple phone” có thể truy xuất kết quả về quả táo (apple - trái cây) và thậm chí cả điện thoại Android.

*Ảnh minh họa: Minh họa đơn giản về điểm TF-IDF, tính toán mức độ liên quan của một từ đối với văn bản tài liệu.*

### Tìm kiếm Ngữ nghĩa (Semantic search)

Tìm kiếm ngữ nghĩa vượt ra ngoài việc chỉ khớp từ khóa; nó hiểu nghĩa của từ và cách chúng liên quan đến nhau thông qua các vector. Vector là các biểu diễn số được tạo ra bởi các mô hình AI, nắm bắt ý nghĩa ngữ cảnh và mối quan hệ giữa các từ. Điều này cho phép tìm kiếm ngữ nghĩa tìm thấy kết quả liên quan ngay cả khi không có các từ khóa chính xác.

Tìm kiếm ngữ nghĩa có ứng dụng trong nhiều lĩnh vực. Bạn có thể sử dụng nó để xây dựng hệ thống tìm kiếm và đề xuất sản phẩm bằng cách tìm các sản phẩm liên quan dựa trên truy vấn người dùng. Bạn có thể tận dụng nó cho bộ phận hỗ trợ khách hàng bằng cách sử dụng nó để diễn giải chính xác truy vấn người dùng. Trong chăm sóc sức khỏe, bạn có thể sử dụng nó để khớp truy vấn bệnh nhân với thông tin y tế liên quan.

Trong nhiều kịch bản, tìm kiếm ngữ nghĩa là lựa chọn tốt hơn so với tìm kiếm từ khóa. Vì nó có thể nắm bắt ý nghĩa đằng sau các truy vấn, nó có thể hiểu các thuật ngữ liên quan hoặc thậm chí các khái niệm liên quan trên các ngôn ngữ khác nhau. Điều này thường dẫn đến kết quả tìm kiếm liên quan hơn.

*Ảnh minh họa: Sơ đồ minh họa việc chuyển đổi tài liệu thành vector nhúng và tìm kiếm ngữ nghĩa trong không gian vector đa chiều.*

## 02. Giới thiệu Tìm kiếm Lai (Hybrid Search)

Nếu tìm kiếm ngữ nghĩa hoạt động tốt hơn tìm kiếm toàn văn trong việc hiểu truy vấn người dùng, tại sao bạn lại cần phương pháp thứ hai? Hầu hết các lĩnh vực có từ vựng, viết tắt, chú thích, ký hiệu hoặc tên gọi cụ thể mà các mô hình ngữ nghĩa có thể không nắm bắt được. Tìm kiếm từ khóa đảm bảo rằng các thuật ngữ cụ thể này không bị bỏ sót.

Vì vậy, bất cứ khi nào bạn cần kết hợp các kết quả khớp chính xác với các kết quả liên quan theo ngữ cảnh, sử dụng tìm kiếm lai sẽ mang lại kết quả tốt hơn.

Hãy lấy một ví dụ từ lĩnh vực tài chính. Sử dụng tìm kiếm lai, một truy vấn như “tin tức và dự đoán AAPL” sẽ truy xuất các tài liệu chứa thuật ngữ “AAPL”, cùng với các tài liệu khác liên quan đến tin tức thị trường chứng khoán của Apple.

Tìm kiếm lai cho phép bạn kết hợp độ chính xác của tìm kiếm từ khóa với khả năng hiểu ngữ cảnh của tìm kiếm vector. Điều này đảm bảo kết quả vừa chính xác vừa liên quan theo ngữ cảnh. Nó cũng thích ứng với nhiều loại truy vấn khác nhau, từ các thuật ngữ kỹ thuật cụ thể đến các tìm kiếm rộng hơn.

### Các trường hợp sử dụng của tìm kiếm lai

Do tính linh hoạt của nó, tìm kiếm lai có thể được sử dụng trong nhiều ngành công nghiệp:

*   **Thương mại điện tử:** Giúp khách hàng tìm sản phẩm bằng cả các thuật ngữ cụ thể và mô tả chung, cải thiện khả năng khám phá sản phẩm.
*   **Nghiên cứu pháp lý:** Cho phép luật sư tìm các vụ án liên quan bằng cả thuật ngữ pháp lý chính xác và các tiền lệ tương tự về mặt khái niệm.
*   **Xem xét tài liệu khoa học:** Hỗ trợ các nhà nghiên cứu tìm các bài báo có thuật ngữ kỹ thuật cụ thể đồng thời khám phá các công trình có chủ đề liên quan.
*   **Các công cụ tìm kiếm:** Một ví dụ tuyệt vời là nghiên cứu điển hình của Stack Overflow. Tìm kiếm lai mới kết hợp sự liên quan về ngữ nghĩa với việc khớp mã chính xác, cho phép nó tìm nội dung liên quan về mặt khái niệm đồng thời đáp ứng các đoạn mã cụ thể do người dùng nhập.
*   **Hỗ trợ khách hàng:** Nâng cao khả năng hiểu và phản hồi của hệ thống hỗ trợ đối với nhiều loại yêu cầu của khách hàng.

## 03. Triển khai Tìm kiếm Lai với Cohere, Pgvector, và Pgai

*Ảnh minh họa: Sơ đồ các bước triển khai tìm kiếm lai.*

Hãy cùng xem các bước để triển khai tìm kiếm lai. Chúng ta sẽ sử dụng mô hình nhúng Cohere `embed-english-v3.0` và tận dụng các extension `pgvector` và `pgai` để đưa khả năng AI vào cơ sở dữ liệu PostgreSQL.

*   **`embed-english-v3.0`:** Một mô hình nhúng văn bản từ Cohere được thiết kế để tạo ra các vector nhúng chất lượng cao từ văn bản.
*   **`pgai`:** Cho phép tích hợp với các mô hình AI và cho phép bạn thực hiện các truy vấn hỗ trợ AI ngay trong cơ sở dữ liệu PostgreSQL. Mặc dù `pgai` cho phép bạn thực hiện các hoạt động AI phức tạp (bao gồm tạo vector nhúng văn bản), với `pgai Vectorizer`, bạn còn có thể tự động hóa quá trình tạo và quản lý các vector nhúng mà không cần rời khỏi môi trường cơ sở dữ liệu PostgreSQL của mình.
*   **`pgvector`:** Cho phép bạn lưu trữ và truy vấn vector nhúng, kích hoạt các tìm kiếm ngữ nghĩa hiệu quả trên dữ liệu của bạn. Trong hướng dẫn này, chúng ta sử dụng `pgvectorscale`, extension này bổ sung hiệu suất truy vấn và khả năng mở rộng cho chức năng của `pgvector`, giúp tăng tốc cơ sở dữ liệu PostgreSQL của bạn để xử lý các tập dữ liệu lớn một cách dễ dàng.

### Chuẩn bị

Trước khi tiếp tục, bạn sẽ cần một khóa API từ Cohere. Lưu khóa API dưới dạng biến môi trường.

Bạn cũng sẽ cần một thiết lập PostgreSQL hoạt động cùng với các extension `pgai` và `pgvector`. Bạn có thể cài đặt thủ công, sử dụng Docker container dựng sẵn hoặc đơn giản là sử dụng Timescale Cloud, nơi đã cài đặt sẵn `pgai` và `pgvector`.

Cài đặt các thư viện cần thiết...

```bash
!pip install pandas pyarrow pgvector cohere psycopg2-binary
```

...và sau đó import chúng.

```python
import pandas as pd
import cohere
import psycopg2
import os
import json # Cần thêm thư viện json cho phần reranking
from ast import literal_eval
```

Tiếp theo, khởi tạo và thiết lập kết nối cơ sở dữ liệu PostgreSQL.

```python
conn = psycopg2.connect("postgres://<username>:<password>@<host>:<port>/<database>")
```

Chúng ta cũng cần bật các extension.

```sql
with conn:
    with conn.cursor() as cursor:
        cursor.execute("CREATE EXTENSION IF NOT EXISTS vectorscale CASCADE;")
        cursor.execute("CREATE EXTENSION IF NOT EXISTS ai CASCADE;")
```

Để sử dụng Cohere trong psql, bạn nên làm như sau:

```bash
export COHERE_API_KEY=YOUR_API_KEY
PGOPTIONS="-c ai.cohere_api_key=$COHERE_API_KEY"
psql -d "postgres://postgres:password@localhost/postgres"
```

### Tải xuống tập dữ liệu

Chúng ta đang sử dụng tập dữ liệu Cohere/Movies từ Hugging Face. Tải xuống tập dữ liệu và tạo một tệp CSV cho thuận tiện, sử dụng một trăm dòng đầu tiên từ tập dữ liệu.

```python
# Tải tập dữ liệu
df = pd.read_parquet("hf://datasets/Cohere/movies/movies.parquet")
df = df.head(100)
df.to_csv('movies.csv', index=False)
```

Tập dữ liệu có các trường sau cho mỗi 'bộ phim':

| Trường    | Mô tả                                                      |
| :-------- | :--------------------------------------------------------- |
| `title`   | Một chuỗi chứa tiêu đề của bộ phim                         |
| `overview`| Một chuỗi chứa mô tả tóm tắt về bộ phim                  |
| `genres`  | Một chuỗi chứa danh sách thể loại (ngăn cách bằng dấu phẩy)|
| `producer`| Một chuỗi chứa danh sách nhà sản xuất (ngăn cách bằng dấu phẩy)|
| `cast`    | Một chuỗi chứa danh sách diễn viên (ngăn cách bằng dấu phẩy)|

Đối với ví dụ tìm kiếm lai của chúng ta, chúng ta sẽ sử dụng kết hợp các trường `title` và `overview`.

### Tạo bảng

Bây giờ chúng ta sẽ tạo bảng, điền dữ liệu vào đó, và sau đó tạo các vector nhúng bằng hàm `cohere_embed` và mô hình nhúng `embed-english-v3.0`.

Hai bước đầu tiên rất đơn giản: chúng ta sẽ sử dụng các truy vấn SQL để tạo bảng “movies” với một cột `embedding` để lưu trữ các vector nhúng. Sau đó, chúng ta điền dữ liệu vào bảng từ tệp CSV của mình.

Cuối cùng, chúng ta tạo các vector nhúng. Để làm điều này, chúng ta sẽ sử dụng hàm `pgai` `cohere_embed`. Chỉ với một truy vấn SQL, `pgai` cho phép chúng ta tạo các vector nhúng và lưu trữ chúng trong cột `embedding`.

```python
COHERE_API_KEY = os.getenv("COHERE_API_KEY")

def ingest_data(conn):
    with conn:
        with conn.cursor() as cur:
            # Tạo bảng
            cur.execute("""
            CREATE TABLE IF NOT EXISTS movies (
                title TEXT NOT NULL,
                overview TEXT,
                genres TEXT,
                producer TEXT,
                "cast" TEXT,
                embedding VECTOR(1024)
            )
            """)

            # Sao chép dữ liệu từ tệp CSV
            with open('movies.csv', 'r') as f:
                cur.copy_expert('COPY movies(title, overview, genres, producer, "cast") FROM STDIN WITH CSV HEADER', f)

            # Cập nhật vector nhúng
            with conn.cursor() as cursor:
                # Sử dụng cohere_embed để tạo vector nhúng
                cursor.execute("""
                UPDATE movies
                SET embedding = ai.cohere_embed(
                    'embed-english-v3.0'
                    , CONCAT_WS('. ',
                        title,
                        COALESCE(overview, '')
                    )
                    , input_type=>'search_document'
                    , api_key=>%s
                ) where embedding is null;
                """, (COHERE_API_KEY, ))

ingest_data(conn)
```

Tại thời điểm này, bạn nên có một bảng dữ liệu với các vector nhúng được tạo từ các cột `title` và `overview`.

## 04. Triển khai Tìm kiếm Từ khóa (Keyword Search)

Tiếp theo, chúng ta sẽ triển khai tìm kiếm từ khóa.

Các toán tử tìm kiếm truyền thống như `~`, `~*`, `LIKE`, và `ILIKE` rất đơn giản: chúng không thể xử lý các từ phái sinh, không xếp hạng kết quả và thiếu hỗ trợ chỉ mục. Để khắc phục những hạn chế này, PostgreSQL đã giới thiệu các kiểu dữ liệu `tsquery` và `tsvector` cho khả năng tìm kiếm văn bản nâng cao hơn. Chúng ta sẽ sử dụng những kiểu này để thực hiện tìm kiếm từ khóa của mình:

```python
def keyword_search(conn, query):
    sql = """
    SELECT title, overview
    FROM movies, plainto_tsquery('english', %s) query
    WHERE to_tsvector('english', title || ' ' || COALESCE(overview, '')) @@ query
    ORDER BY ts_rank_cd(to_tsvector('english', title || ' ' || COALESCE(overview, '')), query) DESC
    LIMIT 5
    """

    with conn:
        with conn.cursor() as cur:
            cur.execute(sql, (query,))
            return cur.fetchall()
```

Dưới đây là phân tích chi tiết câu lệnh SQL:

*   `plainto_tsquery` chuyển văn bản thuần túy thành một `tsquery` (truy vấn tìm kiếm văn bản).
*   `to_tsvector('english', ...)` chuyển tiêu đề và mô tả được nối lại thành một `tsvector` (vector tìm kiếm văn bản).
*   `@@` là toán tử khớp tìm kiếm văn bản.
*   `ts_rank_cd()` là một hàm xếp hạng các kết quả khớp và sắp xếp kết quả theo thứ tự giảm dần.

Bạn có thể kiểm tra hàm bằng cách truyền một truy vấn.

```python
query = "The Avengers"
results = keyword_search(conn, query)

# In results
# for r in results:
#     print(r)
```

Kết quả sẽ chứa các kết quả khớp từ khóa chính xác từ bảng “movies”.

```
[('Avengers: Age of Ultron', 'When Tony Stark tries to jumpstart a dormant peacekeeping program, things go awry and Earth\'s Mightiest Heroes are put to the ultimate test as the fate of the planet hangs in the balance. As the villainous Ultron emerges, it is up to The Avengers to stop him from enacting his terrible plans, and soon uneasy alliances and unexpected action pave the way for an epic and unique global adventure.'), ('The Avengers', 'When an unexpected enemy emerges and threatens global safety and security, Nick Fury, director of the international peacekeeping agency known as S.H.I.E.L.D., finds himself in need of a team to pull the world back from the brink of disaster. Spanning the globe, a daring recruitment effort begins!'), ('Captain America: Civil War', 'Following the events of Age of Ultron, the collective governments of the world pass an act designed to regulate all superhuman activity. This polarizes opinion amongst the Avengers, causing two factions to side with Iron Man or Captain America, which causes an epic battle between former allies.'), ('Captain America: The Winter Soldier', 'After the cataclysmic events in New York with The Avengers, Steve Rogers, aka Captain America is living quietly in Washington, D.C. and trying to adjust to the modern world. But when a S.H.I.E.L.D. colleague comes under attack, Steve becomes embroiled in a web of intrigue that threatens to put the world at risk. Joining forces with the Black Widow, Captain America struggles to expose the ever-widening conspiracy while fighting off professional assassins sent to silence him at every turn. When the full scope of the villainous plot is revealed, Captain America and the Black Widow enlist the help of a new ally, the Falcon. However, they soon find themselves up against an unexpected and formidable enemy--the Winter Soldier.')]
```

## 05. Triển khai Tìm kiếm Ngữ nghĩa (Semantic Search)

Tiếp theo, hãy xây dựng khả năng tìm kiếm ngữ nghĩa. Trong tìm kiếm ngữ nghĩa, chúng ta trước tiên chuyển truy vấn thành một vector nhúng bằng cách sử dụng hàm `cohere_embed` và mô hình nhúng tương tự mà chúng ta đã sử dụng trước đó. Sau đó, chúng ta sử dụng một độ đo khoảng cách (hoặc hàm khoảng cách) để tìm các vector nhúng từ bảng “movies” gần nhất với vector nhúng của truy vấn. Khoảng cách giữa một vector và vector truy vấn của chúng ta càng nhỏ, độ tương đồng càng cao.

`pgvector` hỗ trợ một số độ đo khoảng cách:

| Toán tử | Độ đo Khoảng cách         |
| :------ | :------------------------ |
| `<->`   | Khoảng cách L2            |
| `<=>`   | Độ tương đồng Cosine      |
| `<#>`   | Tích trong Âm (Negative Inner Product) |
| `<+>`   | Khoảng cách L1            |

Chúng ta sẽ sử dụng tích trong âm (`<#>`) trong truy vấn SQL của mình, được biểu diễn bởi toán tử `<=>` (lưu ý: `<=>` trong `pgvector` thực sự biểu diễn Cosine Distance, tức là 1 trừ Cosine Similarity, và sắp xếp theo `<=>` tăng dần tương đương với sắp xếp theo Cosine Similarity giảm dần. Tích trong âm `<#>` thì sắp xếp theo `<#>` tăng dần tương đương với sắp xếp theo Inner Product giảm dần). Ví dụ này dùng `<=>`.

```python
# Hàm tìm kiếm ngữ nghĩa
def semantic_search(conn, query):
    sql = """
    WITH query_embedding AS (
        SELECT ai.cohere_embed(
            'embed-english-v3.0'
            , %s
            , input_type=>'search_query' -- Đã sửa lại tên tham số cho đúng với API
            , api_key=>%s             -- Đã sửa lại tên tham số cho đúng với API
        ) AS embedding
    )
    SELECT title, overview
    FROM movies, query_embedding
    ORDER BY movies.embedding <=> query_embedding.embedding
    LIMIT 5
    """

    with conn:
        with conn.cursor() as cur:
            cur.execute(sql, (query, COHERE_API_KEY,))
            return cur.fetchall()
```

Bạn có thể kiểm tra hàm tìm kiếm ngữ nghĩa bằng cách truyền một truy vấn.

```python
query = "Exciting space adventure movies"
results = semantic_search(conn, query)

# In results
# for r in results:
#     print(r)
```

Kết quả:

```
[('Interstellar', 'Interstellar chronicles the adventures of a group of explorers who make use of a newly discovered wormhole to surpass the limitations on human space travel and conquer the vast distances involved in an interstellar voyage.'), ('Transformers: Dark of the Moon', "Sam Witwicky takes his first tenuous steps into adulthood while remaining a reluctant human ally of Autobot-leader Optimus Prime. The film centers around the space race between the USSR and the USA, suggesting there was a hidden Transformers role in it all that remains one of the planet's most dangerous secrets."), ('Star Wars', 'Princess Leia is captured and held hostage by the evil Imperial forces in their effort to take over the galactic Empire. Venturesome Luke Skywalker and dashing captain Han Solo team together with the loveable robot duo R2-D2 and C-3PO to rescue the beautiful princess and restore peace and justice in the Empire.'), ('Avatar', 'In the 22nd century, a paraplegic Marine is dispatched to the moon Pandora on a unique mission, but becomes torn between following orders and protecting an alien civilization.'), ('The Martian', 'During a manned mission to Mars, Astronaut Mark Watney is presumed dead after a fierce storm and left behind by his crew. But Watney has survived and finds himself stranded and alone on the hostile planet. With only meager supplies, he must draw upon his ingenuity, wit and spirit to subsist and find a way to signal to Earth that he is alive.')]
```

Tiếp theo, hãy kết hợp tìm kiếm từ khóa và tìm kiếm ngữ nghĩa để tạo hàm tìm kiếm lai của chúng ta.

## 06. Triển khai Tìm kiếm Lai (Hybrid Search)

Chúng ta sẽ kết hợp kết quả của cả hai hàm tìm kiếm để triển khai tìm kiếm lai. Để sắp xếp kết quả theo độ liên quan, chúng ta sẽ sử dụng mô hình reranker của Cohere, `rerank-english-v3.0`.

Các mô hình Reranker sắp xếp lại các kết quả bằng cách tính toán điểm liên quan giữa truy vấn và các đối tượng dữ liệu, sau đó sắp xếp chúng từ liên quan nhất đến ít liên quan nhất. Vì việc reranking tốn kém về mặt tính toán, chúng ta thường sử dụng nó ở giai đoạn cuối cùng sau khi tìm kiếm lai đã truy xuất một tập hợp kết quả ban đầu.

```python
def hybrid_search(conn, query):
    sem_results = semantic_search(conn, query)
    key_results = keyword_search(conn, query)

    # Kết hợp kết quả và loại bỏ trùng lặp
    combined_results = list({(title, overview) for title, overview in sem_results + key_results})

    # Chuẩn bị tài liệu cho việc reranking
    documents = [f"{title}. {overview}" for title, overview in combined_results]
    # Chuẩn bị mảng JSON của các tài liệu
    documents_json = json.dumps([{"text": doc} for doc in documents])

    # Truy vấn SQL để reranking sử dụng pgai
    sql = """
    SELECT
        x."index",
        x.document->>'text' as "text",
        x.relevance_score
    FROM jsonb_to_recordset(
        ai.cohere_rerank(
            'rerank-english-v3.0',
            %s,
            %s::jsonb,
            return_documents => true,
            api_key=>%s
        )->'results'
    ) AS x("index" int, "document" jsonb, relevance_score float8)
    ORDER BY relevance_score DESC
    LIMIT 5;
    """

    reranked_results = []
    with conn.cursor() as cur:
        cur.execute(sql, (query, documents_json, COHERE_API_KEY,))
        reranked_results = cur.fetchall()

    # Ánh xạ kết quả đã rerank trở lại dữ liệu gốc
    final_results = []
    for result in reranked_results:
        # Tìm index của tài liệu trong danh sách ban đầu
        try:
            index = documents.index(result[1])
            final_results.append(combined_results[index])
        except ValueError:
            # Xử lý trường hợp tài liệu không tìm thấy (ví dụ: có lỗi trong quá trình)
            print(f"Warning: Document text from reranking not found in combined_results: {result[1][:50]}...")
            pass


    return final_results
```

Hàm `cohere_rerank` sử dụng một mô hình ngôn ngữ để đánh giá độ liên quan của mỗi tài liệu đối với truy vấn và sau đó sắp xếp lại các tài liệu dựa trên điểm liên quan của chúng.

### Kết quả

Bây giờ chúng ta có thể sử dụng hàm tìm kiếm lai của mình để truy xuất các tài liệu tận dụng cả khớp từ khóa và tìm kiếm ngữ nghĩa.

```python
query = "Space adventure movies with advanced alien technology"
results = hybrid_search(conn, query)

# In results
# for r in results:
#     print(r)
```

Kết quả:

```
[('Interstellar', 'Interstellar chronicles the adventures of a group of explorers who make use of a newly discovered wormhole to surpass the limitations on human space travel and conquer the vast distances involved in an interstellar voyage.'), ('Avatar', 'In the 22nd century, a paraplegic Marine is dispatched to the moon Pandora on a unique mission, but becomes torn between following orders and protecting an alien civilization.'), ('Transformers: Dark of the Moon', "Sam Witwicky takes his first tenuous steps into adulthood while remaining a reluctant human ally of Autobot-leader Optimus Prime. The film centers around the space race between the USSR and the USA, suggesting there was a hidden Transformers role in it all that remains one of the planet's most dangerous secrets."), ('Indiana Jones and the Kingdom of the Crystal Skull', "Set during the Cold War, the Soviets - led by sword-wielding Irina Spalko - are in search of a crystal skull which has supernatural powers related to a mystical Lost City of Gold. After being captured and then escaping from them, Indy is coerced to head to Peru at the behest of a young man whose friend - and Indy's colleague - Professor Oxley has been captured for his knowledge of the skull's whereabouts."), ('Mission: Impossible - Ghost Protocol', 'In the 4th installment of the Mission Impossible series, Ethan Hunt (Cruise) and his team are racing against time to track down a dangerous terrorist named Hendricks (Nyqvist), who has gained access to Russian nuclear launch codes and is planning a strike on the United States. An attempt to stop him ends in an explosion causing severe destruction to the Kremlin and the IMF to be implicated in the bombing, forcing the President to disavow them. No longer being aided by the government, Ethan and his team chase Hendricks around the globe, although they might still be too late to stop a disaster.')]
```

Điều này hoàn thành việc triển khai tìm kiếm lai của chúng ta trong PostgreSQL. Bạn có thể sử dụng phương pháp này để xây dựng các hệ thống tìm kiếm, Generation Tăng cường Truy xuất (RAG) hoặc các ứng dụng khác cần sử dụng truy xuất chính xác.

## 07. Các Thực hành Tốt nhất cho Tìm kiếm Lai

Trong bản demo trên, chúng ta đã triển khai tìm kiếm lai trong PostgreSQL. Đối với trường hợp sử dụng cụ thể của bạn, bạn có thể muốn tối ưu hóa phương pháp tiếp cận thêm bằng cách sử dụng một trong các phương pháp sau:

*   **Cân bằng các phương pháp tìm kiếm:** Tìm sự cân bằng phù hợp giữa tìm kiếm ngữ nghĩa và tìm kiếm từ khóa. Tùy thuộc vào dữ liệu và trường hợp sử dụng của bạn, bạn có thể cần trọng số một phương pháp cao hơn phương pháp kia. Thử nghiệm với các kết hợp khác nhau để tìm sự kết hợp tối ưu.
*   **Caching và tiền xử lý:** Triển khai các cơ chế caching cho các truy vấn được tìm kiếm thường xuyên hoặc dữ liệu được truy cập phổ biến. Tiền xử lý và lưu trữ các vector nhúng cho tìm kiếm ngữ nghĩa để giảm chi phí tính toán thời gian thực.
*   **Đánh giá và tinh chỉnh:** Thường xuyên đánh giá hiệu suất của hệ thống tìm kiếm lai của bạn. Sử dụng các chỉ số như độ chính xác (precision), độ thu hồi (recall), và MAP (mean average precision) để đánh giá chất lượng tìm kiếm và thực hiện các điều chỉnh cần thiết.
*   **Tận dụng pgvectorscale:** Đối với các tập dữ liệu lớn hơn, hãy cân nhắc sử dụng `pgvectorscale`, một extension của PostgreSQL tận dụng chỉ mục StreamingDiskANN và Định lượng nhị phân thống kê (Statistical Binary Quantization) để tăng tốc tìm kiếm tương đồng vector.



# Tối Ưu Hiệu Suất INSERT PostgreSQL Gấp Đôi với Kỹ Thuật UNNEST

Khi làm việc với PostgreSQL, việc chèn hàng loạt dữ liệu (bulk insert) có thể gặp vấn đề về hiệu suất. Bài viết này giới thiệu một kỹ thuật mạnh mẽ sử dụng hàm `UNNEST` để tăng tốc độ `INSERT` lên gấp đôi so với phương pháp truyền thống.

## Vấn Đề Với INSERT Truyền Thống

Phương pháp `INSERT ... VALUES` là phổ biến nhưng hiệu suất sẽ giảm đáng kể khi số lượng bản ghi (batch size) tăng lên hàng nghìn. PostgreSQL phải xử lý từng giá trị riêng lẻ, dẫn đến thời gian *planning* kéo dài.

```sql
INSERT INTO products (name, price, category_id)
VALUES
('iPhone 15', 999.99, 1),
('Samsung Galaxy S24', 899.99, 1);
```

## Giải Pháp: Sử Dụng Hàm UNNEST

Thay vì cung cấp từng giá trị, chúng ta có thể nhóm dữ liệu thành các mảng (arrays) và sử dụng hàm `UNNEST` để "mở rộng" chúng thành các hàng.

```sql
INSERT INTO products (name, price, category_id)
SELECT * FROM unnest(
    ARRAY['iPhone 15', 'Samsung Galaxy S24'],
    ARRAY[999.99, 899.99],
    ARRAY[1, 1]
);
```

## Tại Sao UNNEST Nhanh Hơn?

Sự khác biệt về hiệu suất chủ yếu đến từ *thời gian planning*:

*   **INSERT VALUES:** PostgreSQL cần phân tích và lập kế hoạch cho hàng nghìn tham số riêng lẻ (ví dụ: 1000 bản ghi với 3 cột = 3000 tham số).
*   **INSERT UNNEST:** Chỉ cần xử lý số lượng mảng tương ứng với số cột (ví dụ: 3 mảng), giảm đáng kể chi phí lập kế hoạch.

Thời gian thực thi có thể tương đương hoặc `UNNEST` hơi chậm hơn một chút do việc "bung" mảng, nhưng lợi ích từ việc tiết kiệm thời gian *planning* vượt trội.

## Ví Dụ Thực Tế

Với bảng `system_logs`:

```sql
CREATE TABLE system_logs (
    log_id SERIAL PRIMARY KEY,
    timestamp TIMESTAMPTZ,
    level TEXT,
    message TEXT,
    user_id INTEGER
);
```

**Với UNNEST:**

```sql
INSERT INTO system_logs (timestamp, level, message, user_id)
SELECT * FROM unnest(
    ARRAY['2024-01-01 10:00:00'::timestamptz, '2024-01-01 10:01:15'::timestamptz, '2024-01-01 10:02:30'::timestamptz]::timestamptz[],
    ARRAY['INFO', 'WARNING', 'ERROR']::text[],
    ARRAY['User login successful', 'Failed login attempt', 'Database connection timeout']::text[],
    ARRAY[1001, 1002, NULL]::integer[]
);
```

## Kết Quả Benchmark

Trong các thử nghiệm với 1 triệu bản ghi:

*   **Batch size 1,000:** `UNNEST` nhanh hơn **2.13 lần**.
*   **Batch size 5,000 & 10,000:** Tỷ lệ này được duy trì ổn định.
*   Lợi ích của `UNNEST` càng rõ rệt hơn với các bảng có nhiều cột.

## Khi Nào Nên Sử Dụng UNNEST?

**Nên sử dụng khi:**

*   Thực hiện batch insert với số lượng bản ghi lớn (ví dụ: >100 records).
*   Cần các tính năng của `INSERT` (như `ON CONFLICT`).
*   Không thể sử dụng `COPY` command.
*   Hiệu suất là ưu tiên hàng đầu.

**Cân nhắc:**

*   Mã nguồn có thể phức tạp và khó đọc hơn một chút.
*   Yêu cầu chuyển đổi dữ liệu thành cấu trúc mảng.
*   Các ORM (Object-Relational Mappers) có thể không hỗ trợ cú pháp này một cách tự nhiên.
*   Cần đảm bảo đội ngũ phát triển hiểu rõ về kỹ thuật này.

## Lưu Ý Quan Trọng: COPY Command

`COPY command` vẫn là phương pháp nhanh nhất cho các tác vụ bulk insert thực sự lớn. Tuy nhiên, `UNNEST` là một giải pháp trung gian tuyệt vời, mang lại sự linh hoạt của `INSERT` mà vẫn đạt được hiệu suất cao đáng kể.

## Kết Luận

Kỹ thuật `UNNEST` là một công cụ mạnh mẽ, tuy ít được biết đến, có thể giúp bạn tăng gấp đôi hiệu suất `INSERT` trong PostgreSQL. Mặc dù có một số đánh đổi về độ phức tạp, đây là một kỹ thuật đáng giá trong bộ công cụ của mọi nhà phát triển PostgreSQL khi đối mặt với các vấn đề về hiệu suất batch inserts. Luôn khuyến khích thử nghiệm kỹ lưỡng với dữ liệu và môi trường thực tế của bạn để đánh giá lợi ích chính xác.


