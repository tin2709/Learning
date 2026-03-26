Dưới đây là nội dung bài viết được chuyển đổi thành định dạng **README.md** chuyên nghiệp, giúp bạn lưu trữ kiến thức hoặc đưa vào tài liệu dự án để cảnh báo đội ngũ phát triển về lỗi phân biệt hoa thường (Case-Sensitive).

---

# 🛑 Cơn Ác Mộng Naming Case-Sensitive: Tại sao Docker chạy trên Windows "ngon" nhưng lên Linux lại lỗi?

![OS Difference](https://img.shields.io/badge/OS-Windows%20vs%20Linux-blue) ![Docker](https://img.shields.io/badge/Docker-Troubleshooting-2496ED?logo=docker) ![NodeJS](https://img.shields.io/badge/Node.js-Best%20Practices-339933?logo=node.js)

Bạn đã bao giờ gặp tình huống: Code chạy mượt mà trên máy local (Windows/macOS), nhưng vừa đóng gói vào Docker hoặc deploy lên server Linux là hệ thống báo lỗi `Module not found` chưa?

Thủ phạm thường không nằm ở Logic code, mà nằm ở một "lời nói dối" ngọt ngào của hệ điều hành.

## 🧠 1. Sự khác biệt về Triết lý Hệ điều hành (Kernel Level)

Mọi rắc rối bắt đầu từ cách các hệ điều hành quản lý **File System** (Hệ thống tệp tin):

| Hệ điều hành | File System | Đặc tính | Ví dụ |
| :--- | :--- | :--- | :--- |
| **Windows** | NTFS | **Case-Insensitive** | `User.service.ts` và `user.service.ts` là **MỘT**. |
| **Linux** | Ext4/XFS | **Case-Sensitive** | `User.ts` và `user.ts` là **HAI** tệp tin riêng biệt. |

## 🤥 2. "Cú lừa" từ Docker Desktop trên Local

Docker trên Windows/macOS không chạy trực tiếp trên nhân OS đó mà thông qua một máy ảo Linux siêu nhẹ (Linux VM).

**Kịch bản lỗi:**
1. Bạn đặt tên file vật lý là: `UserMapper.ts`.
2. Bạn viết code import: `import { UserMapper } from './usermapper'`.
3. **Trên Windows:** Hệ điều hành bảo: *"Ok, tôi hiểu bạn muốn tìm UserMapper, tôi sẽ đưa nó cho bạn"*.
4. **Trên Linux (Production/CI):** Khi Docker quét tệp tin, nó bảo: *"Tôi không thấy file nào tên là `usermapper` (viết thường) cả!"*.

**Kết quả:** Hệ thống sập ngay tại bước khởi tạo (Runtime Error hoặc Build Error).

## 💉 3. "Vaccine" cho lỗi Naming trong phát triển dự án

Đừng chỉ dựa vào sự cẩn thận của con người. Hãy dùng công cụ để "ép" mọi thứ vào khuôn khổ.

### Lớp 1: ESLint Rule (Chặn từ trong trứng nước)
Sử dụng plugin `eslint-plugin-import` để báo lỗi ngay trên VS Code nếu path bạn import không khớp 100% với tên file thực tế.

```json
// .eslintrc.json
{
  "rules": {
    "import/no-unresolved": "error"
  }
}
```

### Lớp 2: CI/CD Pipeline (Chốt chặn cuối cùng)
Tận dụng **GitHub Actions** chạy trên môi trường Ubuntu. Nếu bạn lỡ tay đặt tên sai, Pipeline sẽ "fail" ngay lập tức khi build hoặc chạy test, không cho phép code lỗi lọt xuống Production.

### Lớp 3: Standard Naming Convention
Luôn tuân thủ quy tắc đặt tên thống nhất cho toàn bộ dự án:
- **kebab-case:** `user-controller.ts`
- **PascalCase:** `UserMapper.ts`

---

## 🛠 Giải pháp sẵn có (Boilerplate)

Nếu bạn muốn một bộ khung Node.js đã được cấu hình sẵn để "miễn nhiễm" với các lỗi môi trường này (kèm theo Kafka, Redis, Clean Architecture), hãy tham khảo dự án:

- **GitHub:** [nodejs-quickstart-structure](https://github.com/paudang/nodejs-quickstart-structure)
- **NPM:** `npx nodejs-quickstart-structure init`

## 📝 Lời kết
Môi trường Linux không tạo ra lỗi, nó chỉ giúp bạn nhìn thấy những lỗi mà Windows đã che giấu. Việc đồng bộ môi trường từ Local đến Production là yếu tố then chốt của một **Senior Developer**.

---
*Nội dung được chia sẻ bởi **paudang** trên Viblo.*