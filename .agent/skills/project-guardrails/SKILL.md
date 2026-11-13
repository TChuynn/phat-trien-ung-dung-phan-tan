---
name: project-guardrails
description: Quy tắc bắt buộc khi viết, sửa hoặc kiểm tra bất kỳ mã nào của dự án đặt món và xử lý tại bếp. Dùng trước mọi tác vụ mã nguồn, Dockerfile, docker-compose hoặc cấu hình trong dự án này. Chống thêm thắt ngoài phạm vi, chống mã thừa và ngăn báo cáo "xong" khi chưa chạy thử.
---

# Quy tắc chống "AI slop" cho dự án

Nguồn sự thật duy nhất là `coding-convention.md` ở thư mục gốc. Đọc file đó trước khi làm. Nếu yêu cầu mâu thuẫn hoặc thiếu thông tin, dừng lại và hỏi người dùng; không tự bịa quyết định mới.

## 1. Giữ đúng phạm vi
- Chỉ làm đúng việc mà skill hoặc yêu cầu hiện tại nêu. Không làm trước phần của giai đoạn sau.
- Ngoài phạm vi ở giai đoạn này: publish hoặc consume RabbitMQ, SignalR, logic xử lý của Kitchen Worker, chuyển trạng thái `Preparing`/`Ready`/`Cancelled`, xác thực, phân trang, lọc, Swagger/OpenAPI, health endpoint.
- Không tạo stub, interface rỗng hay hàm "để dành sau". Phần chưa làm thì không có mặt trong mã.
- Không đổi công nghệ đã chốt trong `coding-convention.md` (PostgreSQL + EF Core, SignalR, RabbitMQ.Client, Console App).

## 2. Mã phải tối giản
Cấm, trừ khi người dùng yêu cầu rõ:
- MediatR, AutoMapper, FluentValidation, Polly, Serilog, Swagger và mọi gói NuGet không nằm trong skill đang dùng.
- Repository hay Unit of Work bọc lên EF Core; lớp `Service` cho logic chỉ có vài dòng; interface chỉ có một lớp cài đặt; lớp cơ sở chung; CQRS; thư mục phân tầng (Domain/Application/Infrastructure).
- `catch (Exception)` nuốt lỗi, `try/catch` bao quanh mọi thứ.
- Giá trị cấu hình viết cứng (connection string, host, port, URL).
- Chuỗi ma thuật lặp lại: dùng hằng số hoặc enum.
- Mỗi gói NuGet thêm vào phải nêu được lý do dùng trong một câu.

## 3. Comment và tài liệu
- Chỉ comment khi giải thích vì sao, không mô tả lại điều dòng mã đã nói.
- Không comment XML cho mọi thành viên, không comment kiểu `// Create the order`, không để mã bị comment lại, không để `TODO` hay `FIXME` thừa.
- Không emoji trong mã, log hay thông báo.
- Không tự viết README dài. Chỉ cập nhật tài liệu khi được yêu cầu.

## 4. Quy ước đặt tên và ngôn ngữ
- Định danh trong mã (lớp, biến, endpoint) dùng tiếng Anh, đúng như `coding-convention.md`.
- Thông điệp `ApiError.Message` và dòng in ra console dùng tiếng Việt có dấu.
- Log dùng `ILogger` với message template có tham số (`"Đã lưu đơn {OrderId}"`), không nối chuỗi.

## 5. Git
- Chạy `git init` ở bước dựng khung. Mỗi skill hoàn thành là một commit riêng, thông điệp ngắn, nói rõ đã làm gì.
- Không bao giờ commit `.env`, `bin/`, `obj/`. Chỉ commit `.env.example`.

## 6. Kiểm chứng trước khi báo xong
Không được nói "đã xong" khi chưa chạy thật. Với mỗi skill:
1. `dotnet build` không lỗi và không cảnh báo.
2. Chạy thực tế phần vừa làm, và dán kết quả thật (mã trạng thái, JSON, output console).
3. Nếu không chạy được (thiếu Docker, mạng, cổng bị chiếm), nói thẳng là chưa kiểm chứng và nêu phần nào chưa kiểm.

## 7. Báo cáo cuối mỗi tác vụ
Viết ngắn gọn, đúng sự thật, gồm 3 mục:
- Đã làm: danh sách file và hành vi.
- Đã chạy thử: lệnh và kết quả.
- Chưa làm hoặc chưa kiểm chứng: nêu rõ, không giấu.
Không khen ngợi, không tổng kết dài dòng, không hứa các tính năng chưa làm.
