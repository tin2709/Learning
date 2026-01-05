Dưới đây là bản phân tích chi tiết về kiến trúc, kỹ thuật và luồng hoạt động của dự án **Amurex AI Meeting Copilot** dựa trên mã nguồn bạn đã cung cấp.

---

# 🚀 Phân tích Hệ thống Amurex AI Copilot

## 1. Công nghệ cốt lõi (Core Technologies)
*   **Ngôn ngữ:** JavaScript (ES6+), HTML5, CSS3.
*   **Nền tảng:** Chrome Extension Manifest V3 (Tiêu chuẩn mới nhất của Google).
*   **Giao tiếp thời gian thực:** WebSockets (`wss://`) để truyền tải transcript và nhận gợi ý AI ngay lập tức.
*   **Theo dõi DOM:** `MutationObserver` API để phát hiện thay đổi trong giao diện cuộc họp (phụ đề).
*   **Lưu trữ:** `chrome.storage.local` & `chrome.storage.sync` để quản lý dữ liệu phiên họp và cài đặt người dùng.
*   **Giao diện người dùng:** `chrome.sidePanel` API mang lại trải nghiệm sidebar hiện đại, không làm gián đoạn cửa sổ chính.

## 2. Kỹ thuật và Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của Amurex được xây dựng theo mô hình **Event-Driven (Hướng sự kiện)** và **Decoupled (Tách biệt thành phần)**:

*   **Content Scripts (`content.js`, `msteams_content.js`):** Đóng vai trò là "Cảm biến". Chúng được tiêm vào trang web (Google Meet/Teams) để quan sát DOM, trích xuất văn bản phụ đề (captions) và tên người nói.
*   **Background Service Worker (`background.js`):** Đóng vai trò "Bộ não điều phối". Nó quản lý vòng đời của extension, xử lý xác thực qua Cookie, điều hướng giữa các Sidepanel và thực hiện các tác vụ nặng như tải xuống tệp tin.
*   **Side Panel UI:** Thành phần tương tác trực tiếp với người dùng. Nó tách biệt hoàn toàn với logic quét dữ liệu, giúp UI mượt mà ngay cả khi dữ liệu transcript đang đổ về liên tục.
*   **Backend Integration:** Sử dụng kết hợp REST API (cho các tác vụ tĩnh như lấy tóm tắt) và WebSockets (cho các tác vụ động như gợi ý câu hỏi thời gian thực).

## 3. Các kỹ thuật chính (Key Techniques)
*   **DOM Scraping thông minh:** Thay vì quét toàn bộ trang, code sử dụng các CSS Selectors đặc hiệu (ví dụ: `.a4cQT` trong GMeet) kết hợp với `MutationObserver` để chỉ bắt các thay đổi trong phần phụ đề, giúp tối ưu hiệu năng.
*   **Cơ chế Đệm (Buffering):** Sử dụng các biến buffer (`transcriptTextBuffer`, `personNameBuffer`) để gom nhóm các đoạn hội thoại ngắn của cùng một người nói trước khi gửi lên Server, tránh spam request.
*   **Đồng bộ xác thực (Auth Sync):** Sử dụng `chrome.cookies` để đọc session từ trang web chính (`app.amurex.ai`), giúp người dùng chỉ cần đăng nhập một nơi.
*   **Xử lý đa nền tảng:** Code tách biệt logic cho Google Meet và MS Teams nhưng vẫn dùng chung một cấu trúc dữ liệu transcript, giúp backend dễ xử lý.

---

## 4. Tóm tắt luồng hoạt động (Project Workflow)

### Bước 1: Khởi tạo (Startup)
1. Người dùng vào Google Meet/MS Teams.
2. `content.js` kiểm tra trạng thái hệ thống và xác thực người dùng.
3. Extension tự động kích hoạt tính năng **Captions (Phụ đề)** của nền tảng họp (vì đây là nguồn dữ liệu chính).

### Bước 2: Thu thập dữ liệu (Capture)
1. `MutationObserver` theo dõi các node phụ đề mới xuất hiện.
2. Khi có người nói, code lấy: **Tên người nói, Nội dung, Dấu thời gian**.
3. Dữ liệu này được lưu tạm vào `chrome.storage` và đồng thời đẩy qua **WebSocket** lên Server AI.

### Bước 3: Xử lý & Tương tác thời gian thực (Real-time Interaction)
1. Server AI nhận transcript, phân tích ngữ cảnh.
2. Server gửi lại các "Gợi ý thông minh" (Smart Suggestions) qua WebSocket.
3. Người dùng mở **Side Panel**, xem các câu hỏi gợi ý hoặc tóm tắt nhanh nếu vào họp muộn (Late Join Recap).

### Bước 4: Kết thúc & Tổng hợp (Finalization)
1. Khi nhấn nút "Kết thúc cuộc họp", extension sẽ:
    *   Ngắt các bộ quan sát (Observers).
    *   Gửi toàn bộ transcript cuối cùng lên Backend qua API `/end_meeting`.
    *   Yêu cầu AI tạo **Summary** (Tóm tắt) và **Action Items** (Việc cần làm).
2. Side Panel hiển thị kết quả cuối cùng, cho phép người dùng:
    *   Chỉnh sửa tóm tắt.
    *   Sao chép vào Clipboard.
    *   Gửi Email tóm tắt cho các thành viên tham gia.
    *   Tải transcript về máy (.txt).

---

# Amurex Meeting Copilot - Trợ lý AI cho công việc

Amurex là một tiện ích mở rộng Chrome mã nguồn mở, đóng vai trò như một người bạn đồng hành vô hình trong các cuộc họp. Hệ thống tự động ghi chép, tóm tắt và đưa ra gợi ý thông minh dựa trên nội dung hội thoại thực tế.

### 🌟 Tính năng chính
*   **Ghi chép thời gian thực:** Tự động chuyển lời nói thành văn bản trên Google Meet và MS Teams.
*   **Gợi ý thông minh:** Đưa ra các câu hỏi hoặc ý tưởng thảo luận ngay trong lúc họp.
*   **Tóm tắt quyền năng:** Tự động trích xuất các ý chính và hành động cần thực hiện (Action Items) sau cuộc họp.
*   **Bắt kịp nội dung:** Tính năng "Late Join Recap" giúp bạn hiểu ngay những gì đã thảo luận nếu vào họp muộn.
*   **Gửi Email tự động:** Gửi tóm tắt cuộc họp cho đồng nghiệp chỉ với một cú click.

### 🛠 Cài đặt cho nhà phát triển
1. **Clone Repo:** `git clone https://github.com/thepersonalaicompany/amurex.git`
2. **Cấu hình:** Tạo tệp `config.js` trong thư mục `extension/` với các thông số Backend URL của bạn.
3. **Cài đặt Extension:**
    *   Mở Chrome, truy cập `chrome://extensions/`.
    *   Bật "Developer mode".
    *   Chọn "Load unpacked" và trỏ tới thư mục `extension/`.
4. **Backend:** Cần chạy kèm với [Amurex Backend](https://github.com/thepersonalaicompany/amurex-backend).

### 🔒 Bảo mật & Riêng tư
Amurex ưu tiên sự minh bạch. Mọi dữ liệu transcript đều được xử lý theo cấu hình của bạn, hỗ trợ tự lưu trữ (Self-hosting) để đảm bảo quyền riêng tư tuyệt đối cho doanh nghiệp.

---
*Made with ❤️ by The Personal AI Company.*