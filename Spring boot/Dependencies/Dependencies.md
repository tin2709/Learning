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

        # README: Tản mạn về Liquibase và cách tích hợp vào ứng dụng Spring Boot
```
# 2 Giới thiệu Liquibase


Liquibase là một công cụ mã nguồn mở mạnh mẽ, giúp đơn giản hóa việc theo dõi, quản lý phiên bản và tự động hóa việc triển khai các thay đổi cơ sở dữ liệu (DB).

## Tại sao cần Liquibase? (Vấn đề & Giải pháp)

Bài viết sử dụng câu chuyện về "Dan" để minh họa tầm quan trọng của việc tuân thủ kế hoạch. Giống như Dan gặp rắc rối khi phá vỡ lịch trình hàng ngày của mình, việc quản lý thay đổi DB một cách thủ công hoặc không có kế hoạch rõ ràng có thể dẫn đến:

*   Lỗi hệ thống.
*   Dữ liệu không nhất quán giữa các môi trường (dev, test, prod).
*   Khó khăn trong việc theo dõi lịch sử thay đổi.
*   Mất thời gian và tăng rủi ro khi triển khai.

**Liquibase cung cấp giải pháp bằng cách:**

*   **Áp dụng một phương pháp có cấu trúc:** Định nghĩa các thay đổi DB trong các file `changelog`.
*   **Đảm bảo tính nhất quán:** Các thay đổi được áp dụng tuần tự và được theo dõi.
*   **Tự động hóa:** Tích hợp vào quy trình build/deploy để tự động cập nhật DB.
*   **Kiểm soát phiên bản:** Theo dõi lịch sử thay đổi, tương tự như Git cho code.
*   **Giảm thiểu rủi ro:** Có các cơ chế kiểm tra và kiểm soát (như checksum, lock).

## Liquibase hoạt động như thế nào? (Khái niệm cốt lõi)

1.  **Changelog File:** Là trung tâm của Liquibase, thường là file XML (nhưng cũng hỗ trợ YAML, JSON, SQL). File này chứa danh sách các thay đổi cần áp dụng lên DB.
2.  **Changesets:** Changelog được chia thành các đơn vị gọi là `changeset`. Mỗi `changeset` đại diện cho một thay đổi nguyên tử (atomic) đối với DB (ví dụ: tạo bảng, thêm cột). Mỗi changeset được định danh duy nhất bằng `id` và `author`.
3.  **Bảng theo dõi:** Khi chạy lần đầu, Liquibase tự động tạo 2 bảng trong DB của bạn:
    *   `DATABASECHANGELOG`: Ghi lại lịch sử các changeset nào đã được thực thi thành công (dựa trên id, author, tên file, checksum). Liquibase dựa vào bảng này để biết cần chạy những changeset mới nào.
    *   `DATABASECHANGELOGLOCK`: Đảm bảo rằng chỉ có một tiến trình Liquibase chạy trên cùng một DB tại một thời điểm, tránh xung đột. Nó hoạt động như một cơ chế khóa (mutex).

## Tích hợp với Spring Boot (Các bước chính)

Bài viết hướng dẫn các bước cơ bản để tích hợp Liquibase vào Spring Boot:

1.  **Thêm Dependency:** Khai báo dependency `liquibase-core` trong file `pom.xml` (hoặc `build.gradle`).
    ```xml
    <dependency>
        <groupId>org.liquibase</groupId>
        <artifactId>liquibase-core</artifactId>
        <!-- <version>...</version> --> <!-- Spring Boot thường quản lý phiên bản -->
    </dependency>
    ```
2.  **Tạo Configuration Bean (Tùy chọn - Spring Boot có thể tự cấu hình):** Mặc dù Spring Boot thường tự động cấu hình Liquibase nếu phát hiện dependency và file changelog mặc định, bạn có thể tùy chỉnh bằng cách tạo một Bean `SpringLiquibase`. Ví dụ cấu hình cơ bản chỉ định đường dẫn file changelog:
    ```java
    @Configuration
    // @EnableConfigurationProperties({ LiquibaseProperties.class}) // Cần nếu muốn dùng properties từ application.properties
    public class LiquibaseConfiguration {

        @Bean
        public SpringLiquibase liquibase(DataSource dataSource, LiquibaseProperties properties) {
             // Sử dụng LiquibaseProperties nếu cần cấu hình từ application.properties
             // ví dụ: properties.getChangeLog(), properties.getContexts(), ...

            SpringLiquibase liquibase = new SpringLiquibase();
            // Đường dẫn mặc định là classpath:db/changelog/db.changelog-master.yaml (hoặc .xml, .json)
            // Có thể thay đổi qua application.properties: spring.liquibase.change-log=classpath:databaseChangeLog.xml
            liquibase.setChangeLog("classpath:databaseChangeLog.xml"); // Chỉ định file changelog chính
            liquibase.setDataSource(dataSource);
            // liquibase.setContexts(...);
            // liquibase.setDefaultSchema(...);
            // ... các cấu hình khác
            return liquibase;
        }
    }
    ```
    *Lưu ý:* Với Spring Boot, cách phổ biến hơn là cấu hình qua `application.properties` hoặc `application.yml` (ví dụ: `spring.liquibase.change-log=classpath:db/changelog/db.changelog-master.xml`).
3.  **Tạo Changelog File:** Tạo file changelog chính (ví dụ: `databaseChangeLog.xml` hoặc theo đường dẫn cấu hình) trong thư mục `src/main/resources`.
4.  **Định nghĩa Changesets:** Viết các thay đổi DB (tạo bảng, thêm cột,...) bên trong các thẻ `<changeSet>` trong file changelog. Có thể sử dụng thẻ `<include file="path/to/other/changelog.xml"/>` để chia nhỏ và quản lý các file changelog con.
    ```xml
    <databaseChangeLog ...>
        <include file="db/changelog/changes/001-create-initial-tables.xml"/>
        <include file="db/changelog/changes/002-add-user-roles.xml"/>

        <changeSet author="yourname" id="feature-xyz-1">
            <addColumn tableName="your_table">
                <column name="new_column" type="varchar(255)"/>
            </addColumn>
        </changeSet>
        <!-- Các changeset khác -->
    </databaseChangeLog>
    ```
5.  **Khởi chạy ứng dụng:** Khi ứng dụng Spring Boot khởi động, nó sẽ tự động chạy Liquibase (nếu được bật - mặc định là bật). Liquibase sẽ kiểm tra bảng `DATABASECHANGELOG`, tìm các changeset mới trong file changelog và thực thi chúng.

## Điểm quan trọng

*   **Tính bất biến của Changeset:** Liquibase tính toán và lưu trữ `checksum` (tổng kiểm) cho mỗi changeset đã thực thi vào bảng `DATABASECHANGELOG`. Nếu bạn sửa đổi nội dung của một changeset *đã được thực thi*, Liquibase sẽ phát hiện sự thay đổi checksum này khi khởi động lần sau và báo lỗi `ValidationFailedException`, ngăn ứng dụng khởi động. Điều này đảm bảo rằng lịch sử thay đổi DB là bất biến và nhất quán. **Muốn thay đổi cấu trúc đã tạo, bạn phải tạo một `changeset` mới.**
*   **Bảng `DATABASECHANGELOG`:** Là "bộ não" ghi nhớ lịch sử, chứa thông tin chi tiết về các changeset đã chạy (id, author, filename, checksum, ngày chạy, trạng thái,...).
*   **Bảng `DATABASECHANGELOGLOCK`:** Ngăn chặn tình trạng "race condition" khi nhiều instance ứng dụng cùng khởi động và cố gắng chạy migration. Bảng này chứa thông tin về việc ai đang giữ khóa và khi nào. Nếu ứng dụng bị tắt đột ngột khi đang giữ khóa, bạn có thể cần phải giải phóng khóa thủ công. Bài viết gợi ý có thể dùng trạng thái khóa (cột `LOCKED` = 1) để tạm ngưng migration nếu DB gặp sự cố.

## Kết luận

Liquibase là một công cụ giá trị để quản lý các thay đổi cơ sở dữ liệu một cách có hệ thống, đáng tin cậy và tự động, đặc biệt hữu ích trong các dự án phát triển phần mềm hiện đại, nhất là khi tích hợp với các framework như Spring Boot.

---

*Nguồn: [Tản mạn về Liquibase và cách tích hợp vào ứng dụng Spring Boot - Tuanh.net](https://www.tuanh.net/blog/spring/stories-about-liquibase-and-how-to-integrate-it-into-spring-boot-applications)*