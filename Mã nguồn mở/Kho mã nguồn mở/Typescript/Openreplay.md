Dưới đây là bản phân tích chi tiết về kiến trúc, công nghệ và luồng hoạt động của dự án **OpenReplay** dựa trên cấu trúc kho lưu trữ mà bạn đã cung cấp.

---

# 🚀 OpenReplay: Phân tích Kiến trúc & Kỹ thuật Hệ thống

OpenReplay là một nền tảng **Session Replay** (ghi lại phiên làm việc) mã nguồn mở, tập trung vào quyền riêng tư và khả năng tự vận hành (self-hosting). Dự án được tổ chức theo mô hình **Monorepo** với sự kết hợp của nhiều ngôn ngữ lập trình tối ưu cho từng tác vụ cụ thể.

## 1. 🛠 Công nghệ cốt lõi (Core Stack)

Hệ thống được xây dựng trên sự kết hợp đa ngôn ngữ (Polyglot Programming) để tận dụng thế mạnh của từng nền tảng:

*   **Frontend (Dashboard):** React, TypeScript, Redux/MobX, Tailwind CSS. Sử dụng kiến trúc Store-heavy để quản lý trạng thái phức tạp khi xem lại session.
*   **Backend (API & Logic):** 
    *   **Python (Chalice/Flask):** Đảm nhận các API RESTful, quản lý người dùng, bản tin hàng tuần (weekly reports) và tích hợp bên thứ ba.
    *   **Go (Golang):** Dùng cho các service hiệu năng cao (như `backend/cmd/sink`, `ender`, `storage`) để xử lý luồng dữ liệu cực lớn từ tracker gửi về.
    *   **Node.js:** Xử lý các tác vụ thời gian thực (Real-time) như `assist` (hỗ trợ trực tuyến qua WebRTC/Websocket) và `sourcemap-reader`.
*   **Dữ liệu & Lưu trữ (Storage Layer):**
    *   **PostgreSQL:** Lưu trữ dữ liệu quan hệ (người dùng, dự án, thiết lập).
    *   **ClickHouse:** "Trái tim" của hệ thống phân tích. ClickHouse là OLAP database cực nhanh để truy vấn hàng tỷ sự kiện (events).
    *   **Redis:** Caching và quản lý trạng thái phiên làm việc tức thời.
    *   **Kafka:** Hệ thống hàng đợi tin nhắn (Message Queue) giúp điều phối dữ liệu giữa các microservices.
    *   **MinIO/S3:** Lưu trữ các tệp tin ghi lại phiên (session recordings) dưới dạng binary.

## 2. 🏗 Tư duy Kiến trúc & Kỹ thuật Hệ thống

### Kiến trúc Microservices hướng sự kiện (Event-Driven)
Dữ liệu từ Tracker không được ghi trực tiếp vào DB mà đi qua một chuỗi các worker:
1.  **Ingestion:** Dữ liệu nén gửi về `http` service.
2.  **Buffering:** Đưa vào Kafka để tránh mất dữ liệu khi tải cao.
3.  **Processing:** Các worker (viết bằng Go) đọc từ Kafka, phân tích (heuristics), giải mã và phân loại.
4.  **Storage:** Ghi vào ClickHouse (để phân tích) và S3 (để xem lại).

### Tư duy "Privacy First" & Self-hosting
Khác với LogRocket hay FullStory, OpenReplay cho phép triển khai hoàn toàn trên hạ tầng riêng (AWS, GCP, DigitalOcean hoặc On-premise) thông qua Kubernetes/Helm Charts, giúp dữ liệu nhạy cảm của người dùng không bao giờ rời khỏi server của doanh nghiệp.

### Tối ưu hiệu năng Tracker
Tracker được viết bằng TypeScript với kích thước cực nhỏ (~26KB), sử dụng **Web Workers** để mã hóa dữ liệu mà không gây lag cho giao diện người dùng (UI thread).

## 3. 🌟 Các kỹ thuật chính nổi bật

*   **DOM Snapshots & Mutations:** Thay vì quay phim màn hình (tốn băng thông), OpenReplay ghi lại trạng thái ban đầu của DOM và sau đó chỉ ghi lại các thay đổi (mutations). Khi xem lại, hệ thống dựng lại DOM y hệt trong một iframe.
*   **Binary Message Encoding:** Sử dụng một giao thức nhị phân riêng (định nghĩa trong thư mục `mobs`) để nén dữ liệu sự kiện trước khi gửi về server, giảm thiểu tối đa độ trễ mạng.
*   **Assist (Co-browsing):** Sử dụng WebRTC để truyền stream màn hình thời gian thực và Websocket để điều khiển chuột/bàn phím từ xa (Remote Control).
*   **Sourcemap Resolution:** Tự động ánh xạ lỗi JS từ code đã bị minify (obfuscated) về code gốc thông qua `sourcemap-reader`, giúp developer sửa lỗi nhanh chóng.
*   **Intelligent Search:** Tích hợp AI/LLM (trong thư mục `ee/intelligent_search`) giúp người dùng tìm kiếm session bằng ngôn ngữ tự nhiên.

---

## 🔄 Tóm tắt Luồng hoạt động (Project Flow)

Dưới đây là hành trình của một mẩu dữ liệu từ trình duyệt người dùng đến khi xuất hiện trên Dashboard của Developer:

### Bước 1: Thu thập (Capture)
*   **Tracker** (được nhúng vào web khách hàng) theo dõi mọi sự kiện: click, scroll, input, console logs, network requests và thay đổi DOM.
*   Dữ liệu được nén lại và gửi về **Service HTTP** (Go) theo định kỳ.

### Bước 2: Xử lý (Ingestion & Processing)
*   **Sink/Ender (Go):** Nhận dữ liệu, kiểm tra tính hợp lệ và đẩy vào Kafka.
*   **Heuristics (Go):** Phân tích các dấu hiệu "ức chế" của người dùng như *Rage Click* (click liên tục), *Dead Click*, hoặc lỗi hiệu năng (*Slow Page Load*).
*   **Storage Worker:** Đẩy dữ liệu thô vào **S3/MinIO** và dữ liệu sự kiện vào **ClickHouse**.

### Bước 3: Lưu trữ & Tổng hợp (Aggregating)
*   Các bản ghi quan hệ (metadata của session, user profile) được lưu vào **PostgreSQL**.
*   Các chỉ số Web Vitals (LCP, FCP, CLS) được tính toán và lưu vào **ClickHouse** để vẽ biểu đồ analytics.

### Bước 4: Xem lại (Replay & Analysis)
*   Khi Developer mở **Frontend Dashboard**:
    1.  API (Python) lấy thông tin Session từ PostgreSQL.
    2.  Hệ thống fetch dữ liệu binary từ S3.
    3.  **Player** (React) giải mã dữ liệu binary, dựng lại DOM và "phát lại" các mutation theo đúng mốc thời gian, kết hợp đồng bộ với Console Logs và Network calls trong DevTools giả lập.

---

## 📂 Sơ đồ cấu thư mục quan trọng
*   `/api`: Backend chính (Python/Chalice).
*   `/backend`: Các dịch vụ lõi xử lý dữ liệu (Go).
*   `/frontend`: Giao diện Dashboard (React/TS).
*   `/tracker`: Mã nguồn thư viện theo dõi phía Client.
*   `/assist`: Logic hỗ trợ trực tuyến (Node.js).
*   `/scripts/helmcharts`: Cấu hình triển khai Kubernetes.
*   `/ee`: (Enterprise Edition) Các tính năng nâng cao như SSO, SAML, AI Search.

OpenReplay là một ví dụ điển hình của việc kết hợp giữa **High-throughput Engineering** (Go/Kafka/ClickHouse) và **Complex Frontend State Management** (React/Player) để giải quyết bài toán Debugging ở quy mô lớn.