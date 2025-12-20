

# 🛡️ Node.js Security Best Practices

> **"Bảo mật không phải là một tùy chọn, đó là trách nhiệm."**
> Tài liệu này tóm tắt 4 nguyên tắc bảo mật thiết yếu để bảo vệ ứng dụng Node.js khỏi các cuộc tấn công phổ biến và rủi ro rò rỉ dữ liệu.

---

## 📋 Mục lục
1. [Kiểm Soát Thư Viện Bên Thứ Ba](#1-kiểm-soát-thư-viện-bên-thứ-ba)
2. [Xử Lý Dữ Liệu Từ Người Dùng](#2-xử-lý-dữ-liệu-từ-người-dùng)
3. [Bảo Vệ Thông Tin Nhạy Cảm](#3-bảo-vệ-thông-tin-nhạy-cảm)
4. [Hạn Chế Tốc Độ Truy Cập (Rate Limiting)](#4-hạn-chế-tốc-độ-truy-cập)
5. [Danh sách kiểm tra (Checklist)](#-danh-sách-kiểm-tra-nhanh)

---

## 1. Kiểm Soát Thư Viện Bên Thứ Ba

Mã nguồn ứng dụng của bạn thường chứa hàng trăm thư viện từ npm. Một thư viện bị lỗi hoặc chứa mã độc (như vụ việc `event-stream`) có thể phá hủy toàn bộ hệ thống.

### Biện pháp thực hiện:
* **Kiểm tra lỗ hổng thường xuyên:**
  ```bash
  npm audit          # Phát hiện lỗ hổng
  npm audit fix      # Tự động vá các lỗi cơ bản
  ```
* **Luôn lưu trữ tệp khóa phiên bản (`lock file`):** Đảm bảo môi trường Production chạy chính xác phiên bản đã test ở Local.
  * Tệp cần commit: `package-lock.json`, `yarn.lock`, hoặc `pnpm-lock.yaml`.
* **Công cụ giám sát:**
  * [Snyk](https://snyk.io/): Quét lỗ hổng liên tục.
  * [Dependabot](https://github.com/dependabot): Tự động tạo Pull Request cập nhật thư viện an toàn.

---

## 2. Xử Lý Dữ Liệu Từ Người Dùng

Đừng bao giờ tin tưởng dữ liệu đến từ phía client. Việc thiếu kiểm tra sẽ dẫn đến SQL Injection, XSS và các kỹ thuật tấn công chèn mã.

### ✅ Cách thực hiện đúng:
* **Sử dụng câu lệnh có tham số (Parameterized Queries):**
  ```javascript
  // MySQL với mysql2 - TRÁNH cộng chuỗi trực tiếp
  const [rows] = await connection.execute(
    'SELECT * FROM users WHERE email = ?',
    [req.body.email]
  );
  ```
* **Kiểm tra và làm sạch dữ liệu (Validation & Sanitization):**
  Sử dụng thư viện `Joi` hoặc `validator.js`.
  ```javascript
  const Joi = require('joi');
  const schema = Joi.object({
    email: Joi.string().email().required(),
    username: Joi.string().alphanum().min(3).max(30).required()
  });
  ```
* **Sử dụng các Middleware bảo mật:**
  ```javascript
  const helmet = require('helmet');
  const xss = require('xss-clean');

  app.use(helmet()); // Thiết lập các HTTP headers bảo mật
  app.use(xss());    // Loại bỏ mã độc trong request
  ```

---

## 3. Bảo Vệ Thông Tin Nhạy Cảm

Việc để lộ API Keys, mật khẩu Database trên Git là sai lầm chết người.

### ✅ Cách thực hiện đúng:
* **Sử dụng biến môi trường:** Sử dụng thư viện `dotenv`.
* **Cấu hình `.gitignore`:** Tuyệt đối không đẩy tệp `.env` lên kho mã nguồn.
  ```text
  # .gitignore
  .env
  node_modules/
  ```
* **Sử dụng dịch vụ quản lý bí mật (Production):** AWS Secrets Manager, HashiCorp Vault, hoặc Doppler.
* **Xử lý khi bị lộ:** Nếu lỡ commit mã bí mật, hãy thu hồi (revoke) chúng ngay lập tức và sử dụng công cụ như `BFG Repo-Cleaner` để xóa lịch sử Git.

---

## 4. Hạn Chế Tốc Độ Truy Cập (Rate Limiting)

Nếu không giới hạn tốc độ, ứng dụng của bạn sẽ dễ dàng bị đánh sập bởi Brute Force hoặc DDoS.

### ✅ Cách thực hiện đúng:
Sử dụng `express-rate-limit` để giới hạn số lượng request từ một IP.

```javascript
const rateLimit = require('express-rate-limit');

// Giới hạn chung cho API
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 phút
  max: 100,                 // Tối đa 100 yêu cầu/IP
  message: 'Quá nhiều yêu cầu, vui lòng thử lại sau.'
});

app.use('/api/', apiLimiter);

// Giới hạn nghiêm ngặt cho chức năng Login
const loginLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, 
  max: 5, 
  message: 'Thử đăng nhập quá nhiều lần, hãy quay lại sau 1 giờ.'
});

app.post('/api/login', loginLimiter, (req, res) => { ... });
```

---

## 🚨 Danh Sách Kiểm Tra Nhanh (Checklist)

| STT | Hạng mục | Lệnh / Thư viện |
| :--- | :--- | :--- |
| 1 | Quét lỗ hổng thư viện | `npm audit` |
| 2 | Khóa phiên bản thư viện | `package-lock.json` |
| 3 | Bảo vệ HTTP Headers | `npm install helmet` |
| 4 | Chống XSS | `npm install xss-clean` |
| 5 | Kiểm tra dữ liệu đầu vào | `npm install joi validator` |
| 6 | Quản lý biến môi trường | `npm install dotenv` + `.gitignore` |
| 7 | Chống Brute Force | `npm install express-rate-limit` |
| 8 | Làm chậm tấn công | `npm install express-slow-down` |

---
