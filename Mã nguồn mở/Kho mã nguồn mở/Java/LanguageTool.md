Dựa trên cấu trúc thư mục và các tệp tin của dự án **LanguageTool**, đây là phân tích chuyên sâu về công nghệ, kiến trúc và luồng hoạt động của hệ thống kiểm tra ngữ pháp mã nguồn mở hàng đầu này.

### 1. Phân tích Công nghệ Cốt lõi (Core Tech Stack)

LanguageTool là một hệ thống Java thuần túy nhưng tích hợp rất nhiều thư viện NLP (Xử lý ngôn ngữ tự nhiên) mạnh mẽ:

*   **Ngôn ngữ lập trình chính:** **Java (91.5%)**, yêu cầu Java 17 trở lên. Sử dụng Maven để quản lý dự án đa mô-đun (multi-module).
*   **NLP Engines:**
    *   **Morfologik:** Sử dụng cấu trúc dữ liệu Finite State Automata (FSA) để lưu trữ từ điển và gán nhãn từ loại (POS tagging) cực kỳ tiết kiệm bộ nhớ.
    *   **Hunspell:** Tích hợp qua JNA để kiểm tra chính tả truyền thống.
    *   **FastText:** Sử dụng cho việc nhận dạng ngôn ngữ tự động (Language Identification).
    *   **Lucene:** Sử dụng để xây dựng và truy vấn các mô hình ngôn ngữ N-gram (tần suất xuất hiện của cụm từ) nhằm phát hiện các lỗi sai ngữ cảnh (ví dụ: "their" vs "there").
*   **Giao thức:** Hỗ trợ cả **HTTP REST API** (phổ biến nhất) và **gRPC** cho các tác vụ hiệu suất cao hoặc xử lý từ xa.
*   **Kiến trúc nạp quy tắc:** Sử dụng XML để định nghĩa các mẫu lỗi (Pattern rules), cho phép cộng đồng đóng góp quy tắc mà không cần viết code Java.

### 2. Tư duy Kiến trúc (Architectural Philosophy)

Kiến trúc của LanguageTool được thiết kế theo hướng **Modularity (Mô-đun hóa)** và **Extensibility (Khả năng mở rộng)**:

*   **Tách biệt lõi và ngôn ngữ:** Mô-đun `languagetool-core` chứa engine xử lý văn bản chung, trong khi mỗi ngôn ngữ (en, de, fr, vi...) nằm trong mô-đun riêng dưới `languagetool-language-modules`. Điều này cho phép nạp/ngắt các ngôn ngữ một cách linh hoạt.
*   **Mô hình Pipeline xử lý:** Văn bản được chuyển đổi qua nhiều trạng thái: Văn bản thô -> Văn bản có chú thích (AnnotatedText) -> Câu được phân tích (AnalyzedSentence).
*   **Hệ thống quy tắc phân tầng:**
    *   **Spelling Rules:** Dựa trên từ điển.
    *   **Pattern Rules:** Dựa trên các mẫu thẻ từ loại và token (XML).
    *   **Java Rules:** Các logic phức tạp không thể biểu diễn qua XML (ví dụ: kiểm tra sự hòa hợp chủ ngữ - động từ phức tạp).

### 3. Kỹ thuật Lập trình Đặc sắc (Unique Techniques)

*   **Finite State Transducers (FST):** Việc sử dụng thư viện Morfologik giúp dự án có thể tra cứu hàng triệu từ và nhãn POS của chúng chỉ trong vài mili giây với lượng RAM tối thiểu.
*   **POS Disambiguation (Khử nhập nhằng từ loại):** Dự án có các lớp `Disambiguator`. Ví dụ, từ "can" có thể là danh từ hoặc động từ. Disambiguator sẽ dựa vào ngữ cảnh (từ đứng trước/sau) để loại bỏ nhãn sai trước khi chạy quy tắc ngữ pháp.
*   **XML Pattern Matching Engine:** Hệ thống có một bộ máy thực thi quy tắc XML rất mạnh mẽ, hỗ trợ các thuộc tính như `postag`, `negate_pos`, `inflected="yes"`, và các biểu thức chính quy (Regex).
*   **Cơ chế "Premium" Off/On:** Kiến trúc hỗ trợ nạp các quy tắc nâng cao (Premium) một cách có điều kiện thông qua cấu hình `PremiumOff.java`, cho phép phân tách giữa bản cộng đồng và bản thương mại.

### 4. Luồng Hoạt động Hệ thống (System Workflows)

#### A. Luồng xử lý văn bản (Checking Pipeline)
1.  **Input:** Người dùng gửi văn bản (qua CLI, GUI, hoặc API).
2.  **Language Detection:** Hệ thống tự động nhận diện ngôn ngữ nếu không được chỉ định.
3.  **Sentence Tokenization:** Chia văn bản thành các câu (`SentenceTokenizer`).
4.  **Word Tokenization:** Chia câu thành các từ/token (`WordTokenizer`).
5.  **Tagging:** Engine Morfologik gán mọi nhãn từ loại có thể có cho mỗi token.
6.  **Disambiguation:** Khử nhập nhằng POS dựa trên ngữ cảnh câu.
7.  **Chunking:** Nhóm các từ thành cụm từ (Noun Phrase, Verb Phrase).
8.  **Rule Matching:** Chạy song song hàng nghìn quy tắc (XML và Java) trên câu đã phân tích.
9.  **Result Aggregation:** Tập hợp các đối tượng `RuleMatch`, bao gồm vị trí lỗi, thông điệp và gợi ý sửa lỗi.

#### B. Luồng gRPC/Remote Rules
Hệ thống có khả năng gọi các server ML (Machine Learning) từ xa qua gRPC để thực hiện các kiểm tra phức tạp hơn như xếp hạng gợi ý (Ranking suggestions) bằng BERT hoặc các mô hình Deep Learning khác.

### Tổng kết
LanguageTool là một hệ thống **Hybrid (Lai)** hoàn hảo: Nó kết hợp sức mạnh của **Rule-based** (độ chính xác cao, dễ giải thích lỗi) và **Statistical/AI** (nhận diện ngôn ngữ, n-grams). Đây là một kiến trúc mẫu mực cho các ứng dụng xử lý ngôn ngữ đòi hỏi hỗ trợ đa ngôn ngữ trên quy mô lớn.