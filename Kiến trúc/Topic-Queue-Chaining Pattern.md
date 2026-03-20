Dưới đây là bản **README.md** chuyên nghiệp, tóm tắt kiến trúc **Topic-Queue-Chaining Pattern** dành cho các kỹ sư giải pháp và nhà phát triển hệ thống Serverless.

---

# 🏗️ Topic-Queue-Chaining Pattern: Kiến trúc Serverless Bền vững

![Architecture](https://img.shields.io/badge/Architecture-Event--Driven-blue)
![Reliability](https://img.shields.io/badge/Reliability-High-brightgreen)
![Scaling](https://img.shields.io/badge/Scaling-Elastic-orange)

Tài liệu này trình bày về **Topic-Queue-Chaining Pattern** — một giải pháp kiến trúc cốt lõi giúp hệ thống Microservices/Serverless đảm bảo tính toàn vẹn dữ liệu, khả năng chịu lỗi và khả năng mở rộng linh hoạt.

---

## 😟 Vấn đề: Hạn chế của Pub-Sub truyền thống

Trong mô hình Publish-Subscribe (Pub-Sub) thuần túy:
1.  **Publisher** gửi message đến một **Topic**.
2.  **Subscribers** nhận message trực tiếp từ Topic đó.

**Rủi ro:** Nếu một Subscriber Service bị lỗi, crash hoặc tạm dừng để bảo trì, mọi message được gửi đến trong khoảng thời gian đó sẽ **bị mất vĩnh viễn**. Hệ thống không có cơ chế lưu trữ đệm để xử lý lại (replay) các sự kiện đã lỡ.

---

## 💡 Giải pháp: Topic-Queue-Chaining Pattern

Pattern này đề xuất việc đặt một **Hàng đợi (Queue)** ở giữa Topic và mỗi Subscriber Service.

### Luồng hoạt động (Architecture Flow):
```text
[ Publisher Service ]
       |
       ▼
  [ Message Topic ] (e.g., SNS, EventBridge)
       |
       ├─────▶ [ Queue A ] ──▶ [ Subscriber Service 1 ]
       |
       ├─────▶ [ Queue B ] ──▶ [ Subscriber Service 2 ]
       |
       └─────▶ [ Queue C ] ──▶ [ Subscriber Service 3 ]
```

1.  **Publisher** phát sự kiện lên Topic chung.
2.  Topic thực hiện **Fan-out** (phân phối) message đến các Queue riêng biệt của từng Subscriber.
3.  **Subscriber Service** đọc và xử lý message từ Queue của chính nó theo nhịp độ (pace) phù hợp.

---

## 🚀 Lợi ích vượt trội

*   **Độ bền vững (Durability):** Message được lưu trữ bền vững trong Queue. Nếu Subscriber die, message vẫn nằm đó chờ cho đến khi service quay trở lại.
*   **Làm phẳng tải (Load Leveling):** Queue đóng vai trò bộ đệm (buffer), giúp hệ thống không bị "ngộp" khi có lượng traffic tăng đột biến (spikes).
*   **Khả năng mở rộng (Elastic Scaling):** Dễ dàng scale-out Subscriber Service theo chiều ngang. Nhiều consumer có thể cùng đọc từ một Queue để tăng tốc độ xử lý.
*   **Cô lập lỗi (Fault Isolation):** Lỗi của một Subscriber không ảnh hưởng đến các Subscriber khác hoặc phía Publisher.

---

## 🛒 Ứng dụng thực tế: Hệ thống E-commerce

Khi khách hàng nhấn nút **"Đặt hàng"**:
1.  **Basket Service** gửi sự kiện `OrderPlaced` lên EventBridge.
2.  **Topic-Queue-Chaining** phân phối sự kiện này vào 4 Queue độc lập:
    *   **Inventory Queue:** Cập nhật kho hàng.
    *   **Payment Queue:** Xử lý thanh toán.
    *   **Email Queue:** Gửi mail xác nhận cho khách.
    *   **Shipping Queue:** Chuyển thông tin cho bên vận chuyển.

> **Kết quả:** Ngay cả khi cổng thanh toán (Payment Gateway) bị bảo trì, các đơn hàng vẫn được tiếp nhận an toàn trong Queue và sẽ được xử lý ngay sau khi cổng thanh toán hoạt động trở lại.

---

## 🛠️ Triển khai trên AWS Serverless

Sự kết hợp hoàn hảo giữa các dịch vụ quản lý (Managed Services):

| Thành phần | Dịch vụ đề xuất | Vai trò |
| :--- | :--- | :--- |
| **Topic** | **Amazon SNS** hoặc **EventBridge** | Tiếp nhận và phân phối sự kiện toàn hệ thống. |
| **Queue** | **Amazon SQS** | Lưu trữ message bền vững, hỗ trợ Dead Letter Queue (DLQ). |
| **Compute** | **AWS Lambda** | Consumer xử lý logic nghiệp vụ từ Queue. |
| **IaC** | **AWS SAM** hoặc **CDK** | Định nghĩa hạ tầng dưới dạng code (Infrastructure as Code). |

---

## 🔗 Kết hợp các Pattern bổ trợ

*   **Fan-Out Pattern:** Một sự kiện kích hoạt nhiều luồng xử lý song song.
*   **Message Filtering:** Subscriber chỉ nhận các message thỏa mãn điều kiện cụ thể (ví dụ: chỉ nhận đơn hàng có giá trị > $1000).
*   **Dead Letter Queue (DLQ):** Chuyển các message lỗi nhiều lần sang một Queue riêng để kiểm tra và xử lý thủ công (Debugging).

---

## 🏁 Kết luận

**Topic-Queue-Chaining** không chỉ là một kỹ thuật, mà là một tư duy thiết kế hệ thống **Resilient** (Khả năng hồi phục). Trong kỷ nguyên Cloud-native, việc áp dụng pattern này là tiêu chuẩn bắt buộc để xây dựng các hệ thống Microservices ổn định và tin cậy ở quy mô lớn.

---
*Tài liệu hướng dẫn thiết kế kiến trúc hệ thống chịu lỗi cao (High Availability).*