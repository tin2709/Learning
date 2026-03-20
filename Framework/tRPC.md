Dưới đây là bản **README.md** được thiết kế chuyên nghiệp, tóm tắt toàn bộ sức mạnh và cách tiếp cận của **tRPC** cho hệ sinh thái TypeScript.

---

# 🚀 tRPC: End-to-End Type-Safe APIs for TypeScript

![TypeScript](https://img.shields.io/badge/Language-TypeScript-blue)
![tRPC](https://img.shields.io/badge/Framework-tRPC-cyan)
![Zod](https://img.shields.io/badge/Validation-Zod-purple)
![DX](https://img.shields.io/badge/Focus-Developer_Experience-brightgreen)

**tRPC (TypeScript Remote Procedure Call)** là giải pháp API thế hệ mới, cho phép bạn xây dựng các ứng dụng Full-stack với sự an toàn về kiểu dữ liệu (Type-safety) tuyệt đối từ Client đến Server mà không cần Schema Generation hay Runtime Bloat.

---

## 📖 Mục lục
1. [Triết lý tRPC](#-triết-lý-trpc)
2. [Tại sao tRPC là một bước đột phá?](#-tại-sao-trpc-là-một-bước-đột-phá)
3. [So sánh: REST vs GraphQL vs tRPC](#-so-sánh-rest-vs-graphql-vs-trpc)
4. [Cơ chế hoạt động](#-cơ-chế-hoạt-động)
5. [Ưu điểm vượt trội](#-ưu-điểm-vượt-trội)
6. [Khi nào nên sử dụng?](#-khi-nào-nên-sử-dụng)
7. [Hạn chế cần lưu ý](#-hạn-chế-cần-lưu-ý)

---

## 💡 Triết lý tRPC
tRPC xóa bỏ ranh giới giữa Frontend và Backend. Thay vì coi API là các "Endpoint" rời rạc, tRPC coi chúng là các **Hàm (Procedures)**. Bạn định nghĩa hàm ở Server, và gọi chúng ở Client với sự hỗ trợ đầy đủ của TypeScript Inference.

---

## ⚡ Tại sao tRPC là một bước đột phá?
*   **Không cần Code Generation:** Khác với GraphQL (cần generate types), tRPC sử dụng sức mạnh của TypeScript `infer` để tự động đồng bộ kiểu dữ liệu.
*   **Refactoring an toàn:** Nếu bạn đổi tên một tham số ở Backend, IDE sẽ báo lỗi ngay lập tức tại tất cả các vị trí sử dụng ở Frontend.
*   **Autocomplete tuyệt vời:** IntelliSense sẽ gợi ý chính xác các hàm, tham số đầu vào và kiểu dữ liệu trả về ngay khi bạn gõ code.

---

## 📊 So sánh: REST vs GraphQL vs tRPC

| Đặc điểm | REST API | GraphQL | tRPC |
| :--- | :--- | :--- | :--- |
| **Type Safety** | Thủ công (Swagger/OpenAPI) | Cần Code Gen (Apollo/Relay) | **Tự động hoàn toàn** |
| **Hỗ trợ đa ngôn ngữ** | Tốt nhất (Tất cả) | Tốt (Tất cả) | Chỉ dành cho TypeScript |
| **Complexity** | Thấp | Cao | **Rất thấp** |
| **Over-fetching** | Thường xuyên | Tránh được | Tránh được |
| **Serialization Latency** | ~0.3ms | ~0.5ms | **~0.2ms** |

---

## 🏗 Cơ chế hoạt động

tRPC hoạt động dựa trên cấu trúc **RPC** thay vì tài nguyên (Resources) như REST.

1.  **Router:** Tập hợp các Procedures (Query/Mutation).
2.  **Procedure:** 
    *   `Query`: Dùng để lấy dữ liệu (tương đương GET).
    *   `Mutation`: Dùng để thay đổi dữ liệu (tương đương POST/PUT/DELETE).
3.  **Validation:** Tích hợp sâu với **Zod** hoặc **Yup** để validate dữ liệu đầu vào ngay tại runtime và cung cấp kiểu dữ liệu cho compile-time.

---

## 🚀 Ưu điểm vượt trội
*   **Zero Boilerplate:** Không có build step phức tạp, không cần file schema riêng biệt.
*   **Framework Agnostic:** Hoạt động mượt mà với Next.js, React, Express, Fastify, AWS Lambda...
*   **Tốc độ phát triển:** Giảm 30-50% thời gian phát triển nhờ loại bỏ việc định nghĩa thủ công các API contracts.
*   **Small Footprint:** Không có dependencies nặng nề, client-side cực kỳ nhẹ.

---

## 🎯 Khi nào nên sử dụng?

### ✅ Nên dùng khi:
*   Dự án sử dụng **TypeScript** cho cả Frontend và Backend (Monorepo là lý tưởng nhất).
*   Đang phát triển các ứng dụng Next.js, Nuxt.js hoặc các Full-stack Frameworks.
*   Xây dựng Internal Tools, Dashboard hoặc Admin Panel cần tốc độ ship feature nhanh.
*   Ưu tiên sự an toàn về kiểu dữ liệu và dễ dàng refactor.

### ❌ Không nên dùng khi:
*   Backend sử dụng ngôn ngữ khác (Python, Go, Java...).
*   Cần expose Public API cho bên thứ ba (Third-party developers).
*   Cần sự tách biệt hoàn toàn giữa các team không dùng chung codebase/ngôn ngữ.

---

## ⚠️ Hạn chế cần lưu ý
*   **Tight Coupling:** Client và Server phụ thuộc chặt chẽ vào nhau (đây vừa là ưu điểm về tốc độ, vừa là nhược điểm về tính độc lập).
*   **Hệ sinh thái:** Dù đang phát triển nhanh nhưng vẫn nhỏ hơn so với REST/GraphQL đã tồn tại lâu đời.
*   **TypeScript-only:** Rào cản lớn nhất là bạn không thể sử dụng tRPC nếu một trong hai đầu (client/server) không dùng TypeScript.

---

## 🏁 Kết luận
tRPC không thay thế REST hay GraphQL trong mọi trường hợp, nhưng đối với các team **Full-stack TypeScript**, đây là công cụ tối ưu nhất để tăng tốc độ phát triển và giảm thiểu bug runtime.

---
*Tài liệu hướng dẫn hiện đại hóa quy trình phát triển API với tRPC.*