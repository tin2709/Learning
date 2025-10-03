

# 1 Google Drive View-Only PDF Downloader Script

[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## 📄 Mô tả

Script này cung cấp một giải pháp để tải xuống các file PDF chỉ xem (view-only) từ Google Drive. Nó hoạt động bằng cách "chụp" từng trang PDF được hiển thị trong trình duyệt dưới dạng hình ảnh, sau đó kết hợp chúng thành một file PDF mới với chất lượng độ phân giải tốt hơn so với việc chụp màn hình thủ công từng trang.

## 🚨 LƯU Ý QUAN TRỌNG VÀ MIỄN TRỪ TRÁCH NHIỆM

*   **Sử dụng cẩn thận:** Việc chạy các script JavaScript không rõ nguồn gốc trong console của trình duyệt luôn tiềm ẩn rủi ro bảo mật. Hãy đảm bảo bạn hiểu rõ chức năng của script trước khi thực thi.
*   Script này sử dụng thư viện `jsPDF` được tải từ `unpkg.com`, một CDN phổ biến và đáng tin cậy.
*   Chất lượng của file PDF được tạo ra phụ thuộc vào độ phân giải và cách trình duyệt hiển thị các trang PDF tại thời điểm chạy script.
*   Sử dụng script này một cách có trách nhiệm và tuân thủ các chính sách của Google Drive.

## 🚀 Cách hoạt động

1.  Script quét các phần tử `<img>` trên trang có nguồn gốc là `blob:https://drive.google.com/` (là cách Google Drive hiển thị từng trang PDF).
2.  Mỗi hình ảnh trang sẽ được vẽ lên một canvas.
3.  Thư viện `jsPDF` sẽ được sử dụng để tạo một tài liệu PDF mới, thêm từng hình ảnh từ canvas vào làm một trang riêng biệt.
4.  Cuối cùng, script sẽ tự động tải file PDF đã tạo xuống máy tính của bạn.

## 💡 Hướng dẫn sử dụng

Thực hiện theo các bước sau để tải xuống file PDF chỉ xem từ Google Drive:

### Bước 1: Mở file PDF trên Google Drive

1.  Mở file PDF chỉ xem của bạn trên Google Drive.
2.  Nếu bạn đang ở chế độ xem trước (Preview), hãy nhấp vào biểu tượng **ba chấm dọc** (menu) ở góc trên bên phải và chọn **"Mở trong cửa sổ mới" (Open in new window)**.

### Bước 2: Tải toàn bộ nội dung PDF

1.  **Cực kỳ quan trọng:** Trong cửa sổ trình duyệt đang hiển thị PDF, hãy **cuộn xuống cuối tài liệu PDF** để đảm bảo tất cả các trang đã được tải đầy đủ và hiển thị hoàn toàn trong trình duyệt. Script chỉ có thể xử lý những gì đã được tải.

### Bước 3: Mở Developer Console

1.  Trên trang PDF, mở Developer Console của trình duyệt bằng một trong các cách sau:
    *   **Windows / Linux:** Nhấn `F12` hoặc `Ctrl + Shift + I`.
    *   **macOS:** Nhấn `Cmd + Option + I`.
2.  Sau khi mở, hãy chuyển đến tab **`Console`**.

### Bước 4: Sao chép Script

Sao chép toàn bộ đoạn mã JavaScript sau:

```javascript
(function () {
  console.log("Loading script ...");

  let script = document.createElement("script");
  script.onload = function () {
    const { jsPDF } = window.jspdf;

    // Generate a PDF from images with "blob:" sources.
    let pdf = null;
    let imgElements = document.getElementsByTagName("img");
    let validImgs = [];
    let initPDF = true;

    console.log("Scanning content ...");
    for (let i = 0; i < imgElements.length; i++) {
      let img = imgElements[i];

      let checkURLString = "blob:https://drive.google.com/";
      if (img.src.substring(0, checkURLString.length) !== checkURLString) {
        continue;
      }

      validImgs.push(img);
    }

    console.log(`${validImgs.length} content found!`);
    console.log("Generating PDF file ...");
    for (let i = 0; i < validImgs.length; i++) {
      let img = validImgs[i];
      let canvasElement = document.createElement("canvas");
      let con = canvasElement.getContext("2d");
      canvasElement.width = img.naturalWidth;
      canvasElement.height = img.naturalHeight;
      con.drawImage(img, 0, 0, img.naturalWidth, img.naturalHeight);
      let imgData = canvasElement.toDataURL();

      let orientation;
      if (img.naturalWidth > img.naturalHeight) {
        orientation = "l"; // Landscape
      } else {
        orientation = "p"; // Portrait
      }

      let pageWidth = img.naturalWidth;
      let pageHeight = img.naturalHeight;

      if (initPDF) {
        pdf = new jsPDF({
          orientation: orientation,
          unit: "px",
          format: [pageWidth, pageHeight],
        });
        initPDF = false;
      }

      if (!initPDF) {
        pdf.addImage(imgData, "PNG", 0, 0, pageWidth, pageHeight, "", "SLOW");
        if (i !== validImgs.length - 1) {
          pdf.addPage();
        }
      }

      const percentages = Math.floor(((i + 1) / validImgs.length) * 100);
      console.log(`Processing content ${percentages}%`);
    }

    // Check if title contains .pdf in end of the title
    let title = document.querySelector('meta[itemprop="name"]').content;
    if (title.split(".").pop() !== "pdf") {
      title = title + ".pdf";
    }

    // Download the generated PDF.
    console.log("Downloading PDF file ...");
    pdf.save(title, { returnPromise: true }).then(() => {
      document.body.removeChild(script);
      console.log("PDF downloaded!");
    });
  };

  // Load the jsPDF library using the trusted URL.
  let scriptURL = "https://unpkg.com/jspdf@latest/dist/jspdf.umd.min.js";
  let trustedURL;
  if (window.trustedTypes && trustedTypes.createPolicy) {
    const policy = trustedTypes.createPolicy("myPolicy", {
      createScriptURL: (input) => {
        return input;
      },
    });
    trustedURL = policy.createScriptURL(scriptURL);
  } else {
    trustedURL = scriptURL;
  }

  script.src = trustedURL;
  document.body.appendChild(script);
})();
```

### Bước 5: Dán và Thực thi Script

1.  Dán đoạn mã đã sao chép vào tab `Console`.
2.  Nhấn phím `Enter` để thực thi script.

### Bước 6: Chờ đợi và Tải xuống

1.  Bạn sẽ thấy các thông báo tiến trình trong tab `Console` (ví dụ: "Scanning content...", "Processing content X%", "Downloading PDF file...").
2.  Khi quá trình hoàn tất, một file PDF sẽ tự động được tạo và tải xuống máy tính của bạn (thường là vào thư mục "Downloads" mặc định của trình duyệt).
3.  Tên file PDF sẽ được lấy từ tiêu đề của tài liệu trên Google Drive.
