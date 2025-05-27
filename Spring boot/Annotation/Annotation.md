
# 1 So sánh Annotations @Expose và @SerializedName trong Gson

**Cập nhật lần cuối:** 23 tháng 5, 2025 (Lưu ý: Ngày này là từ bài gốc, bạn có thể cập nhật)

**Tác giả:** Bhaskar Ghosh

**Người duyệt:** David Martinez

---

## 1. Giới thiệu

Gson là một thư viện Java mã nguồn mở được phát triển bởi Google để hỗ trợ việc chuyển đổi đối tượng Java sang JSON và ngược lại. Nó cung cấp các kỹ thuật tuần tự hóa (serialization) và giải tuần tự hóa (deserialization) hiệu quả và hỗ trợ các đối tượng phức tạp.

Các thư viện như Gson cung cấp hỗ trợ ánh xạ JSON trực tiếp sang POJO (Plain Old Java Objects). Tuy nhiên, đôi khi, một số thuộc tính cụ thể cần được loại trừ khỏi quá trình tuần tự hóa và giải tuần tự hóa.

Trong hướng dẫn này, chúng ta sẽ thảo luận về hai annotation phổ biến và cần thiết được sử dụng với thư viện Gson: `@Expose` và `@SerializedName`. Mặc dù cả hai annotation này đều liên quan đến khả năng tuần tự hóa và giải tuần tự hóa của các thuộc tính, chúng có các trường hợp sử dụng riêng.

## 2. Thiết lập Gson

Để bắt đầu sử dụng Gson, chúng ta thêm dependency Maven của nó vào file `pom.xml`:

```xml
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.10.1</version>
</dependency>
```

## 3. @Expose trong Gson

Hành vi mặc định của Gson là tuần tự hóa và giải tuần tự hóa tất cả các trường (fields) trong một lớp POJO trừ khi được chỉ định khác. Annotation `@Expose` có thể ghi đè hành vi này và kiểm soát việc bao gồm hoặc loại trừ một trường cụ thể khỏi quá trình tuần tự hóa và giải tuần tự hóa.

Gson chỉ bao gồm các trường được chú thích bằng `@Expose` trong JSON nếu các thuộc tính `serialize` và `deserialize` của chúng được đặt thành `true`. Giá trị mặc định cho `serialize` hoặc `deserialize` là `true`.

Hãy xem một ví dụ. Chúng ta sẽ sử dụng một lớp `User` với các thuộc tính như `id`, `name`, `age`, và `email`. Email là thông tin nhạy cảm, vì vậy chúng ta sẽ loại trừ nó khỏi việc được tuần tự hóa trong JSON đầu ra:

```java
public class User {

    @Expose
    String name;

    @Expose
    int age;

    @Expose(serialize = true, deserialize = false)
    long id;

    @Expose(serialize = false, deserialize = false)
    private String email;

    // Constructors, Getters, and Setters
    // (Các hàm khởi tạo, getters và setters)
}
```

Trong đoạn mã trên, chúng ta chú thích các trường `name` và `age` bằng `@Expose`. Việc không có các thuộc tính `serialize` và `deserialize` rõ ràng cho thấy chúng được mặc định là `true`.

Chúng ta cũng nên lưu ý rằng thuộc tính `email` có cả hai annotation được đặt thành `false`. Do đó, JSON được tuần tự hóa sẽ bỏ qua trường `email`. Tuy nhiên, chúng ta nên sử dụng `excludeFieldsWithoutExposeAnnotation()` khi tạo instance `GsonBuilder` để điều này xảy ra.

Mặt khác, đối với `id`, quá trình tuần tự hóa bao gồm nó, trong khi quá trình giải tuần tự hóa bỏ qua bất kỳ `id` nào có trong JSON:

```java
@Test
public void givenUserObject_whenSerialized_thenCorrectJsonProduced() {
    User user = new User("John Doe", 30, "john.doe@example.com");
    user.setId(12345L);

    Gson gson = new GsonBuilder().excludeFieldsWithoutExposeAnnotation().create();
    String json = gson.toJson(user);

    // Xác minh rằng name, age, và id được tuần tự hóa, nhưng email thì không
    assertEquals("{\"name\":\"John Doe\",\"age\":30,\"id\":12345}", json);
}

@Test
public void givenJsonInput_whenDeserialized_thenCorrectUserObjectProduced() {
    String jsonInput = "{\"name\":\"Jane Doe\",\"age\":25,\"id\":67890,\"email\":\"jane.doe@example.com\"}";

    Gson gson = new GsonBuilder().excludeFieldsWithoutExposeAnnotation()
      .create();
    User user = gson.fromJson(jsonInput, User.class);

    // Xác minh rằng name và age được giải tuần tự hóa, nhưng email và id thì không
    assertEquals("Jane Doe", user.name);
    assertEquals(25, user.getAge());
    assertEquals(0, user.getId()); // id không được giải tuần tự hóa
    assertNull(user.getEmail()); // email không được giải tuần tự hóa
}
```

Chúng ta thấy trong bài kiểm tra đầu tiên rằng JSON được tuần tự hóa không chứa thuộc tính `email`. Trong bài kiểm tra đơn vị thứ hai, chúng ta khẳng định rằng đối tượng `User` được giải tuần tự hóa bỏ qua trường `email` từ JSON của nó.

## 4. @SerializedName trong Gson

Hãy hiểu cách sử dụng annotation `@SerializedName` trong Gson. Khi chúng ta tạo lớp Java và định nghĩa các thuộc tính của nó, biểu diễn JSON có thể yêu cầu một tên khác với tên được định nghĩa trong lớp. Annotation này ánh xạ một thuộc tính POJO tới một tên cụ thể trong biểu diễn JSON được tuần tự hóa của nó.

Tiếp tục với ví dụ trước, giả sử lần này chúng ta muốn biểu diễn JSON của `User` có `firstName` làm tên trường thay vì `name`:

```java
public class User {

    @Expose
    @SerializedName("firstName")
    String name;

    // ... các trường khác và phương thức
}
```

Các bài kiểm tra đơn vị của chúng ta bây giờ nên khẳng định trường `firstName`:

```java
@Test
public void givenUserObject_whenSerialized_thenCorrectJsonProduced() {
    User user = new User("John Doe", 30, "john.doe@example.com");
    user.setId(12345L);

    Gson gson = new GsonBuilder().excludeFieldsWithoutExposeAnnotation().create();
    String json = gson.toJson(user);

    assertEquals("{\"firstName\":\"John Doe\",\"age\":30,\"id\":12345}", json);
}
```

`@SerializedName` hỗ trợ một thuộc tính bổ sung được gọi là `alternate`, nhận vào một danh sách các tên thay thế cho thuộc tính và yêu cầu trình phân tích cú pháp tìm kiếm một trong các giá trị này trong khi giải tuần tự hóa. Đây là một tính năng mạnh mẽ khi tên thuộc tính có thể thay đổi tùy thuộc vào các hệ thống bên ngoài hoặc hệ thống cũ.

Hãy xem xét một hệ thống sử dụng `fullName` thay vì `firstName`. Chúng ta có thể giải mã những điều này một cách chính xác bằng cách điền vào thuộc tính `alternate` một cách phù hợp:

```java
public class User {

    @Expose
    @SerializedName(value = "firstName", alternate = { "fullName", "name" })
    String name;

    // ... các trường khác và phương thức
}
```

```java
@Test
public void givenJsonWithAlternateNames_whenDeserialized_thenCorrectNameFieldMapped() {
    String jsonInput1 = "{\"firstName\":\"Jane Doe\",\"age\":25,\"id\":67890,\"email\":\"jane.doe@example.com\"}";
    String jsonInput2 = "{\"fullName\":\"John Doe\",\"age\":30,\"id\":12345,\"email\":\"john.doe@example.com\"}";
    String jsonInput3 = "{\"name\":\"Alice\",\"age\":28,\"id\":54321,\"email\":\"alice@example.com\"}";

    Gson gson = new GsonBuilder().excludeFieldsWithoutExposeAnnotation().create();

    User user1 = gson.fromJson(jsonInput1, User.class);
    User user2 = gson.fromJson(jsonInput2, User.class);
    User user3 = gson.fromJson(jsonInput3, User.class);

    // Xác minh rằng trường name được giải tuần tự hóa chính xác từ các tên trường JSON khác nhau
    assertEquals("Jane Doe", user1.getName());
    assertEquals("John Doe", user2.getName());
    assertEquals("Alice", user3.getName());
}
```

Chúng ta có thể giải tuần tự hóa chính xác thuộc tính `name` từ các payload JSON đầu vào, có `fullName` và `firstName` làm tên thuộc tính.

## 5. Sự khác biệt giữa @SerializedName và @Expose

Hãy tóm tắt nhanh những khác biệt chính giữa hai annotation này:

| @SerializedName                                                                 | @Expose                                                                                      |
| :------------------------------------------------------------------------------ | :------------------------------------------------------------------------------------------- |
| Ánh xạ một trường Java POJO tới một tên trường JSON.                             | Đánh dấu liệu một trường có nên được tuần tự hóa hoặc giải tuần tự hóa hay không.              |
| Thuộc tính `value` là bắt buộc, và một thuộc tính `alternate` tùy chọn có sẵn. | Hai thuộc tính tùy chọn có sẵn: `serialize` và `deserialize`.                                |
| Hoạt động ngay mà không cần cấu hình thêm (out-of-the-box).                     | Chỉ hoạt động khi được cấu hình với `GsonBuilder` và yêu cầu `GsonBuilder.excludeFieldsWithoutExposeAnnotation()`. |

## 6. Kết luận

Trong bài viết này, chúng ta đã tìm hiểu cách hoạt động của `@SerializedName` và `@Expose` và cách chúng ta có thể sử dụng chúng để xử lý việc tuần tự hóa và giải tuần tự hóa JSON trong Java. Chúng ta cũng đã nêu bật những khác biệt chính giữa hai annotation này.
```