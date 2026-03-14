Dựa trên mã nguồn và tài liệu của dự án **GEO-SEO Claude Code Skill**, dưới đây là bản phân tích chi tiết về công nghệ, kiến trúc và quy trình vận hành của hệ thống này:

### 1. Công nghệ cốt lõi (Core Technologies)
Dự án được xây dựng như một tiện ích mở rộng (Skill) cho **Claude Code** (CLI của Anthropic), kết hợp sức mạnh của AI với các công cụ lập trình truyền thống:

*   **Ngôn ngữ lập trình:** **Python (87%)** đóng vai trò xử lý logic nặng (phân tích, cào dữ liệu, tính toán điểm số). **Shell (13%)** dùng cho các script cài đặt và quản lý môi trường.
*   **Thư viện xử lý dữ liệu:** 
    *   `BeautifulSoup4` & `lxml`: Để trích xuất và phân tích cấu trúc HTML.
    *   `Requests`: Để tương tác HTTP và kiểm tra Robots.txt.
    *   `Playwright`: (Tùy chọn) Để render JavaScript và chụp ảnh màn hình trang web (quan trọng vì AI crawlers thường không đọc được trang web chỉ có JS).
*   **Xử lý văn bản & Định dạng:** 
    *   `Markdown`: Ngôn ngữ chính để định nghĩa "Skills" và tạo báo cáo.
    *   `ReportLab`: Công cụ mạnh mẽ để tạo báo cáo PDF chuyên nghiệp với biểu đồ và sơ đồ từ Python.
    *   `JSON-LD`: Định dạng dữ liệu có cấu trúc (Schema) mà dự án tạo ra để giúp AI hiểu thực thể website.

### 2. Tư duy Kiến trúc (Architectural Thinking)
Kiến trúc của dự án chuyển dịch từ SEO truyền thống sang **GEO (Generative Engine Optimization)** với các trụ cột:

*   **Kiến trúc Orchestrator-Subagent (Điều phối - Đại lý con):** 
    *   Thành phần trung tâm (`geo/SKILL.md`) đóng vai trò là "bộ não" điều phối. 
    *   Khi nhận lệnh, nó chia nhỏ nhiệm vụ và chuyển giao cho 5 đại lý chuyên biệt (Subagents) chạy song song: AI Visibility, Platform Analysis, Technical SEO, Content Quality, và Schema.
*   **Thiết kế Modular (Mô-đun hóa):** Chia nhỏ 11 sub-skills (trong thư mục `skills/`). Mỗi mô-đun giải quyết một bài toán cụ thể (như quét Robots.txt hoặc tính điểm Citability). Điều này cho phép bảo trì và nâng cấp từng phần mà không ảnh hưởng đến toàn bộ hệ thống.
*   **Scoring-as-a-Service (Tính điểm như một dịch vụ):** Hệ thống không chỉ đưa ra nhận xét định tính mà còn lượng hóa chất lượng GEO qua bộ chỉ số (0-100). Cách tiếp cận này giúp người dùng dễ dàng theo dõi tiến độ tối ưu hóa.

### 3. Các kỹ thuật chính (Key Techniques)
*   **Citability Scoring (Tính điểm khả năng trích dẫn):** Đây là kỹ thuật đặc thù của GEO. Hệ thống phân tích các đoạn văn bản dựa trên nghiên cứu khoa học (Princeton/Georgia Tech), đánh giá xem một đoạn văn có độ dài tối ưu (134-167 từ), chứa dữ kiện thống kê và câu trả lời trực tiếp hay không.
*   **Brand Authority Correlation (Tương quan quyền uy thương hiệu):** Dự án tập trung quét các nền tảng có độ tương quan cao với AI (Reddit, YouTube, Wikipedia) thay vì chỉ tập trung vào Backlink truyền thống.
*   **E-E-A-T Automation:** Tự động kiểm tra các tín hiệu về Kinh nghiệm, Chuyên môn, Thẩm quyền và Độ tin cậy thông qua việc phân tích trang "About", thông tin tác giả và sự nhất quán của thực thể (Entity).
*   **Schema Generation:** Tự động tạo mã JSON-LD tối ưu dựa trên các mẫu (templates) có sẵn trong thư mục `schema/`, giúp website "giao tiếp" hiệu quả hơn với các mô hình ngôn ngữ lớn (LLMs).

### 4. Tóm tắt luồng hoạt động (Workflow Summary)
Quy trình khi người dùng chạy lệnh `/geo audit <url>` diễn ra như sau:

1.  **Giai đoạn Khám phá (Discovery):** 
    *   Fetch trang chủ, nhận diện loại hình kinh doanh (SaaS, E-commerce, Local...).
    *   Cào Sitemap hoặc duyệt liên kết nội bộ để xác định 50 trang quan trọng nhất.
2.  **Giai đoạn Phân tích song song (Parallel Analysis):** 
    *   5 Subagents được kích hoạt đồng thời để đánh giá: Kỹ thuật, Nội dung, Dữ liệu cấu trúc, Khả năng hiển thị trên các nền tảng AI (ChatGPT, Perplexity...) và Quyền uy thương hiệu.
3.  **Giai đoạn Tổng hợp (Synthesis):** 
    *   Thu thập báo cáo từ các Subagents.
    *   Tính toán điểm GEO tổng hợp dựa trên trọng số (AI Citability chiếm tỷ trọng cao nhất - 25%).
4.  **Giai đoạn Xuất bản (Reporting):** 
    *   Tạo file Markdown (`GEO-AUDIT-REPORT.md`) chứa kế hoạch hành động ưu tiên.
    *   (Tùy chọn) Chuyển đổi toàn bộ dữ liệu thành báo cáo PDF chuyên nghiệp gửi cho khách hàng.

**Kết luận:** Dự án này là một bộ công cụ "chiến đấu" thực tế cho kỷ nguyên AI Search, biến các chiến lược SEO phức tạp thành các bước hành động có thể lập trình và thực thi tự động thông qua giao diện hội thoại của AI.