---
name: console-client
description: Cài đặt Client dạng Console App gọi Order API bằng HttpClient: lấy menu, tạo đơn, đọc lại đơn, cố ý gửi yêu cầu sai để quan sát 400, kiểm tra mã trạng thái và đọc ApiError. Dùng sau khi Order API đã chạy được. Không nhận thông báo realtime.
---

# Client Console: gọi REST API

Trước khi làm: đọc `project-guardrails` và `coding-convention.md`. `order-api-rest` phải xong và chạy được.

## Phạm vi
Làm: luồng 4 bước gọi Order API, xử lý thành công và lỗi.
Không làm: kết nối SignalR, nhận thông báo realtime, vòng lặp tương tác, menu lựa chọn.

## Cấu hình
- URL của Order API lấy từ biến môi trường `Services__OrderApiUrl`. Thiếu biến thì in lỗi tiếng Việt rõ ràng và thoát với mã khác 0. Không đặt URL mặc định trong mã.
- Không thêm gói NuGet. Dùng `System.Net.Http.Json`.

## Luồng 4 bước (một lần chạy rồi thoát)
Dùng một `HttpClient` đặt `BaseAddress`, trong phạm vi `using`. Đặt `Console.OutputEncoding` là UTF-8.

1. `GET /menu`: in danh sách món (mã, tên, giá có định dạng).
2. `POST /orders`: tạo đơn cho một khách hàng mẫu với món đầu tiên của menu, số lượng 2.
   - Nếu mã trạng thái là `201 Created`: đọc `OrderDto`, in mã đơn và trạng thái ban đầu.
   - Nếu không: đọc `ApiError`, in mã lỗi và thông điệp, rồi thoát với mã khác 0.
3. `GET /orders/{id}`: đọc lại đơn vừa tạo và in khách hàng, trạng thái.
4. `POST /orders` cố ý sai (tên khách hàng rỗng, danh sách món rỗng): in mã trạng thái nhận được (mong đợi `400 BadRequest`) và `ApiError.Code`/`Message`.

Định nghĩa DTO của client là record riêng trong project Client, khớp hợp đồng của Order API (`MenuItemDto`, `CreateOrderRequest`, `CreateOrderItemRequest`, `OrderDto`, `OrderItemDto`, `ApiError`). Không tham chiếu project OrderApi.

## Xử lý lỗi
- Luôn kiểm tra mã trạng thái trước khi đọc thân phản hồi; không giả định yêu cầu nào cũng thành công.
- Bắt `HttpRequestException` đúng một chỗ ở mức chương trình: in thông báo "không kết nối được tới Order API" kèm URL đang dùng, thoát mã khác 0. Không bắt `Exception` chung.
- Một hàm nhỏ in `ApiError`; không thêm lớp tiện ích khác.

## Hoàn thành khi
Chạy cùng hệ thống: `docker compose up -d postgres order-api` rồi `docker compose run --rm client`. Dán output thật. Mong đợi:
- Danh sách 5 món.
- Mã đơn (Guid) và trạng thái `Pending`.
- Đọc lại đơn thành công.
- Yêu cầu sai trả `400 BadRequest` kèm `INVALID_ORDER`.

Kiểm thêm: dừng `order-api` rồi chạy lại client, phải thấy thông báo lỗi kết nối rõ ràng và mã thoát khác 0.
Commit: `Cài đặt Client console`. Báo cáo theo mục 7 của `project-guardrails`.
