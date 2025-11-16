

### I. Các lỗi phổ biến nhất: CS0234, CS0246, CS0400, CS1068, CS1069, CS1070

Đây là các lỗi cơ bản nhất và thường xuyên nhất, báo rằng trình biên dịch không thể tìm thấy một kiểu (class, interface, struct) hoặc một namespace.

*   **CS0234: The type or namespace name 'X' does not exist in the namespace 'Y' (are you missing an assembly reference?)**
    *   Ví dụ: `'Data' does not exist in 'ELearningProj'`
*   **CS0246: The type or namespace name 'X' could not be found (are you missing a using directive or an assembly reference?)**
    *   Ví dụ: `'ApplicationDbContext' could not be found`
*   **CS0400: The type or namespace name 'X' could not be found in the global namespace (are you missing an assembly reference?)**
*   **CS1068, CS1069, CS1070:** Các biến thể của lỗi không tìm thấy kiểu/namespace, thường kèm theo gợi ý về việc "type has been forwarded to another assembly" hoặc "consider adding a reference to that assembly."

**Nguyên nhân chính cho nhóm lỗi này:**

1.  **Thiếu `using` directive:** Bạn đang sử dụng một kiểu (ví dụ: `ApplicationDbContext`) mà namespace của nó (ví dụ: `ELearningProj.API.Data`) chưa được khai báo bằng câu lệnh `using` ở đầu file.
2.  **Sai tên namespace:** Bạn đã gõ sai tên namespace trong câu lệnh `using` hoặc tên namespace của lớp bạn muốn sử dụng (ví dụ: `ApplicationDbContext` có namespace `ELearningProj.Data` nhưng bạn lại `using ELearningProj.API.Data`).
3.  **Thiếu Project Reference:** Dự án hiện tại của bạn cần sử dụng một lớp từ một dự án C# khác (ví dụ: dự án `Models` của bạn) nhưng bạn chưa thêm tham chiếu (reference) đến dự án đó.
4.  **Thiếu Package Reference (NuGet):** Dự án của bạn cần một thư viện bên ngoài (NuGet package) nhưng bạn chưa cài đặt nó (ví dụ: cần `Swashbuckle.AspNetCore` cho Swagger, hoặc `Microsoft.EntityFrameworkCore` cho `DbContext`).
5.  **Sai tên file/thư mục:** Tên file không khớp với tên lớp bên trong, hoặc file không nằm trong thư mục mà namespace của nó ngụ ý.

**Cách sửa:**

1.  **Kiểm tra `using` directives:**
    *   Đối với mỗi kiểu dữ liệu bị lỗi, xác định namespace đầy đủ của nó.
    *   Thêm `using [Full.Namespace.Name];` vào đầu file code.
    *   **Ví dụ:** Nếu lỗi `ApplicationDbContext could not be found`, và `ApplicationDbContext` nằm trong namespace `ELearningProj.API.Data`, hãy thêm `using ELearningProj.API.Data;`
2.  **Kiểm tra tên Namespace của các file bị lỗi:**
    *   Mở file định nghĩa của kiểu bị lỗi (ví dụ: `ApplicationDbContext.cs`).
    *   Đảm bảo `namespace [TênNamespaceCủaFile];` khớp với những gì bạn đang `using` hoặc `namespace` của file hiện tại đang tham chiếu.
    *   **Ví dụ:** Trong `ApplicationDbContext.cs`, đảm bảo `namespace ELearningProj.API.Data` là đúng. Trong `WordsController.cs`, đảm bảo `namespace ELearningProj.API.Controllers` là đúng.
3.  **Kiểm tra Package References (NuGet):**
    *   Đối với các kiểu được cung cấp bởi thư viện bên ngoài (ví dụ: `DbContext`, `ApiController`, `[HttpGet]`), đảm bảo bạn đã cài đặt gói NuGet tương ứng.
    *   **Trong Terminal (trong thư mục `.csproj`):** `dotnet add package [PackageName]`
    *   **Các gói phổ biến cho dự án của bạn:**
        *   `Microsoft.EntityFrameworkCore` (cho `DbContext`, `DbSet`)
        *   `Microsoft.AspNetCore.Mvc.Core` (cho `ControllerBase`, `[ApiController]`, `[HttpGet]`, `IActionResult`)
        *   `Microsoft.AspNetCore.Mvc` (bao gồm `Microsoft.AspNetCore.Mvc.Core`)
        *   `Pomelo.EntityFrameworkCore.MySql` (cho `UseMySql`)
        *   `Swashbuckle.AspNetCore` (cho Swagger)
    *   **Kiểm tra `.csproj`:** Mở file `.csproj` và đảm bảo các thẻ `<PackageReference Include="..." Version="..." />` cho các gói này tồn tại.
4.  **Kiểm tra Project References (nếu có nhiều dự án C#):**
    *   Nếu bạn có một dự án riêng cho Models và một dự án riêng cho Data, bạn cần thêm tham chiếu giữa chúng. (Tuy nhiên, trong dự án hiện tại của bạn, Models và Data đều nằm trong một dự án API duy nhất, nên thường không phải là vấn đề này).
    *   **Trong Terminal (trong thư mục `.csproj` của dự án cần tham chiếu):** `dotnet add reference [PathToOtherProject.csproj]`
5.  **Clean và Build lại:** Sau mỗi thay đổi, hãy chạy `dotnet clean` rồi `dotnet build` để đảm buộc trình biên dịch làm mới trạng thái.

### II. Lỗi CS0012, CS1714, CS7068, CS7069, CS7071, CS7079, CS8090, CS8203

Những lỗi này thường liên quan đến các vấn đề nghiêm trọng hơn với các tham chiếu assembly:

*   **CS0012: The type 'type' is defined in an assembly that is not referenced. You must add a reference to assembly 'assembly'.**
*   **CS1714: The base class or interface of this type could not be resolved or is invalid.**
*   **CS7068, CS7069, CS7071, CS7079, CS8090, CS8203:** Các biến thể của lỗi không thể tìm thấy/giải quyết assembly được tham chiếu, hoặc assembly bị lỗi.

**Nguyên nhân chính:**

1.  **Tham chiếu đến một assembly không tồn tại:** Có thể bạn đã xóa một dự án/thư viện hoặc đổi tên nó nhưng tham chiếu vẫn còn trong `.csproj`.
2.  **Assembly bị hỏng hoặc không tương thích:** Phiên bản của một gói NuGet không tương thích với phiên bản .NET hoặc các gói khác.
3.  **Lỗi khi khôi phục gói NuGet:** `dotnet restore` gặp sự cố và không tải xuống được tất cả các gói.
4.  **Môi trường build bị lỗi:** Cache NuGet bị hỏng hoặc môi trường .NET SDK có vấn đề.

**Cách sửa:**

1.  **Xóa `bin` và `obj` folders:**
    *   Mở Terminal trong thư mục gốc của dự án Backend của bạn.
    *   `Remove-Item -Path bin -Recurse -Force`
    *   `Remove-Item -Path obj -Recurse -Force`
2.  **Xóa cache NuGet toàn cục:**
    *   Trong Terminal, chạy `dotnet nuget locals all --clear` để xóa tất cả cache NuGet.
3.  **Chạy lại `dotnet restore`:**
    *   `dotnet restore`
4.  **Kiểm tra file `.csproj` của bạn:**
    *   Đảm bảo tất cả các thẻ `<PackageReference>` đều có `Version="..."` chính xác và không có các tham chiếu lạ hoặc trùng lặp.
    *   Kiểm tra `TargetFramework` (ví dụ: `net8.0`) có đúng với phiên bản .NET SDK bạn đã cài đặt không.
5.  **Clean và Build lại:** `dotnet clean` sau đó `dotnet build`.

### III. Các lỗi khác: CS0735, CS1704, CS1760, CS7008, CS9286

*   **CS0735: Invalid type specified as an argument for TypeForwardedToAttribute attribute.**
    *   Lỗi hiếm gặp, thường liên quan đến việc cấu hình nâng cao hoặc lỗi trong một thư viện khác khi cố gắng chuyển tiếp kiểu.
*   **CS1704: An assembly with the same simple name has already been imported. Try removing one of the references or sign them to enable side-by-side.**
*   **CS1760: Multiple assemblies refer to the same metadata but only one is a linked reference...**
    *   Hai lỗi này xảy ra khi có hai thư viện hoặc dự án khác nhau cung cấp cùng một kiểu hoặc namespace, gây ra sự mơ hồ.
    *   **Cách sửa:** Kiểm tra `.csproj` để tìm các tham chiếu NuGet hoặc Project References trùng lặp. Đôi khi xảy ra khi bạn thêm nhiều gói NuGet cung cấp các tính năng tương tự hoặc khi có nhiều dự án trong solution.
*   **CS7008: The assembly name is reserved and cannot be used as a reference in an interactive session.**
    *   Lỗi này không liên quan đến project thông thường, chỉ xảy ra trong các phiên tương tác (interactive sessions) của C#.
*   **CS9286: Type does not contain a definition and no accessible extension member for receiver type could be found...**
    *   Đây là một biến thể của `CS1061` (không tìm thấy thành viên), nhưng cho biết có thể là một thuộc tính/phương thức mở rộng. Khắc phục tương tự như `CS1061` (kiểm tra định nghĩa kiểu, `using` directive, package reference).

---

### Tóm lại quy trình sửa lỗi chung:

1.  **Đọc lỗi cẩn thận:** Xác định tên kiểu/namespace bị thiếu và file/dòng code bị lỗi.
2.  **Kiểm tra `using` directives:** Đảm bảo namespace đầy đủ được khai báo.
3.  **Kiểm tra tên Namespace thực tế:** Mở file định nghĩa của kiểu bị lỗi và xem `namespace ...` của nó.
4.  **Kiểm tra `.csproj` (Package References):** Đảm bảo các gói NuGet cần thiết đã được cài đặt.
5.  **Xóa `bin` và `obj`:** `dotnet clean` (hoặc xóa thủ công).
6.  **Xóa cache NuGet toàn cục:** `dotnet nuget locals all --clear`.
7.  **`dotnet restore`**
8.  **`dotnet build`**
9.  **`dotnet run`** (nếu build thành công).

Hãy bắt đầu với nhóm lỗi `CS0234` và `CS0246` trước, vì chúng là cơ bản nhất và việc sửa chúng thường sẽ giải quyết được nhiều lỗi khác cùng lúc.