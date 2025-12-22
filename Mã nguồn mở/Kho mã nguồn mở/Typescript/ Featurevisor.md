**Featurevisor** là một giải pháp quản lý tính năng (Feature Management) hiện đại, tập trung vào lập trình viên. Nó cho phép quản lý **Feature Flags** (bật/tắt tính năng), **A/B Testing** (thử nghiệm) và **Remote Config** (cấu hình từ xa) thông qua quy trình làm việc với Git (GitOps).

Dưới đây là giải thích chi tiết về công nghệ và các kĩ thuật chính mà dự án này sử dụng:

### 1. Công nghệ cốt lõi (Tech Stack)

*   **Ngôn ngữ lập trình:** **TypeScript** là ngôn ngữ chủ đạo (chiếm 98%). Việc sử dụng TypeScript giúp đảm bảo tính an toàn về kiểu dữ liệu (type-safety) cho cả CLI, Core và các SDK.
*   **Kiến trúc Monorepo:** Dự án sử dụng mô hình Monorepo để quản lý nhiều gói (packages) trong một kho lưu trữ duy nhất thông qua **Lerna** và **NPM Workspaces**. Các gói bao gồm:
    *   `@featurevisor/cli`: Công cụ dòng lệnh để khởi tạo, kiểm tra và build.
    *   `@featurevisor/core`: Thư viện lõi xử lý logic build dữ liệu.
    *   `@featurevisor/sdk`: Các bộ phát triển phần mềm cho nhiều nền tảng (JavaScript, React, Vue...).
*   **Công cụ Build & Bundle:** Sử dụng **Webpack** để đóng gói các thư viện và **ts-node** để thực thi TypeScript trực tiếp trong quá trình phát triển.
*   **Kiểm thử (Testing):** Sử dụng **Jest** để viết Unit Test, đảm bảo các logic về phân đoạn người dùng (segmentation) và phân bổ lưu lượng (bucketing) hoạt động chính xác.
*   **Tài liệu & Trang web:** Sử dụng **Next.js** để xây dựng trang tài liệu và trang web chính thức.

### 2. Các kĩ thuật và tư duy kiến trúc chính

#### A. Triết lý GitOps & Declarative Configuration
Thay vì sử dụng một giao diện web (UI) và lưu dữ liệu vào Database như các dịch vụ SaaS khác (LaunchDarkly, Split), Featurevisor yêu cầu lập trình viên định nghĩa các tính năng dưới dạng tệp **YAML** hoặc **JSON** ngay trong mã nguồn.
*   **Kĩ thuật:** Mọi thay đổi về cấu hình tính năng đều phải qua quy trình: *Tạo nhánh -> Pull Request -> Review -> Merge*. Điều này giúp kiểm soát phiên bản (version control) và minh bạch hóa lịch sử thay đổi.

#### B. Kiến trúc Static Datafile (Cloud-Native)
Đây là kĩ thuật đặc sắc nhất của Featurevisor. Thay vì ứng dụng phải gọi API liên tục đến server quản lý để kiểm tra xem tính năng có bật hay không, Featurevisor thực hiện bước "Build".
*   **Cơ chế:** CLI sẽ quét các tệp YAML và biên dịch chúng thành các tệp **JSON tĩnh (datafiles)**.
*   **Lợi ích:** Các tệp JSON này được đẩy lên CDN (như Cloudflare Pages hoặc AWS S3). Ứng dụng chỉ cần tải tệp JSON này một lần (hoặc định kỳ). Việc kiểm tra tính năng diễn ra hoàn toàn ở local (Client-side), giúp độ trễ (latency) gần như bằng 0 và không bị phụ thuộc vào sự sống còn của server quản lý.

#### C. Consistent Bucketing (Phân đoạn nhất quán)
Để thực hiện A/B testing hoặc triển khai tính năng theo tỷ lệ phần trăm (ví dụ: chỉ cho 10% người dùng thấy tính năng mới), Featurevisor sử dụng kĩ thuật **Hashing**.
*   **Kĩ thuật:** Sử dụng thuật toán băm (thường là **MurmurHash**) dựa trên `userId` hoặc `deviceId` cộng với khóa của tính năng. Kết quả băm sẽ cho ra một con số từ 0-100.
*   **Đặc điểm:** Kĩ thuật này đảm bảo tính nhất quán (Consistent). Cùng một người dùng sẽ luôn nhận được cùng một trải nghiệm (Variation) trên các thiết bị khác nhau mà không cần lưu trữ trạng thái người dùng trên server.

#### D. Phân đoạn người dùng nâng cao (Advanced Segmentation)
Featurevisor hỗ trợ các toán tử logic phức tạp để nhắm mục tiêu người dùng.
*   **Kĩ thuật:** Hỗ trợ các thuộc tính (Attributes) như phiên bản ứng dụng, quốc gia, thiết bị... và các toán tử `and`, `or`, `not`, `matches` (Regex). Điều này cho phép tạo ra các quy tắc triển khai cực kỳ chi tiết.

#### E. Khả năng mở rộng qua Plugin
Kiến trúc của CLI và Core cho phép người dùng viết thêm các Plugin tùy chỉnh.
*   **Kĩ thuật:** Cung cấp API `Datasource` để thay đổi cách đọc/ghi dữ liệu (không chỉ giới hạn ở File System mà có thể từ Database hoặc nguồn khác).

### Tóm tắt luồng hoạt động:
1.  **Định nghĩa:** Viết cấu hình tính năng bằng YAML.
2.  **Lint & Test:** Chạy `featurevisor lint` và `featurevisor test` để đảm bảo logic đúng.
3.  **Build:** Chạy `featurevisor build` tạo ra tệp JSON tĩnh.
4.  **Deploy:** Đẩy tệp JSON lên CDN qua CI/CD (GitHub Actions).
5.  **Evaluate:** SDK trong ứng dụng tải JSON và quyết định hiển thị tính năng cho người dùng cuối dựa trên `Context`.

Đây là một giải pháp tối ưu cho các hệ thống đòi hỏi hiệu suất cao, bảo mật (dữ liệu cấu hình không rời khỏi hạ tầng của bạn) và chi phí thấp (chỉ tốn tiền lưu trữ file tĩnh).