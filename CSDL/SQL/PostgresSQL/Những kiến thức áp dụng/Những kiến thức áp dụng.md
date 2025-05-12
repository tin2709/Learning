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