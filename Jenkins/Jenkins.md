

# 🚀 Jenkins Pipeline: 10 Nguyên Tắc Vàng (Best Practices)

Tài liệu này tóm tắt các quy tắc thiết yếu để xây dựng hệ thống CI/CD bằng Jenkins Pipeline hiệu quả, an toàn và dễ bảo trì.

---

## 🏗️ 1. Lưu trữ Pipeline trong SCM (Git)
*   **Nguyên tắc:** Đừng bao giờ viết code Pipeline trực tiếp trên giao diện (UI) của Jenkins. Hãy lưu tệp `Jenkinsfile` vào hệ thống quản lý mã nguồn (Git).
*   **Lợi ích:** Có lịch sử thay đổi (audit trail), dễ dàng phối hợp thông qua Pull Request và quản lý phiên bản chuyên nghiệp.

## 📝 2. Ưu tiên cú pháp Declarative Pipeline
*   **Nguyên tắc:** Sử dụng cú pháp **Declarative** (có cấu trúc `pipeline { ... }`) thay vì Scripted Pipeline (Groovy tự do).
*   **Lợi ích:** Dễ đọc hơn, ít lỗi hơn và hỗ trợ các tính năng hiện đại như `matrix build`.

## 📚 3. Sử dụng Shared Libraries (Thư viện dùng chung)
*   **Nguyên tắc:** Khi thấy mình dùng thẻ `script` quá nhiều hoặc lặp lại code ở nhiều dự án, hãy tách chúng ra thành **Shared Library**.
*   **Lợi ích:** Tái sử dụng mã nguồn, giữ cho `Jenkinsfile` gọn gàng và dễ quản lý tập trung.

## 🛠️ 4. Đừng coi Shared Library là một dự án phần mềm đa năng
*   **Nguyên tắc:** Shared Library chỉ nên dùng cho các tác vụ CI/CD. Đừng viết những đoạn mã lập trình phức tạp không liên quan đến CI.
*   **Lưu ý:** Code quá phức tạp chạy trên Jenkins Controller có thể làm treo hoặc giảm hiệu năng của toàn bộ hệ thống Jenkins.

## 🪜 5. Thứ tự ưu tiên khi viết Pipeline
Hãy tuân thủ quy trình đưa ra quyết định sau:
1.  Bắt đầu với **Declarative Pipeline**.
2.  Nếu cần xử lý logic phức tạp hơn, hãy dùng **Shared Library**.
3.  Chỉ dùng **Scripted Pipeline** khi cả hai phương án trên không thể giải quyết được (ví dụ: cần tính toán động việc chọn Agent).

## 🛑 6. Đừng đặt lệnh `input` bên trong khối `agent`
*   **Sai:** Đợi người dùng nhấn "Confirm" khi đang giữ một Agent.
*   **Đúng:** Đặt lệnh `input` bên ngoài `agent`.
*   **Giải thích:** Agent là tài nguyên đắt đỏ (vùng nhớ, CPU, executor). Việc bắt Agent ngồi chờ con người phê duyệt là cực kỳ lãng phí.

```groovy
// CÁCH LÀM ĐÚNG
stage('Chờ phê duyệt') {
    steps {
        input "Triển khai lên Production?"
    }
}
stage('Deploy') {
    agent { label 'linux' }
    steps { sh 'echo Deploying...' }
}
```

## ⏱️ 7. Luôn bao bọc `input` trong `timeout`
*   **Nguyên tắc:** Bất kỳ bước chờ đợi nào cũng phải có thời gian hết hạn (timeout).
*   **Lợi ích:** Tránh việc Pipeline bị treo vô thời hạn nếu không có ai vào phê duyệt, giúp dọn dẹp tài nguyên hệ thống.

## 💻 8. Mọi công việc nặng phải chạy trên Agent
*   **Nguyên tắc:** Mọi tác vụ như build code, chạy test, quét bảo mật phải dùng lệnh `sh` hoặc `bat` để đẩy việc sang **Agent**.
*   **Lưu ý:** Tuyệt đối không chạy code logic nặng trực tiếp trên Jenkins Controller. Controller chỉ đóng vai trò điều phối.

## ⚡ 9. Lấy Agent trong các bước song song (Parallel)
*   **Nguyên tắc:** Nếu bạn có nhiều tác vụ chạy song song (`parallel`), hãy yêu cầu cấp Agent bên trong các nhánh đó. Điều này giúp tận dụng tối đa sức mạnh của nhiều máy Agent cùng lúc.

## 🛡️ 10. Tránh phê duyệt Script (Script Security)
*   **Nguyên tắc:** Hạn chế tối đa việc phải vào màn hình "In-process Script Approval" để nhấn "Approve" cho các dòng code nguy hiểm.
*   **Lời khuyên:** Nếu code của bạn bị Jenkins chặn vì lý do bảo mật, hãy tìm cách viết lại code đó an toàn hơn thay vì chỉ đơn giản là nhấn nút phê duyệt.

---

## 💡 Kết luận: Lộ trình tối ưu
> **Declarative** ➔ **Shared Library** ➔ **Scripted** (Trường hợp cuối cùng)

Bằng cách tuân thủ các quy tắc này, hệ thống Jenkins của bạn sẽ hoạt động ổn định, bảo mật và dễ dàng mở rộng cho hàng trăm dự án khác nhau.

