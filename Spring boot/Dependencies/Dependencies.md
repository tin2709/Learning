# 1 Google Guava: Core Libraries for Java

Guava là một bộ thư viện Java cốt lõi mã nguồn mở từ Google, bao gồm các kiểu collection mới (như multimap và multiset), các collection bất biến (immutable), thư viện đồ thị (graph library), và các tiện ích cho concurrency, I/O, hashing, caching, kiểu dữ liệu nguyên thủy, xử lý chuỗi, và nhiều hơn nữa.

Nó được sử dụng rộng rãi trong nhiều dự án Java tại Google và cộng đồng mã nguồn mở, nhằm mục đích làm cho việc phát triển Java trở nên dễ dàng, hiệu quả và đáng tin cậy hơn.

## Tại sao sử dụng Guava?

*   **Tăng Năng Suất:** Giảm mã boilerplate, giúp viết code Java ngắn gọn và rõ ràng hơn.
*   **Độ Tin Cậy:** Cung cấp các thành phần mạnh mẽ, được kiểm thử kỹ lưỡng, đặc biệt là các collection bất biến giúp đơn giản hóa lập trình đa luồng.
*   **Chức Năng Mở Rộng:** Bổ sung nhiều tiện ích và cấu trúc dữ liệu hữu ích còn thiếu hoặc chưa tối ưu trong JDK chuẩn.
*   **Hiệu Năng:** Cung cấp các triển khai hiệu quả cho các tác vụ phổ biến như caching và xử lý collection.

## Các Thành phần và Tiện ích Cốt lõi

Dưới đây là danh sách các lĩnh vực và thành phần nổi bật của Guava:

### 1. Collections Nâng cao (`com.google.common.collect`)

Đây là một trong những phần mạnh mẽ và được sử dụng rộng rãi nhất của Guava.

*   **Immutable Collections (Collections Bất biến):**
    *   Cung cấp các phiên bản thread-safe, không thể sửa đổi của các collection chuẩn (`ImmutableList`, `ImmutableSet`, `ImmutableMap`, `ImmutableSortedSet`, `ImmutableSortedMap`, v.v.).
    *   Đảm bảo tính toàn vẹn dữ liệu, đơn giản hóa lập trình đa luồng và hiệu quả về bộ nhớ. Rất tốt để trả về dữ liệu hoặc lưu trữ cấu hình cố định.
*   **New Collection Types (Các Kiểu Collection Mới):**
    *   `Multiset`: Cho phép đếm số lần xuất hiện của phần tử (giống Bag). Hữu ích cho thống kê.
    *   `Multimap`: Dễ dàng ánh xạ một khóa tới nhiều giá trị (`ListMultimap`, `SetMultimap`), tránh boilerplate `Map<K, List<V>>`. Rất tốt để nhóm đối tượng.
    *   `BiMap`: Map hai chiều, đảm bảo khóa và giá trị đều duy nhất, cho phép tra cứu ngược.
    *   `Table`: Cấu trúc dữ liệu dạng bảng, sử dụng hai khóa (hàng, cột) để truy cập giá trị.
*   **Collection Utilities (Tiện ích Collection):**
    *   Các lớp tiện ích như `Lists`, `Sets`, `Maps`, `Iterables`, `Iterators`, `Multimaps`, `Multisets`, `Tables`.
    *   `FluentIterable`: API trôi chảy để lọc, biến đổi collections (dù Java Streams hiện đại hơn).
    *   `Ordering`: Tạo và kết hợp các `Comparator` một cách linh hoạt.
    *   Cung cấp nhiều phương thức tiện lợi để tạo, sao chép, lọc, biến đổi, phân vùng (partition), và thực hiện các phép toán tập hợp (union, intersection, difference).

### 2. Caching (`com.google.common.cache`)

*   `CacheBuilder`: Tạo các cache trong bộ nhớ (in-memory) hiệu năng cao.
*   `Cache`, `LoadingCache`: Giao diện cache, `LoadingCache` có thể tự động tải giá trị khi cache miss.
*   **Lợi ích:** Cải thiện hiệu năng bằng cách lưu trữ kết quả tính toán tốn kém hoặc dữ liệu truy cập thường xuyên. Cung cấp chính sách loại bỏ linh hoạt (kích thước, thời gian, tham chiếu), thống kê cache, làm mới tự động.

### 3. Preconditions (`com.google.common.base.Preconditions`)

*   Các phương thức tĩnh như `checkNotNull()`, `checkArgument()`, `checkState()`, `checkElementIndex()`, `checkPositionIndexes()`.
*   **Lợi ích:** Cách thức ngắn gọn, rõ ràng để kiểm tra điều kiện tiên quyết (đối số hợp lệ, trạng thái đúng) ở đầu phương thức, giúp phát hiện lỗi sớm và làm code dễ hiểu hơn. Ném ra các ngoại lệ chuẩn.

### 4. String Manipulation (`com.google.common.base`)

*   `Joiner`: Nối chuỗi hoặc đối tượng với dấu phân cách, xử lý null linh hoạt.
*   `Splitter`: Tách chuỗi mạnh mẽ hơn `String.split()` với nhiều tùy chọn (trim, bỏ qua rỗng, giới hạn).
*   `CharMatcher`: Xác định và thao tác với các tập hợp ký tự (ví dụ: chỉ giữ lại số).
*   `CaseFormat`: Chuyển đổi giữa các quy ước định dạng tên (ví dụ: `lowerCamel` sang `UPPER_UNDERSCORE`).
*   **Lợi ích:** Cung cấp công cụ xử lý chuỗi mạnh mẽ và linh hoạt hơn JDK.

### 5. Primitives Utilities (`com.google.common.primitives`)

*   Các lớp tiện ích như `Ints`, `Longs`, `Doubles`, `Booleans`, v.v.
*   **Lợi ích:** Cung cấp các phương thức tiện ích cho kiểu dữ liệu nguyên thủy (đặc biệt là thao tác mảng và chuyển đổi an toàn `tryParse`) mà các lớp Wrapper không có.

### 6. I/O Utilities (`com.google.common.io`)

*   `Files`, `ByteStreams`, `CharStreams`, `ByteSource`, `CharSource`, `ByteSink`, `CharSink`.
*   **Lợi ích:** Đơn giản hóa các tác vụ đọc/ghi file và stream, thường đảm bảo đóng tài nguyên đúng cách. Cung cấp cách trừu tượng hóa nguồn/đích I/O.

### 7. Hashing (`com.google.common.hash`)

*   `Hashing`: API trôi chảy để tạo các hàm băm khác nhau (MD5, SHA-256, MurmurHash,...).
*   `HashCode`: Đối tượng đại diện cho kết quả băm.
*   **Lợi ích:** Cung cấp cách nhất quán và dễ sử dụng để làm việc với giá trị băm.

### 8. Concurrency Utilities (`com.google.common.util.concurrent`)

*   `ListenableFuture`: Mở rộng `Future`, cho phép đăng ký callback xử lý bất đồng bộ.
*   `Futures`: Lớp tiện ích cho `Future` và `ListenableFuture`.
*   `RateLimiter`: Giới hạn tần suất thực thi hành động.
*   `Service` Framework: Quản lý vòng đời của các dịch vụ chạy nền.
*   **Lợi ích:** Cung cấp công cụ nâng cao cho xử lý đa luồng và bất đồng bộ (dù `CompletableFuture` của Java 8+ và Spring `@Async` cũng rất mạnh).

### 9. EventBus (`com.google.common.eventbus`)

*   `EventBus`: Cơ chế publish-subscribe đơn giản cho giao tiếp trong cùng ứng dụng (in-process).
*   **Lợi ích:** Giảm sự phụ thuộc trực tiếp giữa các component. (Trong Spring, `ApplicationEventPublisher` thường được ưu tiên hơn).

### 10. Range (`com.google.common.collect.Range`)

*   `Range`: Biểu diễn các khoảng giá trị (đóng, mở, vô hạn).
*   `RangeSet`, `RangeMap`: Các cấu trúc dữ liệu làm việc với khoảng.
*   **Lợi ích:** Mô hình hóa và truy vấn các khoảng giá trị một cách rõ ràng.

### 11. Reflection (`com.google.common.reflect`)

*   `TypeToken`: Giải quyết vấn đề type erasure của Java khi làm việc với generic type tại thời điểm chạy.
*   **Lợi ích:** Đơn giản hóa các tác vụ reflection phức tạp, đặc biệt với generics.

## Sử dụng Guava

Để sử dụng Guava trong dự án Maven của bạn, hãy thêm dependency sau vào file `pom.xml`:

```xml
<dependency>
			<groupId>com.google.guava</groupId>
			<artifactId>guava</artifactId>
			<version>${guava.version}</version>
		</dependency>