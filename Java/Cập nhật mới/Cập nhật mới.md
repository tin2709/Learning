
# 1 Giới thiệu về Class-File API

Cập nhật lần cuối: 14 tháng 5, 2025

Tác giả: Pedro Lopes
Người duyệt: Eric Martin
Chủ đề: Core Java Class Loaders

## 1. Giới thiệu

Java Class-File API được giới thiệu trong JEP-484, là một phần của Java 24. Nó nhằm mục đích tạo ra một giao diện cho phép xử lý các tệp class mà không cần dựa vào bản sao nội bộ (internal copy) của thư viện ASM trong JDK cũ.

Trong hướng dẫn này, chúng ta sẽ xem cách xây dựng tệp class từ đầu (from scratch) và cách biến đổi một tệp class thành tệp khác bằng cách sử dụng Class-File API.

## 2. Các Thành Phần Cốt Lõi của Class-File API

Class-File API có ba thành phần cốt lõi để tạo và biến đổi các tính năng mà chúng ta sẽ tìm hiểu sau:

*   Một **element** (phần tử) đại diện cho một phần của mã, chẳng hạn như biến, chỉ thị (instruction), phương thức (method) hoặc lớp (class). Ngoài ra, một element có thể chứa các element khác. Ví dụ, một element lớp có thể chứa các element phương thức, và các element phương thức này lại bao gồm các element biến hoặc chỉ thị.
*   **Builders** (trình xây dựng), chẳng hạn như `MethodBuilder` và `CodeBuilder`, được sử dụng để tạo từng loại element.
*   Một hàm **transform** (biến đổi) có thể được sử dụng để biến đổi các element thành các element khác bằng cách sử dụng builders.

Hãy cùng khám phá cách ba thành phần này kết nối với nhau thông qua các ví dụ thực tế trong các phần sau.

## 3. Tạo Tệp Class (Generating a Class-File)

Trong phần này, chúng ta sẽ xem cách tạo một tệp class bằng cách sử dụng các lớp `MethodBuilder` và `CodeBuilder`.

### 3.1. Phương Thức Ví Dụ (Demo Method)

Để minh họa việc tạo class, hãy xem một đoạn mã đơn giản tính toán tiền thưởng hàng năm của nhân viên dựa trên chức năng (vai trò) và lương cơ bản của họ:

```java
public double calculateAnnualBonus(double baseSalary, String role) {
    if (role.equals("sales")) {
        return baseSalary * 0.35;
    }

    if (role.equals("engineer")) {
        return baseSalary * 0.25;
    }

    return baseSalary * 0.15;
}
```

### 3.2. Sử dụng MethodBuilder và CodeBuilder

Để tạo một phương thức có chức năng tương tự như `calculateAnnualBonus()`, chúng ta có thể sử dụng các lớp `MethodBuilder` và `CodeBuilder`. Do đó, trước tiên hãy định nghĩa phương thức `generate()` với một `Consumer<MethodBuilder>` sẽ được sử dụng để xây dựng các phương thức:

```java
public static void generate() throws IOException {
    Consumer<MethodBuilder> calculateAnnualBonusBuilder = methodBuilder -> methodBuilder.withCode(codeBuilder -> {
        Label notSales = codeBuilder.newLabel();
        Label notEngineer = codeBuilder.newLabel();
        ClassDesc stringClass = ClassDesc.of("java.lang.String");

        // ... mã bytecode sẽ ở đây ...
    });
    // ... mã tạo class và ghi file sẽ ở đây ...
}
```

Đầu tiên, chúng ta định nghĩa hai đối tượng `Label` sẽ được sử dụng sau này để nhảy giữa các câu lệnh điều kiện. Ngoài ra, chúng ta đã định nghĩa một hằng số `ClassDesc` đại diện cho tệp class `String` để sử dụng sau này.

Sau đó, chúng ta có thể thêm phần đầu tiên của logic vào biểu thức lambda của `calculateAnnualBonusBuilder`:

```java
codeBuilder.aload(3) // Tải tham số 'role' (slot 3)
  .ldc("sales")    // Tải hằng số "sales"
  .invokevirtual(stringClass, "equals", MethodTypeDesc.of(ClassDesc.of("Z"), stringClass)) // Gọi String.equals("sales")
  .ifeq(notSales)  // Nếu kết quả = false (không phải "sales"), nhảy đến nhãn 'notSales'
  .dload(1)        // Nếu kết quả = true (là "sales"), tải tham số 'baseSalary' (slot 1)
  .ldc(0.35)       // Tải hằng số 0.35
  .dmul()          // Nhân baseSalary với 0.35
  .dreturn();       // Trả về kết quả (double)
```

Hãy xem chi tiết từng dòng mã logic ở trên:

*   Chúng ta bắt đầu bằng cách sử dụng `aload(3)` để tải tham số phương thức `role` vào một tham chiếu. Lưu ý rằng tham số của `aload()` là số slot của biến trong danh sách tham số phương thức, trong đó các kiểu `long` và `double` chiếm hai slot. Do đó, tham số `baseSalary` đầu tiên nằm ở slot 1, và `role` nằm ở slot 3.
*   Sau đó, chúng ta sử dụng `ldc()` để lưu hằng số `"sales"` vào stack toán hạng cho các thao tác tiếp theo.
*   Tiếp theo, chúng ta gọi `invokevirtual()` trên các toán hạng cuối cùng từ stack, đó là hằng số `"sales"` và tham số `role`. Hơn nữa, chúng ta gọi phương thức `equals()` của lớp `String` được lưu trong biến `stringClass` để so sánh các toán hạng. Phần `ClassDesc.of(Z)` nói rằng chúng ta mong đợi một giá trị boolean làm kiểu trả về của lời gọi phương thức đó.
*   Sau đó, chúng ta gọi `ifeq()` truyền biến nhãn `notSales`. Điều đó có nghĩa là các chỉ thị tiếp theo của builder sẽ chỉ chạy nếu kết quả boolean trước đó của `invokevirtual()` trả về `true`. Ngược lại, chương trình sẽ nhảy đến liên kết `notSales` mà chúng ta sẽ định nghĩa sau.
*   Cuối cùng, nếu điều kiện của `ifeq()` trả về `true`, chúng ta tải tham số `baseSalary` bằng cách sử dụng `dload(1)`. Sau đó, chúng ta tải hằng số 0.35 vào stack toán hạng và sử dụng `dmul()` để nhân các toán hạng đã lưu. Cuối cùng, chúng ta trả về giá trị bằng cách sử dụng `dreturn()`.

Phần đầu tiên đó bao gồm câu lệnh `if` đầu tiên của phương thức chúng ta muốn tạo. Do đó, để tạo câu lệnh `if` thứ hai, chúng ta có thể thêm nhiều lời gọi phương thức vào `codeBuilder()` của mình, sau lời gọi `dreturn()`:

```java
  // ... sau .dreturn() của phần trước ...
  .labelBinding(notSales) // Điểm nhảy nếu không phải "sales"
  .aload(3)              // Tải tham số 'role' (slot 3)
  .ldc("engineer")       // Tải hằng số "engineer"
  .invokevirtual(stringClass, "equals", MethodTypeDesc.of(ClassDesc.of("Z"), stringClass)) // Gọi String.equals("engineer")
  .ifeq(notEngineer)     // Nếu kết quả = false (không phải "engineer"), nhảy đến nhãn 'notEngineer'
  .dload(1)              // Nếu kết quả = true (là "engineer"), tải tham số 'baseSalary' (slot 1)
  .ldc(0.25)             // Tải hằng số 0.25
  .dmul()                // Nhân baseSalary với 0.25
  .dreturn();             // Trả về kết quả (double)
```

`labelBinding(notSales)` chạy nếu biểu thức `ifeq(notSales)` trả về `false`. Các thao tác khác tương tự như những gì chúng ta đã trình bày trước đây để xử lý câu lệnh `if` đầu tiên.

Cuối cùng, chúng ta có thể thêm phần cuối cùng để xử lý giá trị trả về mặc định:

```java
  // ... sau .dreturn() của phần trước ...
  .labelBinding(notEngineer) // Điểm nhảy nếu không phải "engineer"
  .dload(1)                 // Tải tham số 'baseSalary' (slot 1)
  .ldc(0.15)                // Tải hằng số 0.15
  .dmul()                   // Nhân baseSalary với 0.15
  .dreturn();                // Trả về kết quả (double)
```

Điều tương tự xảy ra với việc gắn nhãn các nhánh, nhưng bây giờ là cho nhãn `notEngineer`. Phần cuối cùng này chạy nếu `ifeq(notEngineer)` trả về `false`.

Cuối cùng, để hoàn tất phương thức `generate()` của chúng ta, chúng ta cần định nghĩa đối tượng `ClassFile` và ghi nó ra một tệp `.class`:

```java
// ... sau phần codeBuilder ...
    }); // Kết thúc lambda của withCode
    // ... kết thúc lambda của calculateAnnualBonusBuilder

    var classBuilder = ClassFile.of()
      .build(ClassDesc.of("EmployeeSalaryCalculator"), // Tên class
        cb -> cb.withMethod("calculateAnnualBonus",    // Tên method
                            MethodTypeDesc.of(ClassDesc.of("D"), // Kiểu trả về: double
                                            ClassDesc.of("D"), // Tham số 1: double
                                            ClassDesc.of("Ljava/lang/String;")), // Tham số 2: String
                            AccessFlag.PUBLIC.mask(), // Access modifier: public
                            calculateAnnualBonusBuilder)); // Consumer chứa logic method

    Files.write(Path.of("EmployeeSalaryCalculator.class"), classBuilder);
} // Kết thúc phương thức generate()
```

Chúng ta đã sử dụng `ClassFile.of().build()` để khởi tạo một trình xây dựng tệp class, và truyền hai đối số cho nó. Đối số đầu tiên là tên class được bọc bên trong lời gọi `ClassDesc.of()`. Đối số thứ hai là một consumer `ClassBuilder` tạo class với các phương thức mong muốn. Để làm điều đó, chúng ta đã sử dụng `withMethod()` truyền tên phương thức, chữ ký phương thức (method signature), cờ truy cập (access flag) và trình xây dựng mã phương thức đã định nghĩa trước đó (`calculateAnnualBonusBuilder`).

Đáng chú ý, chúng ta đã định nghĩa chữ ký phương thức là `MethodTypeDesc.of(ClassDesc.of("D"), ClassDesc.of("D"), ClassDesc.of("Ljava/lang/String;"))`, có nghĩa là phương thức được tạo trả về kiểu `double` (được định nghĩa bởi tham số đầu tiên) và nhận một tham số kiểu `double` và một tham số kiểu `String`. (Lưu ý: `ClassDesc.of("D")` cho double và `ClassDesc.of("Ljava/lang/String;")` cho String Object).

Sau đó, chúng ta ghi mảng byte được lưu trữ trong biến `classBuilder` ra một tệp bằng cách sử dụng các bộ ghi của `Files`.

## 4. Biến Đổi Một Tệp Class Thành Tệp Khác (Transforming a Class-File Into Another)

Bây giờ, giả sử chúng ta muốn sao chép toàn bộ nội dung của một tệp class sang một tệp khác. Chúng ta có thể làm điều đó bằng cách sử dụng các phép biến đổi khác nhau:

```java
public static void transform() throws IOException {
    var basePath = Files.readAllBytes(Path.of("EmployeeSalaryCalculator.class")); // Đọc tệp class gốc

    // CodeTransform đơn giản chấp nhận tất cả CodeElement
    CodeTransform codeTransform = ClassFileBuilder::accept;

    // MethodTransform áp dụng codeTransform cho code của method
    MethodTransform methodTransform = MethodTransform.transformingCode(codeTransform);
    // ClassTransform áp dụng methodTransform cho các method của class
    ClassTransform classTransform = ClassTransform.transformingMethods(methodTransform);

    ClassFile classFile = ClassFile.of();
    // Phân tích tệp class gốc và áp dụng biến đổi
    byte[] transformedClass = classFile.transformClass(classFile.parse(basePath), classTransform);
    // Ghi tệp class đã biến đổi
    Files.write(Path.of("TransformedEmployeeSalaryCalculator.class"), transformedClass);
}
```

Trong ví dụ trên, trước tiên chúng ta đọc tệp class mà chúng ta đã tạo ở phần trước, `EmployeeSalaryCalculator.class`.

Sau đó, chúng ta định nghĩa một `CodeTransform` chấp nhận tất cả các `CodeElement` được định nghĩa trong class gốc. Hơn nữa, chúng ta tạo một `MethodTransform` sử dụng `codeTransform` và một `ClassTransform` sử dụng `methodTransform`. Cấu trúc lồng nhau như vậy giúp dễ dàng tổng quát hóa và sử dụng lại các transformer cho các mục đích khác nhau.

Các phép biến đổi mã (code transform) và phương thức (method transform) tùy chỉnh hơn có thể được định nghĩa bằng cách sử dụng các biểu thức lambda rõ ràng hơn. Chẳng hạn, chúng ta có thể định nghĩa một `MethodTransform` tùy chỉnh bằng cách sử dụng biểu thức lambda chỉ chấp nhận các phương thức có tên cụ thể:

```java
MethodTransform methodTransform = (methodBuilder, methodElement) -> {
    // Kiểm tra tên phương thức
    if (methodElement.header().name().stringValue().equals("calculateAnnualBonus")) {
        // Nếu tên phù hợp, tạo lại method với code gốc
        methodBuilder.withCode(codeBuilder -> {
            for (var codeElement: methodElement.code()) {
                codeBuilder.accept(codeElement); // Sao chép từng element code
            }
        });
    }
    // Nếu tên không phù hợp, không làm gì cả (hoặc có thể bỏ qua method)
};
```

Trong trường hợp trên, trước tiên chúng ta kiểm tra xem tên phương thức có bằng chuỗi ký tự `calculateAnnualBonus` hay không, sử dụng các phương thức `header()` và `name()`. Nếu có, chúng ta sử dụng `methodBuilder` để tạo một phương thức với chính xác các chỉ thị từ `methodElement` của class gốc.
```