
# 1 Sử dụng Supabase cho Analytics: Hướng dẫn và Giải pháp Tối ưu
Sponsor by https://www.tinybird.co/blog-posts/can-i-use-supabase-for-user-facing-analytics?ref=dailydev

Supabase là một dịch vụ quản lý Postgres phổ biến, hoàn hảo để làm backend CSDL giao dịch cho ứng dụng của bạn. Nhưng liệu nó có thể xử lý các hệ thống analytics thời gian thực, hướng tới người dùng không? Hướng dẫn này sẽ chỉ cho bạn các cách tiếp cận khác nhau để xây dựng analytics với Supabase, từ các phương pháp đơn giản ngay trong Supabase đến các giải pháp có khả năng mở rộng cao bằng cách kết hợp Supabase với các công nghệ được tối ưu hóa cho analytics.

## Bắt đầu bằng việc hiểu rõ yêu cầu phân tích của bạn

Supabase *có thể* được sử dụng cho analytics. Hoặc *không thể*. Cách bạn tiếp cận analytics với Supabase phụ thuộc vào các yếu tố như yêu cầu về độ trễ, kích thước dữ liệu, số lượng yêu cầu người dùng đồng thời và mức độ quan trọng của ứng dụng.

Là một CSDL Postgres, Supabase không thực sự được tối ưu hóa cho analytics. Nó là một CSDL hướng dòng (row-oriented), vì vậy nó không được tối ưu cho các hàm tổng hợp (aggregating functions) trên các giá trị cột. Postgres là một CSDL "một kích cỡ phù hợp với tất cả" tuyệt vời, nhưng nó sẽ gặp phải những hạn chế nhất định.

Trước khi bắt đầu, hãy xem xét các yếu tố chính sau:

*   **Khối lượng dữ liệu (Data Volume)**
    *   Bao nhiêu sự kiện/bản ghi mỗi ngày?
    *   Tốc độ tăng trưởng dự kiến của bạn là gì?
    *   Bạn cần lưu giữ dữ liệu trong bao lâu?
*   **Mẫu truy vấn (Query Patterns)**
    *   Bạn cần loại tổng hợp nào?
    *   Truy vấn của bạn phức tạp đến mức nào?
    *   Bạn có cần kết quả thời gian thực không?
*   **Lượng người dùng đồng thời (User Concurrency)**
    *   Bao nhiêu người dùng đồng thời?
    *   Tải cao điểm của bạn là gì?
    *   Độ trễ truy vấn chấp nhận được của bạn là bao nhiêu?
*   **Độ tươi mới của dữ liệu (Data Freshness)**
    *   Analytics của bạn cần thời gian thực đến mức nào?
    *   Bạn có thể chấp nhận một chút chậm trễ không?
    *   Bạn có cần cập nhật dưới giây không?

Nói một cách đơn giản: Dữ liệu càng lớn, truy vấn càng phức tạp, hoặc bạn càng đặt nhiều yêu cầu về độ trễ thấp/đồng thời cao, bạn càng có khả năng cần chạy analytics ngoài Supabase.

## Các phương pháp đơn giản để xây dựng analytics trên Supabase

Đây là những cách đơn giản nhất để tiếp cận analytics trên Supabase, tất cả đều nằm trong Supabase. Nếu bạn mới bắt đầu với một dự án nhỏ, những cách này hoàn toàn ổn để thử nghiệm một khái niệm hoặc thậm chí triển khai một tính năng nhỏ trong production.

### 1. Truy vấn trực tiếp bảng production

Đây là cách tiếp cận đơn giản nhất. Nó hoạt động tốt cho:

*   Tập dữ liệu nhỏ đến trung bình (tối đa ~500K hàng)
*   Tổng hợp và bộ lọc đơn giản
*   Lượng truy vấn thấp (< ~100 truy vấn/phút)
*   Yêu cầu không phải thời gian thực (chấp nhận được độ trễ vài phút)

Giả sử bạn có một bảng tên là `orders`:

```sql
CREATE TABLE IF NOT EXISTS orders (
    id BIGSERIAL PRIMARY KEY,
    order_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    order_date TIMESTAMP WITH TIME ZONE NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) NOT NULL,
    payment_method VARCHAR(50) NOT NULL,
    shipping_address JSONB NOT NULL,
    items JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

Đây là bảng giao dịch chính của bạn. Để lấy một số chỉ số tổng hợp:

```sql
-- Ví dụ: Truy vấn analytics cơ bản trên bảng production
SELECT
    DATE_TRUNC('day', created_at) as date,
    SUM(total_amount) as total_revenue,
    AVG(total_amount) as average_order_value
FROM orders
GROUP BY date
ORDER BY date DESC;
```

Điều này sẽ hoạt động với bảng `orders` tương đối nhỏ, nhưng có nhược điểm: nó sẽ ảnh hưởng đến CSDL production của bạn, có thể dẫn đến độ trễ truy vấn cao hoặc timeout, và không sử dụng CSDL được tối ưu hóa riêng cho analytics.

**Thử nghiệm hiệu năng (trên bảng thô, không có index):**

| Số hàng trong bảng | Độ trễ truy vấn P95 (ms) |
| :----------------- | :----------------------- |
| 1,000              | 71 ms                    |
| 10,000             | 69 ms                    |
| 50,000             | 122 ms                   |
| 100,000            | 121 ms                   |
| 200,000            | 188 ms                   |
| 500,000            | 1,988 ms                 |
| 1,000,000          | 3,427 ms                 |

Có một vách đá hiệu năng rất lớn ở 500k hàng. Với 1 triệu hàng, query plan cho thấy Postgres thực hiện parallel sequential scan toàn bộ bảng. Khi dữ liệu vượt quá giới hạn bộ nhớ, nó chuyển sang dùng đĩa, làm chậm mọi thứ đáng kể.

### 2. Kỹ thuật tối ưu hóa

Tất nhiên, bảng trên hoàn toàn chưa được tối ưu. Có nhiều cách để cải thiện hiệu năng truy vấn analytics trên bảng Supabase:

*   **`CREATE INDEX`**: Index giúp Postgres lưu trữ bảng trên đĩa hiệu quả hơn, tăng tốc các mẫu truy vấn phổ biến.
*   **`PARTITION BY RANGE`**: Phân vùng có thể tăng tốc truy vấn bằng cách phân phối các hoạt động song song trên các phần dữ liệu nhỏ hơn.
*   **`VACUUM`**: Loại bỏ các hàng đã xóa hoặc cập nhật, giải phóng không gian, giảm bớt tình trạng "phình" index.
*   **`CREATE MATERIALIZED VIEW`**: Materialized view trong Supabase có thể được sử dụng để tính toán trước các tổng hợp. Tuy nhiên, chúng phải được làm mới định kỳ (ví dụ: bằng cron job), và có sự đánh đổi giữa tốc độ làm mới và độ tươi mới của dữ liệu.
*   **Sử dụng extension**: Supabase cung cấp hơn 50 extension Postgres. Ví dụ, `timescaledb` có thể hữu ích cho dữ liệu chuỗi thời gian. (Lưu ý: `cstore_fdw`, một extension lưu trữ dạng cột, không có sẵn trên Supabase tại thời điểm viết bài gốc).

### Cảnh báo: Áp lực I/O

**Quan trọng**: Nếu bạn chạy analytics trên bảng giao dịch chính, các truy vấn analytics này sẽ tạo ra áp lực I/O có thể ảnh hưởng đến các hoạt động CRUD (quan trọng hơn) của bạn. Truy vấn analytics càng phức tạp, khả năng chặn I/O càng cao.

Thử nghiệm cho thấy khi thêm các truy vấn analytics phức tạp, các truy vấn giao dịch (SELECT, INSERT) bắt đầu thất bại. Một truy vấn analytics phức tạp nhất đã gây ra 90% thất bại cho các hoạt động đọc.

**Tóm tắt về truy vấn trực tiếp CSDL Supabase production cho analytics:**

*   **Ưu điểm:**
    *   Không cần thiết lập thêm cơ sở hạ tầng.
    *   Truy cập dữ liệu bảng thời gian thực.
    *   Dễ triển khai.
    *   Không cần đồng bộ hóa dữ liệu.
    *   Có thể tối ưu hóa bằng nhiều kỹ thuật.
*   **Nhược điểm:**
    *   Có thể ảnh hưởng đáng kể đến hiệu năng production trên các giao dịch đọc/ghi.
    *   Không đặc biệt có khả năng mở rộng do những hạn chế của PostgreSQL cho analytics.
    *   Không có sự tách biệt về mối quan tâm (separation of concerns).
    *   Có thể đạt đến giới hạn kết nối khi tải cao.
    *   Ngay cả khi sử dụng materialized views, bạn phải quản lý việc làm mới chúng.
    *   Extension lưu trữ dạng cột hàng đầu không có sẵn.

### 3. Cách tiếp cận tốt hơn: Sử dụng Read Replicas

Supabase hỗ trợ read replicas, có thể được sử dụng cho analytics. Read replica là một bản sao của CSDL production chỉ dùng để đọc. Đây là một cách tiếp cận khá phổ biến để chạy analytics trên Supabase (hoặc bất kỳ CSDL Postgres nào).

Cách này cô lập tải do truy vấn analytics gây ra vào read replica, tránh xa CSDL production của bạn. Read replicas trên Supabase có endpoint và connection pool riêng, bảo vệ CSDL chính của bạn.

## Tại sao bạn *không nên* chạy analytics (chỉ) trên Supabase

Supabase tuyệt vời cho nhiều trường hợp sử dụng, nhưng đây là những kịch bản bạn có thể cần khả năng analytics chuyên biệt hơn:

*   **Dữ liệu chuỗi thời gian khối lượng lớn (High-Volume Time-Series Data)**
    *   Hàng triệu sự kiện mỗi ngày.
    *   Cần hiệu năng truy vấn dưới giây.
    *   Tổng hợp phức tạp dựa trên thời gian.
    *   Ví dụ: Hoạt động người dùng, sự kiện web, logs, traces, v.v.
*   **Yêu cầu Analytics thời gian thực (Real-Time Analytics Requirements)**
    *   Độ tươi mới dữ liệu dưới giây.
    *   Lượng người dùng đồng thời cao (1000+).
    *   Tổng hợp thời gian thực phức tạp.
    *   Ví dụ: Dashboard trực tiếp, giám sát thời gian thực, cá nhân hóa thời gian thực, v.v.
*   **Tối ưu hóa tài nguyên (Resource Optimization)**
    *   Cần giảm chi phí CSDL.
    *   Muốn tách biệt workload giao dịch và analytics.
    *   Cần mở rộng analytics một cách độc lập.

Ngay cả khi bạn chưa có những ràng buộc này, với công nghệ phù hợp (như Tinybird), bạn có thể thiết lập cơ sở hạ tầng analytics chuyên dụng cùng với Supabase với chi phí tinh thần tối thiểu, không cần thiết lập thêm cơ sở hạ tầng, và (có lẽ) ít hoặc không tốn thêm chi phí.

## Sử dụng Tinybird cho analytics cùng với Supabase

Tinybird là một backend analytics thời gian thực cho các nhà phát triển phần mềm. Nó được xây dựng trên ClickHouse, một CSDL dạng cột được tối ưu hóa cho analytics thời gian thực trên lượng lớn dữ liệu chuỗi thời gian.

**So sánh hiệu năng (truy vấn tổng hợp tương tự trên bảng `orders` thô):**

| Số hàng trong bảng | Supabase - Độ trễ P95 (ms) | Tinybird - Độ trễ P95 (ms) |
| :----------------- | :------------------------ | :------------------------- |
| 500,000            | 1,988 ms                  | 115 ms                     |
| 1,000,000          | 3,427 ms                  | 117 ms                     |
| 2,000,000          | -                         | 118 ms                     |
| 5,000,000          | -                         | 145 ms                     |
| 10,000,000         | -                         | 188 ms                     |

Tinybird nhanh hơn Supabase nhiều bậc về analytics trên tập dữ liệu lớn mà không cần tối ưu hóa, và tốc độ của nó tương đối ổn định ngay cả khi số lượng hàng tăng lên.

### Tại sao người dùng Supabase sẽ yêu thích Tinybird

Ngoài hiệu năng, Tinybird cung cấp một backend analytics mà người dùng Supabase sẽ thấy trực quan và dễ sử dụng. Tinybird trừu tượng hóa sự phức tạp của CSDL bên dưới thành một quy trình làm việc hướng nhà phát triển, nhấn mạnh vào phát triển local-first, tính năng AI-native, tích hợp CI/CD, và triển khai nhanh chóng/không đau đớn.

### Cách tích hợp Supabase với Tinybird

1.  **Sử dụng PostgreSQL Table Function của Tinybird**
    Cách đơn giản nhất để lấy dữ liệu từ Supabase vào Tinybird là sử dụng table function PostgreSQL của Tinybird:

    ```sql
    -- Trong Tinybird
    NODE copy_supabase
    SQL >
        %
        SELECT *
        FROM
        postgresql(
            'aws-0-eu-west-1.pooler.supabase.com:6543', -- Thay bằng endpoint Supabase của bạn
            'postgres',                                 -- Tên database
            'orders',                                   -- Tên bảng
            {{tb_secret('my_pg_user')}},                -- User Postgres
            {{tb_secret('my_pg_password')}}             -- Password Postgres
        )
        WHERE created_at > (
        SELECT max(created_at)
        FROM orders_in_tinybird -- Bảng đích trong Tinybird
        )
        ORDER BY created_at ASC

    TYPE copy
    TARGET_DATASOURCE orders_in_tinybird
    COPY_SCHEDULE 0 * * * * -- Chạy mỗi giờ
    COPY_MODE append
    ```
    Copy pipes của Tinybird giữ cho bảng Tinybird của bạn đồng bộ với bảng Supabase theo lịch trình bạn xác định. Bộ lọc được đẩy xuống CSDL Postgres, giảm thiểu I/O trên Supabase.

2.  **Sử dụng Change Data Capture (CDC)**
    Nếu bạn muốn đồng bộ gần thời gian thực, hãy sử dụng phương pháp Change Data Capture (CDC). Các công cụ có thể dùng: Redpanda, Confluent, Sequin. Cách này cho phép đồng bộ gần như ngay lập tức từ Write-Ahead Log (WAL) của Supabase sang Tinybird. Tinybird nhận các sự kiện dưới dạng luồng thay đổi, bạn có thể sử dụng engine `ReplacingMergeTree` để loại bỏ trùng lặp và xây dựng trạng thái của bảng Supabase.

3.  **Truyền sự kiện trực tiếp (Direct Event Streaming)**
    Để có khả năng mở rộng tốt nhất, bạn có thể bỏ qua Supabase hoàn toàn cho dữ liệu analytics và truyền sự kiện trực tiếp đến Tinybird bằng Events API (một HTTP streaming endpoint).

    ```javascript
    // Hàm gửi sự kiện đến Tinybird
    async function sendEventsToTinybird(events) {
        try {
            const response = await fetch(`https://api.tinybird.co/v0/events?name=orders_stream`, { // Tên Data Source trong Tinybird
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${TINYBIRD_TOKEN}`, // Token Tinybird của bạn
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(events) // events là một mảng các đối tượng JSON
            });

            if (!response.ok) {
                const error = await response.text();
                throw new Error(`Tinybird API error: ${error}`);
            }
            return await response.json();
        } catch (error) {
            console.error('Error sending events to Tinybird:', error);
            throw error;
        }
    }
    ```
    Cách này hoạt động tốt khi bạn cần tách biệt workload giao dịch (ví dụ: cập nhật dữ liệu người dùng) và workload analytics (ví dụ: tổng hợp dữ liệu chuỗi thời gian). Thông thường, bạn giữ bảng giao dịch trong Supabase và đẩy dữ liệu chuỗi thời gian vào Tinybird. Bạn có thể kết hợp các phương pháp, ví dụ: đồng bộ bảng quan hệ vào Tinybird và truyền sự kiện. Sau đó, khi truy vấn trong Tinybird, bạn có thể làm phong phú sự kiện bằng metadata qua các JOIN thời gian thực.

### Xây dựng API analytics thời gian thực với Tinybird

Phần hay nhất của Tinybird là khả năng chuyển đổi bất kỳ truy vấn SQL nào thành một API endpoint RESTful chỉ bằng một lệnh CLI.

Cú pháp pipe của Tinybird sử dụng SQL (方言 ClickHouse, hơi khác Postgres). Ví dụ, tạo file `aggregate.pipe`:

```sql
NODE endpoint
SQL >
  SELECT
    toDate(created_at) as date,
    SUM(total_amount) as total_revenue,
    AVG(total_amount) as average_order_value
  FROM orders_in_tinybird -- Bảng trong Tinybird
  WHERE created_at >= {{DateTime(start_date, default='2020-01-01 00:00:00')}} -- Tham số với giá trị mặc định
  AND created_at < {{DateTime(end_date, default='2025-01-01 00:00:00')}}
  GROUP BY date
  ORDER BY date DESC

TYPE endpoint
```
Triển khai API:
```bash
tb --cloud deploy
```
Lấy URL endpoint:
```bash
tb --cloud endpoint url aggregate --language curl
# curl -X GET "https://api.us-east.tinybird.co/v0/pipes/aggregate.json?token=p.ey....ars&start_date=2023-01-01&end_date=2023-02-01"
```

## Kết luận

Xây dựng analytics hướng người dùng với Supabase là khả thi trong nhiều trường hợp, nhưng việc biết khi nào cần thêm một backend analytics như Tinybird là rất quan trọng để mở rộng. Tinybird giúp việc bắt đầu xây dựng analytics trở nên dễ dàng đến mức trong nhiều trường hợp, việc bỏ qua tư duy "chỉ dùng Postgres" và đưa dữ liệu vào Tinybird ngay từ đầu là hợp lý. Dù sao đi nữa, bằng cách hiểu rõ yêu cầu của bạn và chọn cách tiếp cận phù hợp, bạn có thể xây dựng analytics vừa mạnh mẽ vừa hiệu quả về chi phí.

**Ghi nhớ**: giải pháp tốt nhất thường là sự kết hợp của nhiều cách tiếp cận. Sử dụng Supabase cho dữ liệu giao dịch và các tính năng thời gian thực, và bổ sung nó bằng Tinybird khi bạn cần khả năng analytics.

## FAQ: Hiểu về Supabase và Tinybird

**Đâu là sự khác biệt chính giữa Supabase và Tinybird?**
Supabase là một CSDL PostgreSQL được quản lý, tối ưu cho workload giao dịch (CRUD), trong khi Tinybird là một backend analytics thời gian thực xây dựng trên ClickHouse, tối ưu cho truy vấn analytics trên dữ liệu chuỗi thời gian. Hãy coi Supabase là CSDL hoạt động (operational) và Tinybird là CSDL analytics của bạn.

**Khi nào tôi nên sử dụng Supabase và khi nào là Tinybird?**
*   **Sử dụng Supabase cho:**
    *   Xác thực và phân quyền người dùng.
    *   Các hoạt động CRUD (create, read, update, delete).
    *   Đăng ký và cập nhật thời gian thực (real-time subscriptions).
    *   Dữ liệu quan hệ với các mối quan hệ phức tạp.
    *   Dữ liệu giao dịch cần tuân thủ ACID.
*   **Sử dụng Tinybird cho:**
    *   Analytics và tổng hợp thời gian thực.
    *   Phân tích dữ liệu chuỗi thời gian.
    *   Xử lý dữ liệu sự kiện khối lượng lớn.
    *   Xây dựng API analytics.
    *   Truy vấn analytics phức tạp cần hiệu năng dưới giây.

**Làm thế nào để tích hợp Supabase với Tinybird?**
Có ba cách chính:
1.  **PostgreSQL Table Function:** Sử dụng connector PostgreSQL tích hợp của Tinybird để đồng bộ dữ liệu định kỳ.
2.  **Change Data Capture (CDC):** Sử dụng các công cụ như Redpanda, Confluent, hoặc Sequin để sao chép gần thời gian thực.
3.  **Direct Event Streaming:** Gửi sự kiện trực tiếp đến Events API của Tinybird trong khi giữ dữ liệu giao dịch trong Supabase. Sử dụng JOINs tại thời điểm truy vấn hoặc trong materialized views của Tinybird để làm phong phú sự kiện.

**Tôi có thể chạy analytics trực tiếp trên Supabase không?**
Có, nhưng có giới hạn:
*   Hoạt động tốt với tập dữ liệu nhỏ (<500K hàng với ít cột).
*   Tổng hợp và bộ lọc đơn giản.
*   Lượng truy vấn thấp (<100 truy vấn/phút).
*   Yêu cầu không phải thời gian thực.
*   Cân nhắc sử dụng read replicas để tránh ảnh hưởng đến hiệu năng production.
*   Cân nhắc sử dụng các extension có thể cải thiện hiệu năng analytics.

**Sự khác biệt về hiệu năng giữa Tinybird và Supabase là gì?**
Đối với truy vấn analytics trên tập dữ liệu 1 triệu hàng:
*   Supabase: ~3.4 giây độ trễ P95.
*   Tinybird: ~117ms độ trễ P95.
Tinybird duy trì hiệu năng ổn định ngay cả khi dữ liệu tăng lên 10 triệu+ hàng.

**Làm thế nào để xử lý analytics thời gian thực với Supabase?**
Để có analytics thời gian thực (hiệu quả):
1.  Giữ dữ liệu giao dịch trong Supabase.
2.  Truyền sự kiện trực tiếp đến Tinybird.
3.  Sử dụng khả năng truy vấn thời gian thực của Tinybird.
4.  Xây dựng API bằng cú pháp pipe của Tinybird.
5.  Join với dữ liệu Supabase khi cần bằng khả năng join thời gian thực của Tinybird.

**Làm thế nào để bắt đầu với cả Supabase và Tinybird?**
1.  Thiết lập dự án Supabase của bạn cho dữ liệu giao dịch.
2.  Tạo tài khoản Tinybird.
3.  Chọn một phương pháp tích hợp (đồng bộ PostgreSQL, CDC, hoặc truyền trực tiếp).
4.  Bắt đầu xây dựng API analytics với Tinybird.
5.  Sử dụng Supabase cho các tính năng thời gian thực và dữ liệu người dùng.
```