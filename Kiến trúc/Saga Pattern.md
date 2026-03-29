Dưới đây là nội dung bài viết về **Saga Pattern** đã được chuyển đổi thành định dạng **README.md**. File này được thiết kế để làm tài liệu kỹ thuật, giúp team dev nắm bắt nhanh các "ác mộng" và giải pháp khi triển khai Saga trên thực tế.

---

# 🛡️ Saga Pattern: Từ Tutorial đến "Ác mộng" Production

![Microservices](https://img.shields.io/badge/Architecture-Microservices-red)
![Transaction](https://img.shields.io/badge/Distributed-Transactions-blue)
![Status](https://img.shields.io/badge/Focus-Reliability%20%26%20Consistency-brightgreen)

Saga Pattern thường được ca tụng là "vị cứu tinh" cho giao dịch phân tán. Nhưng trên Production, nó là một "con thú dữ" với hàng tá edge-cases. Tài liệu này tổng hợp các kiến thức thực chiến để sống sót khi triển khai Saga.

---

## 📑 Mục lục
1. [Nỗi đau của Giao dịch phân tán](#1-nỗi-đau-của-giao-dịch-phân-tán)
2. [Saga Pattern là gì?](#2-saga-pattern-là-gì)
3. [Choreography vs Orchestration](#3-choreography-vs-orchestration)
4. [Vấn đề Isolation & "Rollback"](#4-vấn-đề-isolation--rollback)
5. [Tính Lũy đẳng (Idempotency)](#5-tính-lũy-đẳng-idempotency)
6. [Transactional Outbox & DLQ](#6-transactional-outbox--dlq)

---

## 1. Nỗi đau của Giao dịch phân tán
Trong kiến trúc **Database-per-service**, chúng ta không thể sử dụng lệnh `ROLLBACK` của SQL xuyên qua nhiều database khác nhau.
- **2PC (Two-Phase Commit):** Đảm bảo nhất quán nhưng giết chết hiệu năng và dễ gây deadlock.
- **Saga:** Chấp nhận **Eventual Consistency** (Nhất quán sau cùng) để đổi lấy khả năng mở rộng.

---

## 2. Saga Pattern là gì?
Saga chia nhỏ một giao dịch lớn thành một chuỗi các **Local Transactions**. 
- Nếu một bước thất bại, Saga kích hoạt các **Compensating Transactions** (Giao dịch bù trừ) để hoàn tác các bước trước đó.
- *Ví dụ:* Nếu trừ kho thành công nhưng thanh toán lỗi -> Chạy hàm cộng lại kho.

---

## 3. Choreography vs Orchestration
Việc lựa chọn mô hình quyết định độ phức tạp của hệ thống:

| Đặc điểm | Choreography (Múa ba-lê) | Orchestration (Nhạc trưởng) |
| :--- | :--- | :--- |
| **Cơ chế** | Các service tự lắng nghe & bắn Event. | Một bộ điều phối tập trung ra lệnh. |
| **Phù hợp** | Ít bước (2-3 bước). | Quy trình phức tạp (> 4 bước). |
| **Nhược điểm** | "Event Spaghetti" - Khó trace log. | Rủi ro "God Service" (nhét quá nhiều logic). |

> **⚠️ Lưu ý:** Orchestrator chỉ quản lý luồng (Workflow), KHÔNG chứa logic nghiệp vụ (Domain Logic).

---

## 4. Vấn đề Isolation & "Rollback"
Saga thiếu tính **Isolation** dẫn đến lỗi **Dirty Read** (Dữ liệu chưa chốt đã bị đọc).

### Giải pháp thực chiến:
1. **Semantic Lock (Khóa ngữ nghĩa):** Thay vì trừ thẳng kho, hãy dùng trạng thái `PENDING` hoặc `RESERVED`.
2. **Reordering Steps (Đảo trình tự):** Đưa những bước dễ lỗi nhất (như Payment) lên đầu để giảm thiểu việc phải chạy bù trừ.

---

## 5. Tính Lũy đẳng (Idempotency) & Retry
Mạng mẽo không hoàn hảo, hệ thống sẽ **Retry**. Nếu không có tính lũy đẳng, khách hàng sẽ bị trừ tiền nhiều lần.

### Kỹ thuật Idempotency Key:
- **Cơ chế:** Client gửi kèm một `Idempotency-Key` duy nhất.
- **Chống Race Condition:** Sử dụng Database Constraint (`UNIQUE INDEX`) trên bảng `idempotency_records`.
- **Luồng xử lý:**
    1. Nhận request -> INSERT key vào DB.
    2. Nếu trùng key -> Trả về kết quả cũ (Cached response).
    3. Nếu chưa trùng -> Thực hiện nghiệp vụ.

---

## 6. Transactional Outbox & DLQ

### 📦 Transactional Outbox Pattern
Giải quyết vấn đề: *Lưu DB thành công nhưng bắn Event sang Kafka/RabbitMQ thất bại.*
- Tạo bảng `Outbox` cùng Database với bảng nghiệp vụ.
- Dùng Local Transaction để lưu cả **Dữ liệu** và **Event** vào DB.
- Một tiến trình ngầm (Relay) sẽ quét bảng Outbox để đẩy Event đi.

### ☠️ Dead Letter Queue (DLQ)
Khi lệnh hoàn tác (Compensating) cũng thất bại sau N lần retry (do DB chết hẳn):
- Đẩy message vào **DLQ**.
- Kích hoạt Alert (Slack/SMS) để con người can thiệp thủ công.
- **Thông tin cần đính kèm trong DLQ:**
    - `Correlation ID` (Saga ID).
    - `Original Payload` (Dữ liệu gốc).
    - `Error Stacktrace` (Tại sao lỗi).
    - `Failed Step` (Lỗi ở bước nào).

---

## 🏁 Tổng kết: Không có "Viên đạn bạc"
Saga là sự đánh đổi khốc liệt. Để sống sót trên Production, bạn phải:
1. Thiết kế với tâm thế **"Mọi thứ đều có thể sập"**.
2. Luôn có cơ chế **Idempotency** và **Outbox**.
3. Đầu tư vào hệ thống **Tracing/Logging** (Correlation ID) để biết transaction đang kẹt ở đâu.

---
*Tài liệu dựa trên chia sẻ của Bách Nguyễn Ngọc @ Viblo.*