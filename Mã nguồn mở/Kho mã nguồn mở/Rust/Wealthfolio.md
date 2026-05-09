Dựa trên cấu trúc thư mục và nội dung mã nguồn của dự án **Wealthfolio**, dưới đây là bản phân tích chuyên sâu về hệ thống quản lý tài chính cá nhân này.

---

### 1. Phân tích Công nghệ Cốt lõi (Core Stack)

Wealthfolio được xây dựng với triết lý **"Local-first"** (ưu tiên dữ liệu cục bộ), sử dụng các công nghệ hiện đại nhất để đảm bảo hiệu suất và quyền riêng tư.

*   **Ngôn ngữ & Runtime:**
    *   **Rust:** Sử dụng cho toàn bộ logic lõi (Core), quản lý dữ liệu và backend. Lựa chọn này đảm bảo tốc độ tính toán tài chính phức tạp và an toàn bộ nhớ.
    *   **TypeScript/React:** Sử dụng cho giao diện người dùng (Frontend).
*   **Framework đa nền tảng:**
    *   **Tauri (v2):** Đây là điểm đặc sắc nhất. Tauri giúp Wealthfolio chạy như một ứng dụng Desktop (Windows, macOS, Linux) và Mobile (iOS/Android) nhưng có kích thước cực nhẹ vì sử dụng Webview hệ thống và backend bằng Rust.
*   **Cơ sở dữ liệu:**
    *   **SQLite + Diesel ORM:** Dữ liệu lưu trữ trong một tệp `.db` duy nhất trên máy người dùng. Diesel giúp truy vấn SQL trong Rust một cách an toàn (Type-safe).
*   **Web Server (Web Mode):**
    *   **Axum:** Một framework web hiệu năng cao của hệ sinh thái Rust được sử dụng khi chạy ở chế độ Server/Docker.
*   **Giao diện (UI/UX):**
    *   **Tailwind CSS v4 & shadcn/ui:** Cung cấp giao diện hiện đại, tối giản.
    *   **Recharts:** Thư viện đồ thị mạnh mẽ để trực quan hóa biến động tài sản.

---

### 2. Tư duy Kiến trúc (Architectural Mindset)

Wealthfolio áp dụng kiến trúc **Adapter & Modular** cực kỳ linh hoạt:

1.  **Hệ thống Adapter (Tauri vs Web):**
    *   Trong `apps/frontend/src/adapters/`, dự án tách biệt cách ứng dụng giao tiếp với backend.
    *   Nếu chạy **Desktop**, nó dùng `Tauri IPC` (gọi trực tiếp hàm Rust).
    *   Nếu chạy **Web/Docker**, nó dùng `Axum REST API`.
    *   *Tư duy:* Điều này cho phép một codebase Frontend duy nhất chạy trên mọi môi trường mà không cần sửa đổi logic hiển thị.
2.  **Addon System (Hệ sinh thái mở):**
    *   Wealthfolio có một hệ thống Plugin (Addons) rất chuyên nghiệp. Các addon như `goal-progress-tracker` hay `investment-fees-tracker` được phát triển bằng TypeScript SDK riêng (`@wealthfolio/addon-sdk`).
    *   *Tư duy:* Tách nhỏ tính năng để người dùng tùy biến, tránh làm "nặng" ứng dụng lõi (fighting feature creep).
3.  **Local-First & E2EE Sync:**
    *   Dữ liệu mặc định không lên mây. Khi cần đồng bộ giữa các thiết bị (Wealthfolio Connect), hệ thống sử dụng **End-to-End Encryption (E2EE)** (xem `crates/device-sync/`). Chỉ người dùng mới có khóa giải mã.

---

### 3. Kỹ thuật Lập trình Đặc sắc (Distinctive Techniques)

*   **AI Tool-calling Architecture:** Trong `crates/ai/src/tools/`, hệ thống định nghĩa các "Tools" cho AI (như `accounts.rs`, `holdings.rs`). Khi người dùng chat với trợ lý AI, LLM có thể "gọi" các hàm này để lấy dữ liệu tài chính thực tế và thực hiện phân tích.
*   **Financial Math in Rust:** Các phép tính lợi nhuận (FIFO/LIFO), quy đổi tỷ giá (FX) và tính toán Net Worth được thực hiện ở tầng `crates/core/`. Việc đưa logic toán học vào Rust giúp xử lý hàng ngàn giao dịch trong tích tắc.
*   **Keyring Security:** Bí mật (như API Key của các sàn) không lưu vào file text hay SQLite thuần túy mà được đẩy vào **Keyring hệ thống** (Keychain trên macOS, Credential Manager trên Windows) thông qua Rust.
*   **Zod-to-Rust Validation:** Sử dụng Zod ở Frontend và các struct tương ứng trong Rust để đảm bảo dữ liệu giao dịch luôn khớp nhau 100% giữa hai tầng.

---

### 4. Luồng Hoạt động Hệ thống (System Flow)

1.  **Khởi động:** 
    *   Backend Rust khởi tạo SQLite, chạy Migration (tạo bảng).
    *   Frontend load các cấu hình từ `SettingsService`.
2.  **Nhập dữ liệu (Ingestion):**
    *   Người dùng upload CSV -> Frontend dùng `csv_parser` -> Chuyển đổi qua Adapter -> Rust thực hiện kiểm tra trùng lặp (Idempotency) -> Lưu vào DB.
3.  **Đồng bộ dữ liệu thị trường (Market Sync):**
    *   `MarketDataService` gọi các Provider (Yahoo Finance, Alpha Vantage) -> Rust tính toán lại giá trị Portfolio -> Phát sự kiện (Domain Events) -> Frontend cập nhật đồ thị thời gian thực thông qua `React Query`.
4.  **Xử lý Addon:**
    *   Runtime của ứng dụng quét thư mục `addons/` -> Khởi tạo một Sandbox an toàn -> Addon đăng ký các mục vào Sidebar và Route của ứng dụng chính.

---

### 5. Đánh giá Tổng quan

**Ưu điểm:**
*   **Sạch sẽ (Clean Code):** Phân chia crate trong Rust rất rõ ràng (core, storage, market-data, connect).
*   **Bảo mật vượt trội:** Không có khái niệm "Server-side" tập trung, giảm thiểu rủi ro rò rỉ dữ liệu quy mô lớn.
*   **Tính mở:** Hệ thống Addon cho phép cộng đồng đóng góp tính năng mà không làm hỏng lõi ứng dụng.

**Nhược điểm:**
*   **Phức tạp trong Build:** Do kết hợp quá nhiều công nghệ (Rust + Node + Tauri), việc cài đặt môi trường phát triển (Prerequisites) khá nặng nề đối với người mới.

**Kết luận:** Wealthfolio là hình mẫu lý tưởng cho một ứng dụng **Desktop-Web hybrid** hiện đại. Nó kết hợp được sức mạnh hiệu năng của Rust với sự linh hoạt của React, tạo ra một công cụ tài chính chuyên nghiệp, an toàn và cực kỳ thẩm mỹ.