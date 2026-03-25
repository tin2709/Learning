Dưới đây là nội dung bài viết được chuyển đổi sang định dạng `README.md` chuyên nghiệp, tối ưu cho việc lưu trữ và tra cứu kỹ thuật.

---

# Tối ưu Hiệu năng Spring Boot: Giảm 80% Tài Nguyên Sử Dụng

![Spring Boot](https://img.shields.io/badge/Framework-Spring--Boot-brightgreen)
![Java Version](https://img.shields.io/badge/Java-17%2B-orange)
![Performance](https://img.shields.io/badge/Performance-Optimized-red)

Tối ưu hóa Spring Boot không chỉ là về code, mà là một quá trình mang tính hệ thống từ cấu hình JVM, cú pháp ngôn ngữ đến kiến trúc xử lý. Hướng dẫn này tổng hợp các kỹ thuật hiện đại giúp ứng dụng chạy nhanh hơn và tiết kiệm chi phí hạ tầng cloud.

## 📖 Mục lục
- [Cấu hình JVM & Container](#1-cấu-hình-jvm--container)
- [Cú pháp Java hiện đại](#2-cú-pháp-java-hiện-đại)
- [Garbage Collection & Native Image](#3-garbage-collection--native-image)
- [Kiến trúc bất đồng bộ](#4-kiến-trúc-bất-đồng-bộ)
- [Kết luận](#kết-luận)

---

## 1. Cấu hình JVM & Container

Đừng để JVM sử dụng cấu hình mặc định trên Cloud. Việc giới hạn tài nguyên rõ ràng giúp tránh tình trạng lãng phí bộ nhớ và giảm hóa đơn hàng tháng.

### Tinh chỉnh tham số khởi chạy
Sử dụng `-XX:+UseContainerSupport` để JVM nhận diện chính xác giới hạn của Docker/K8s.

```bash
export JAVA_OPTS="-Xms512m -Xmx1024m -XX:+UseContainerSupport -XX:ParallelGCThreads=2 -XX:MaxMetaspaceSize=256m"
java $JAVA_OPTS -jar app-service.jar
```

### Giới hạn Tomcat Threads
Mặc định 200 threads là quá lớn cho các dịch vụ vừa và nhỏ. Hãy cấu hình lại trong `application.yaml`:

```yaml
server:
  tomcat:
    threads:
      max: 60
```

---

## 2. Cú pháp Java hiện đại (Java 17+)

Sử dụng tính năng mới giúp giảm boilerplate code và tối ưu hóa cách dữ liệu được xử lý trong bộ nhớ.

### Record (Thay thế DTO truyền thống)
Loại bỏ hàng chục dòng code getter/setter, constructor bằng 1 dòng duy nhất:
```java
public record UserResponse(Long id, String nickname, String email) {}
```

### Text Blocks & Switch Expression
Giúp code SQL hoặc JSON sạch sẽ và logic Switch an toàn hơn:
```java
// Text Block
String sql = """
    SELECT * FROM product_info
    WHERE category = 'ELECTRONICS'
    AND stock > 0
    """;

// Switch Expression
String categoryName = switch (typeCode) {
    case 1 -> "Điện tử";
    case 2 -> "Gia dụng";
    default -> "Khác";
};
```

### Pattern Matching (Java 17+)
Loại bỏ việc ép kiểu (casting) thủ công:
```java
static String handleEvent(Object event) {
    return switch (event) {
        case OrderEvent o -> "Mã đơn hàng: " + o.id();
        case UserEvent u -> "Tên người dùng: " + u.name();
        case null -> "Sự kiện rỗng";
        default -> "Sự kiện không xác định";
    };
}
```

---

## 3. Garbage Collection & Native Image

### ZGC (Z Garbage Collector)
Đối với ứng dụng yêu cầu độ trễ cực thấp (Pause time < 1ms), hãy kích hoạt ZGC:
```bash
-XX:+UseZGC
```

### GraalVM Native Image
Chuyển đổi Spring Boot thành file thực thi native để đạt hiệu suất tối đa:
- **Thời gian khởi động:** Giảm từ vài giây xuống mili giây.
- **Bộ nhớ:** Giảm tới **80%**.

**Lệnh build:**
```bash
./mvnw native:compile -Pnative
```

---

## 4. Kiến trúc bất đồng bộ

Tuyệt đối không xử lý các tác vụ nặng (Gửi email, xuất PDF, xử lý ảnh) trực tiếp trong Web Thread (Controller). 

**Giải pháp:** Đẩy tác vụ vào Message Queue để giải phóng luồng xử lý API.

```java
@PostMapping("/submit-report")
public ResponseEntity<String> handleReport(@RequestBody ReportConfig config) {
    taskQueue.send("report_gen_topic", config);
    return ResponseEntity.accepted().body("Đã bắt đầu tạo báo cáo. Vui lòng kiểm tra sau.");
}
```

---

## ✅ Kết luận

Bằng cách kết hợp **Java hiện đại**, **tinh chỉnh JVM** và **kiến trúc hướng sự kiện**, bạn có thể:
1. Giảm **80%** tài nguyên tiêu thụ.
2. Giảm chi phí vận hành Cloud đáng kể.
3. Tăng trải nghiệm người dùng thông qua giảm độ trễ (Latency).

---
*Nội dung được tóm tắt từ bài chia sẻ của **Lamri Abdellah Ramdane** trên Viblo.*