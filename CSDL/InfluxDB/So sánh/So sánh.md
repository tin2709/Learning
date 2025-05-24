Tuyệt vời! Dưới đây là nội dung được dịch và định dạng lại thành file `README.md` bằng tiếng Việt.

```markdown
# Flux so với InfluxQL

Trang này ghi lại tài liệu cho phiên bản cũ hơn của InfluxDB OSS. **InfluxDB 3 Core là phiên bản ổn định mới nhất.**

Flux là một giải pháp thay thế cho InfluxQL và các ngôn ngữ truy vấn giống SQL khác để truy vấn và phân tích dữ liệu. Flux sử dụng các mẫu ngôn ngữ lập trình hàm, giúp nó trở nên cực kỳ mạnh mẽ, linh hoạt và có khả năng vượt qua nhiều hạn chế của InfluxQL. Bài viết này phác thảo nhiều tác vụ có thể thực hiện với Flux mà InfluxQL không thể, đồng thời cung cấp thông tin về sự tương đương giữa Flux và InfluxQL.

- [Các khả năng của Flux](#các-khả-năng-của-flux)
- [Tính tương đương giữa InfluxQL và Flux](#tính-tương-đương-giữa-influxql-và-flux)

## Các khả năng của Flux

Dưới đây là các tính năng mà Flux hỗ trợ, trong đó nhiều tính năng không có hoặc hạn chế trong InfluxQL:

- Phép nối (Joins)
- Tính toán toán học giữa các phép đo (measurements)
- Sắp xếp theo thẻ (tags)
- Nhóm theo cột bất kỳ
- Phân cửa sổ theo tháng và năm dương lịch
- Làm việc với nhiều nguồn dữ liệu
- Truy vấn kiểu DatePart (lọc theo phần của ngày/giờ)
- Xoay bảng (Pivot)
- Biểu đồ tần suất (Histograms)
- Hiệp phương sai (Covariance)
- Ép kiểu boolean sang số nguyên
- Thao tác chuỗi và định hình dữ liệu
- Làm việc với dữ liệu không gian-thời gian (geo-temporal)

### Phép nối (Joins)

InfluxQL chưa bao giờ hỗ trợ phép nối. Chúng có thể được thực hiện bằng cách sử dụng TICKscript, nhưng ngay cả khả năng nối của TICKscript cũng bị hạn chế. Hàm `join()` của Flux cho phép bạn nối dữ liệu từ bất kỳ bucket, bất kỳ measurement nào và trên bất kỳ cột nào, miễn là mỗi tập dữ liệu bao gồm các cột mà chúng sẽ được nối. Điều này mở ra cánh cửa cho các hoạt động thực sự mạnh mẽ và hữu ích.

```flux
dataStream1 = from(bucket: "bucket1")
  |> range(start: -1h)
  |> filter(fn: (r) =>
    r._measurement == "network" and
    r._field == "bytes-transferred"
  )

dataStream2 = from(bucket: "bucket1")
  |> range(start: -1h)
  |> filter(fn: (r) =>
    r._measurement == "httpd" and
    r._field == "requests-per-sec"
    )

join(
    tables: {d1:dataStream1, d2:dataStream2},
    on: ["_time", "_stop", "_start", "host"]
  )
```

Để xem hướng dẫn chi tiết về cách sử dụng hàm `join()`, hãy xem [Cách nối dữ liệu với Flux](https://docs.influxdata.com/influxdb/v2/query-data/flux/join/). (Lưu ý: Liên kết có thể bằng tiếng Anh)

### Tính toán toán học giữa các phép đo (Math across measurements)

Khả năng thực hiện phép nối giữa các measurement cũng cho phép bạn chạy các phép tính sử dụng dữ liệu từ các measurement riêng biệt – một tính năng được cộng đồng InfluxData yêu cầu rất nhiều. Ví dụ dưới đây lấy hai luồng dữ liệu từ các measurement riêng biệt, `mem` và `processes`, nối chúng, sau đó tính toán lượng bộ nhớ trung bình được sử dụng cho mỗi tiến trình đang chạy:

```flux
// Bộ nhớ đã sử dụng (tính bằng byte)
memUsed = from(bucket: "telegraf/autogen")
  |> range(start: -1h)
  |> filter(fn: (r) =>
    r._measurement == "mem" and
    r._field == "used"
  )

// Tổng số tiến trình đang chạy
procTotal = from(bucket: "telegraf/autogen")
  |> range(start: -1h)
  |> filter(fn: (r) =>
    r._measurement == "processes" and
    r._field == "total"
    )

// Nối bộ nhớ đã sử dụng với tổng số tiến trình và tính toán
// bộ nhớ trung bình (tính bằng MB) được sử dụng cho các tiến trình đang chạy.
join(
    tables: {mem:memUsed, proc:procTotal},
    on: ["_time", "_stop", "_start", "host"]
  )
  |> map(fn: (r) => ({
    _time: r._time,
    _value: (r._value_mem / r._value_proc) / 1000000.0 // Đảm bảo phép chia số thực
  })
)
```

### Sắp xếp theo thẻ (Sort by tags)

Khả năng sắp xếp của InfluxQL rất hạn chế, chỉ cho phép bạn kiểm soát thứ tự sắp xếp của thời gian bằng mệnh đề `ORDER BY time`. Hàm `sort()` của Flux sắp xếp các bản ghi dựa trên danh sách các cột. Tùy thuộc vào loại cột, các bản ghi được sắp xếp theo thứ tự từ điển, số học hoặc thời gian.

```flux
from(bucket:"telegraf/autogen")
  |> range(start:-12h)
  |> filter(fn: (r) =>
    r._measurement == "system" and
    r._field == "uptime"
  )
  |> sort(columns:["region", "host", "_value"])
```

### Nhóm theo cột bất kỳ (Group by any column)

InfluxQL cho phép bạn nhóm theo thẻ (tags) hoặc theo khoảng thời gian, nhưng không gì khác. Flux cho phép bạn nhóm theo bất kỳ cột nào trong tập dữ liệu, bao gồm cả `_value`. Sử dụng hàm `group()` của Flux để xác định các cột cần nhóm dữ liệu.

```flux
from(bucket:"telegraf/autogen")
  |> range(start:-12h)
  |> filter(fn: (r) => r._measurement == "system" and r._field == "uptime" )
  |> group(columns:["host", "_value"])
```

### Phân cửa sổ theo tháng và năm dương lịch (Window by calendar months and years)

InfluxQL không hỗ trợ phân cửa sổ dữ liệu theo tháng và năm dương lịch do độ dài thay đổi của chúng. Flux hỗ trợ các đơn vị thời gian tháng và năm dương lịch (`1mo`, `1y`) và cho phép bạn phân cửa sổ và tổng hợp dữ liệu theo tháng và năm dương lịch.

```flux
from(bucket:"telegraf/autogen")
  |> range(start:-1y)
  |> filter(fn: (r) => r._measurement == "mem" and r._field == "used_percent" )
  |> aggregateWindow(every: 1mo, fn: mean)
```

### Làm việc với nhiều nguồn dữ liệu (Work with multiple data sources)

InfluxQL chỉ có thể truy vấn dữ liệu được lưu trữ trong InfluxDB. Flux có thể truy vấn dữ liệu từ các nguồn dữ liệu khác như CSV, PostgreSQL, MySQL, Google BigTable, v.v. Nối dữ liệu đó với dữ liệu trong InfluxDB để làm phong phú kết quả truy vấn.

- Gói Flux CSV (Flux CSV package)
- Gói Flux SQL (Flux SQL package)
- Gói Flux BigTable (Flux BigTable package)

```flux
import "csv"
import "sql"

csvData = csv.from(csv: "raw_csv_data_here") // Thay thế bằng dữ liệu CSV thực tế hoặc đường dẫn
sqlData = sql.from(
  driverName: "postgres",
  dataSourceName: "postgresql://user:password@localhost/dbname", // Cập nhật chuỗi kết nối
  query:"SELECT * FROM example_table"
)
data = from(bucket: "telegraf/autogen")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "sensor")

// Giả sử csvData và sqlData có cột 'sensor_id' để nối
// và 'data' cũng có thể được chuẩn bị để có 'sensor_id'
// Ví dụ này giả định cấu trúc dữ liệu phù hợp cho việc nối
// auxData = join(tables: {csv: csvData, sql: sqlData}, on: ["sensor_id"])
// enrichedData = join(tables: {data: data, aux: auxData}, on: ["sensor_id"])

// enrichedData
//  |> yield(name: "enriched_data")

// Lưu ý: Phần nối với dữ liệu CSV và SQL cần dữ liệu mẫu cụ thể
// để hoạt động đúng. Đoạn code trên là minh họa khái niệm.
// Dưới đây là ví dụ đơn giản hơn chỉ với dữ liệu InfluxDB:
data
    |> yield(name: "influx_data_only")
```
Để xem hướng dẫn chi tiết về cách truy vấn dữ liệu SQL, hãy xem [Truy vấn nguồn dữ liệu SQL](https://docs.influxdata.com/influxdb/v2/query-data/flux/query-sql/). (Lưu ý: Liên kết có thể bằng tiếng Anh)

### Truy vấn kiểu DatePart (DatePart-like queries)

InfluxQL không hỗ trợ các truy vấn kiểu DatePart chỉ trả về kết quả trong các giờ cụ thể trong ngày. Hàm `hourSelection` của Flux chỉ trả về dữ liệu có giá trị thời gian trong một phạm vi giờ được chỉ định.

```flux
from(bucket: "telegraf/autogen")
  |> range(start: -1h)
  |> filter(fn: (r) =>
    r._measurement == "cpu" and
    r.cpu == "cpu-total"
  )
  |> hourSelection(start: 9, stop: 17) // Chỉ lấy dữ liệu từ 9 giờ sáng đến 5 giờ chiều
```

### Xoay bảng (Pivot)

Việc xoay bảng dữ liệu chưa bao giờ được hỗ trợ trong InfluxQL. Hàm `pivot()` của Flux cung cấp khả năng xoay bảng dữ liệu bằng cách chỉ định các tham số `rowKey`, `columnKey`, và `valueColumn`.

```flux
from(bucket: "telegraf/autogen")
  |> range(start: -1h)
  |> filter(fn: (r) =>
    r._measurement == "cpu" and
    r.cpu == "cpu-total"
  )
  |> pivot(
    rowKey:["_time"],
    columnKey: ["_field"],
    valueColumn: "_value"
  )
```

### Biểu đồ tần suất (Histograms)

Khả năng tạo biểu đồ tần suất là một tính năng được yêu cầu nhiều cho InfluxQL, nhưng chưa bao giờ được hỗ trợ. Hàm `histogram()` của Flux sử dụng dữ liệu đầu vào để tạo biểu đồ tần suất tích lũy, với sự hỗ trợ cho các loại biểu đồ tần suất khác trong tương lai.

```flux
from(bucket: "telegraf/autogen")
  |> range(start: -1h)
  |> filter(fn: (r) =>
    r._measurement == "mem" and
    r._field == "used_percent"
  )
  |> histogram(
    // Định nghĩa các khoảng (buckets) cho biểu đồ tần suất
    column: "_value", // Cột để tạo histogram
    upperBoundColumn: "le", // Tên cột cho giới hạn trên của bucket
    countColumn: "_value", // Tên cột cho số lượng trong bucket
    bins: [10.0, 20.0, 30.0, 40.0, 50.0, 60.0, 70.0, 80.0, 90.0, 100.0] // Sử dụng số thực cho bins
  )
```
Để xem ví dụ về cách sử dụng Flux để tạo biểu đồ tần suất tích lũy, hãy xem [Tạo biểu đồ tần suất](https://docs.influxdata.com/influxdb/v2/query-data/flux/visualize-data/histogram/). (Lưu ý: Liên kết có thể bằng tiếng Anh)

### Hiệp phương sai (Covariance)

Flux cung cấp các hàm để tính toán hiệp phương sai đơn giản. Hàm `covariance()` tính toán hiệp phương sai giữa hai cột và hàm `cov()` tính toán hiệp phương sai giữa hai luồng dữ liệu.

**Hiệp phương sai giữa hai cột**
```flux
from(bucket: "telegraf/autogen")
  |> range(start:-5m)
  // Giả sử measurement này có các trường 'x' và 'y'
  |> filter(fn: (r) => r._measurement == "some_measurement_with_x_y")
  |> covariance(columns: ["x", "y"]) // Đảm bảo các trường x, y tồn tại và là số
```

**Hiệp phương sai giữa hai luồng dữ liệu**
```flux
table1 = from(bucket: "telegraf/autogen")
  |> range(start: -15m)
  |> filter(fn: (r) =>
    r._measurement == "measurement_1" // Và trường giá trị số phù hợp
  )

table2 = from(bucket: "telegraf/autogen")
  |> range(start: -15m)
  |> filter(fn: (r) =>
    r._measurement == "measurement_2" // Và trường giá trị số phù hợp
  )

cov(x: table1, y: table2, on: ["_time", "_field"])
```

### Ép kiểu boolean sang số nguyên (Cast booleans to integers)

InfluxQL hỗ trợ ép kiểu, nhưng chỉ cho các kiểu dữ liệu số (số thực sang số nguyên và ngược lại). Các hàm chuyển đổi kiểu của Flux cung cấp hỗ trợ rộng hơn nhiều cho việc chuyển đổi kiểu và cho phép bạn thực hiện một số thao tác được yêu cầu từ lâu như ép kiểu giá trị boolean sang số nguyên.

**Ép kiểu giá trị trường boolean sang số nguyên**
```flux
from(bucket: "telegraf/autogen")
  |> range(start: -1h)
  |> filter(fn: (r) =>
    r._measurement == "m" and
    r._field == "bool_field" // Đảm bảo 'bool_field' chứa giá trị boolean
  )
  |> toInt()
```

### Thao tác chuỗi và định hình dữ liệu (String manipulation and data shaping)

InfluxQL không hỗ trợ thao tác chuỗi khi truy vấn dữ liệu. Gói `strings` của Flux là một tập hợp các hàm hoạt động trên dữ liệu chuỗi. Khi kết hợp với hàm `map()`, các hàm trong gói `strings` cho phép các hoạt động như làm sạch chuỗi và chuẩn hóa.

```flux
import "strings"

from(bucket: "telegraf/autogen")
  |> range(start: -1h)
  |> filter(fn: (r) =>
    r._measurement == "weather" and
    r._field == "temp" // Giả sử các tag 'location', 'sensor', 'status' tồn tại
  )
  |> map(fn: (r) => ({
    r with // 'with' chỉ hoạt động với bản ghi, không phải trực tiếp với r
    // Tạo bản ghi mới với các giá trị đã biến đổi
    _time: r._time,
    _value: r._value,
    _field: r._field,
    _measurement: r._measurement,
    // Giả sử các tag này tồn tại trên bản ghi 'r'
    location: strings.toTitle(v: r.location),
    sensor: strings.replaceAll(v: r.sensor, t: " ", u: "-"),
    status: strings.substring(v: r.status, start: 0, end: 8)
  }))
```

### Làm việc với dữ liệu không gian-thời gian (Work with geo-temporal data)

InfluxQL không cung cấp chức năng để làm việc với dữ liệu không gian-thời gian. Gói `geo` của Flux (hiện tại là `experimental/geo`) là một tập hợp các hàm cho phép bạn định hình, lọc và nhóm dữ liệu không gian-thời gian.

```flux
import "experimental/geo" // Hoặc import "geo" nếu đã ổn định

from(bucket: "geo/autogen") // Đảm bảo bucket này tồn tại và có dữ liệu geo
  |> range(start: -1w)
  |> filter(fn: (r) => r._measurement == "taxi") // Và có các trường latitude, longitude
  // Đảm bảo các trường latField, lonField tồn tại
  |> geo.shapeData(latField: "latitude", lonField: "longitude", level: 20)
  |> geo.filterRows(
    region: {lat: 40.69335938, lon: -73.30078125, radius: 20.0}, // Bán kính tính bằng km
    strict: true // true để chỉ giữ các điểm trong vùng, false để giữ các điểm có thể trong vùng
  )
  |> geo.asTracks(groupBy: ["fare-id"], timestampColumn: "_time") // Cần timestampColumn
```

## Tính tương đương giữa InfluxQL và Flux

Flux đang hướng tới sự tương đương hoàn toàn với InfluxQL và các hàm mới đang được thêm vào để đạt được mục tiêu đó. Bảng dưới đây cho thấy các câu lệnh, mệnh đề và hàm của InfluxQL cùng với các hàm Flux tương đương của chúng.

Để có danh sách đầy đủ các hàm Flux, hãy xem [tất cả các hàm Flux](https://docs.influxdata.com/flux/v0.x/reference/functions/). (Lưu ý: Liên kết có thể bằng tiếng Anh)

| InfluxQL Statement/Clause/Function | Flux Functions                                    |
|------------------------------------|---------------------------------------------------|
| `SELECT`                           | `filter()`                                        |
| `WHERE`                            | `filter()`, `range()`                             |
| `GROUP BY`                         | `group()`                                         |
| `INTO`                             | `to()` *                                          |
| `ORDER BY`                         | `sort()`                                          |
| `LIMIT`                            | `limit()`                                         |
| `SLIMIT`                           | – (Sử dụng `group()`, `limit()` kết hợp)          |
| `OFFSET`                           | – (Flux không có offset trực tiếp, cần logic tùy chỉnh) |
| `SOFFSET`                          | –                                                 |
| `SHOW DATABASES`                   | `buckets()`                                       |
| `SHOW MEASUREMENTS`                | `v1.measurements()` (cho InfluxDB 1.x compatibility) hoặc `schema.measurements()` (cho InfluxDB 2.x+) |
| `SHOW FIELD KEYS`                  | `keys()` (trên bảng đã lọc) hoặc `schema.fieldKeys()` |
| `SHOW RETENTION POLICIES`          | `buckets()` (RPs được quản lý ở cấp bucket trong 2.x) |
| `SHOW TAG KEYS`                    | `v1.tagKeys()`, `v1.measurementTagKeys()` (1.x compat) hoặc `schema.tagKeys()` |
| `SHOW TAG VALUES`                  | `v1.tagValues()`, `v1.measurementTagValues()` (1.x compat) hoặc `schema.tagValues()` |
| `SHOW SERIES`                      | – (Xây dựng logic tương tự bằng cách nhóm và đếm) |
| `CREATE DATABASE`                  | – (Quản lý qua `influx` CLI hoặc API cho `buckets`) |
| `DROP DATABASE`                    | – (Quản lý qua `influx` CLI hoặc API cho `buckets`) |
| `DROP SERIES`                      | `delete()` (với predicates phù hợp)              |
| `DELETE`                           | `delete()`                                        |
| `DROP MEASUREMENT`                 | `delete()` (với `_measurement` predicate)         |
| `DROP SHARD`                       | – (Không quản lý trực tiếp qua Flux)              |
| `CREATE RETENTION POLICY`          | – (Quản lý qua `influx` CLI hoặc API cho `buckets`) |
| `ALTER RETENTION POLICY`           | – (Quản lý qua `influx` CLI hoặc API cho `buckets`) |
| `DROP RETENTION POLICY`            | – (Quản lý qua `influx` CLI hoặc API cho `buckets`) |
| **Hàm tổng hợp**                  |                                                   |
| `COUNT`                            | `count()`                                         |
| `DISTINCT`                         | `distinct()`                                      |
| `INTEGRAL`                         | `integral()`                                      |
| `MEAN`                             | `mean()`                                          |
| `MEDIAN`                           | `median()` hoặc `quantile(q: 0.5)`                |
| `MODE`                             | `mode()`                                          |
| `SPREAD`                           | `spread()`                                        |
| `STDDEV`                           | `stddev()`                                        |
| `SUM`                              | `sum()`                                           |
| **Hàm chọn**                       |                                                   |
| `BOTTOM`                           | `bottom()`                                        |
| `FIRST`                            | `first()`                                         |
| `LAST`                             | `last()`                                          |
| `MAX`                              | `max()`                                           |
| `MIN`                              | `min()`                                           |
| `PERCENTILE`                       | `quantile()`                                      |
| `SAMPLE`                           | `sample()`                                        |
| `TOP`                              | `top()`                                           |
| **Hàm toán học & biến đổi**        |                                                   |
| `ABS`                              | `math.abs()`                                      |
| `ACOS`                             | `math.acos()`                                     |
| `ASIN`                             | `math.asin()`                                     |
| `ATAN`                             | `math.atan()`                                     |
| `ATAN2`                            | `math.atan2()`                                    |
| `CEIL`                             | `math.ceil()`                                     |
| `COS`                              | `math.cos()`                                      |
| `CUMULATIVE_SUM`                   | `cumulativeSum()`                                 |
| `DERIVATIVE`                       | `derivative()`                                    |
| `DIFFERENCE`                       | `difference()`                                    |
| `ELAPSED`                          | `elapsed()`                                       |
| `EXP`                              | `math.exp()`                                      |
| `FLOOR`                            | `math.floor()`                                    |
| `HISTOGRAM` (InfluxQL)             | `histogram()` (Flux, linh hoạt hơn)               |
| `LN`                               | `math.log()`                                      |
| `LOG` (cơ số b)                    | `math.logb()`                                     |
| `LOG2`                             | `math.log2()`                                     |
| `LOG10`                            | `math.log10()`                                    |
| `MOVING_AVERAGE`                   | `movingAverage()`                                 |
| `NON_NEGATIVE_DERIVATIVE`          | `derivative(nonNegative:true)`                    |
| `NON_NEGATIVE_DIFFERENCE`          | `difference(nonNegative:true)`                    |
| `POW`                              | `math.pow()`                                      |
| `ROUND`                            | `math.round()`                                    |
| `SIN`                              | `math.sin()`                                      |
| `SQRT`                             | `math.sqrt()`                                     |
| `TAN`                              | `math.tan()`                                      |
| **Hàm dự báo & nâng cao**          |                                                   |
| `HOLT_WINTERS`                     | `holtWinters()`                                   |
| `CHANDE_MOMENTUM_OSCILLATOR`       | `chandeMomentumOscillator()` (trong `influxdata/flux`) |
| `EXPONENTIAL_MOVING_AVERAGE`       | `exponentialMovingAverage()` (trong `influxdata/flux`) |
| `DOUBLE_EXPONENTIAL_MOVING_AVERAGE`| `doubleEMA()` (trong `influxdata/flux`)           |
| `KAUFMANS_EFFICIENCY_RATIO`        | `kaufmansER()` (trong `influxdata/flux`)          |
| `KAUFMANS_ADAPTIVE_MOVING_AVERAGE` | `kaufmansAMA()` (trong `influxdata/flux`)         |
| `TRIPLE_EXPONENTIAL_MOVING_AVERAGE`| `tripleEMA()` (trong `influxdata/flux`)           |
| `TRIPLE_EXPONENTIAL_DERIVATIVE`    | `tripleExponentialDerivative()` (trong `influxdata/flux`) |
| `RELATIVE_STRENGTH_INDEX`          | `relativeStrengthIndex()` (trong `influxdata/flux`) |

\* Hàm `to()` chỉ ghi vào InfluxDB 2.0 trở lên (bao gồm cả InfluxDB Cloud).
```