---
name: order-api-rest
description: Cài đặt Order API (ASP.NET Core Minimal API) với 4 endpoint GET /menu, POST /orders, GET /orders/{id}, GET /orders, DTO, ApiError, mã trạng thái 201/400/404, lưu PostgreSQL bằng EF Core, seed món ăn, logging kèm mã đơn. Dùng sau khi đã có khung solution. Không đẩy đơn vào RabbitMQ.
---

# Order API: phần REST

Trước khi làm: đọc `project-guardrails` và `coding-convention.md`. Khung solution (skill `scaffold-solution`) phải có sẵn.

## Phạm vi
Làm: 4 endpoint, DTO, kiểm tra dữ liệu, mã trạng thái, lưu và đọc từ PostgreSQL, seed, logging.
Không làm: gửi đơn vào RabbitMQ, đổi trạng thái khác `Pending`, xác thực, phân trang, Swagger.

## Cấu trúc file (gọn, không thêm tầng)

```
src/OrderApi/
├── Program.cs
├── Contracts/          # DTO và ApiError (record)
├── Data/               # entity, AppDbContext, seed
└── Endpoints/          # nhóm endpoint (MapMenu, MapOrders)
```

Chỉ thêm gói `Npgsql.EntityFrameworkCore.PostgreSQL`. Không thêm gói khác.

## Hợp đồng API

| Endpoint | Thành công | Lỗi |
|---|---|---|
| `GET /menu` | `200` danh sách `MenuItemDto(Id, Name, Price)` | |
| `POST /orders` | `201 Created`, header `Location: /orders/{id}`, thân là `OrderDto` | `400` với `ApiError` |
| `GET /orders/{id}` | `200` với `OrderDto` | `404` với `ApiError` |
| `GET /orders` | `200` danh sách `OrderDto`, mới nhất trước | |

DTO (record, tách khỏi entity):
- `CreateOrderRequest(string CustomerName, List<CreateOrderItemRequest> Items)`
- `CreateOrderItemRequest(int MenuItemId, int Quantity)`
- `OrderDto(Guid Id, string CustomerName, string Status, DateTimeOffset OrderedAt, List<OrderItemDto> Items)`
- `OrderItemDto(int MenuItemId, string MenuItemName, int Quantity)`
- `ApiError(string Code, string Message)`

Mã lỗi (UPPER_SNAKE) và thông điệp tiếng Việt:
- `INVALID_ORDER`: thiếu tên khách hàng, danh sách món rỗng, hoặc số lượng không dương.
- `INVALID_MENU_ITEM`: mã món không tồn tại.
- `ORDER_NOT_FOUND`: không có đơn với mã đã cho.

Tuyến `{id}` ràng buộc kiểu `guid` (`/orders/{id:guid}`). Mã đơn là `Guid` do server sinh.

## Dữ liệu (khớp mục C6 của convention)
- Entity: `MenuItem(Id, Name, Price)`, `Order(Id, CustomerName, OrderedAt, Status)`, `OrderItem(OrderId, MenuItemId, Quantity)`.
- `OrderStatus` là enum có đủ 4 giá trị `Pending`, `Preparing`, `Ready`, `Cancelled`, lưu dạng chuỗi trong DB. Ở giai đoạn này chỉ dùng `Pending`.
- `Price` kiểu `decimal` với độ chính xác khai báo rõ.
- `OrderedAt` gán bằng `DateTimeOffset.UtcNow` ở server.
- Seed 5 món ăn mẫu khi bảng `MenuItems` còn trống.
- Khởi tạo schema bằng `EnsureCreated` khi ứng dụng khởi động, bọc trong vòng lặp thử lại có giới hạn (ví dụ 10 lần, cách nhau vài giây) để chịu được việc PostgreSQL chưa sẵn sàng. Mỗi lần thất bại ghi log cảnh báo; hết số lần thì để ứng dụng dừng với lỗi rõ ràng.
- Connection string lấy từ cấu hình `ConnectionStrings:Default` (biến môi trường `ConnectionStrings__Default`). Không viết cứng.

## Quy tắc xử lý
- Kiểm tra dữ liệu bằng mã thủ công ở đầu endpoint rồi trả `Results.BadRequest(new ApiError(...))`. Không thêm thư viện validation.
- Chỉ trả `404` khi đơn không tồn tại; không dùng `404` cho lỗi dữ liệu.
- Tạo đơn: kiểm tra tất cả món trong một truy vấn, lưu `Order` và `OrderItem` trong một lần `SaveChangesAsync`.
- Không trả entity EF ra ngoài, luôn ánh xạ sang DTO bằng mã viết tay.

## Logging (ILogger, kèm mã đơn)
- Nhận yêu cầu tạo đơn (khách hàng, số món).
- Đã lưu đơn `{OrderId}` ở trạng thái `Pending`.
- Yêu cầu bị từ chối kèm mã lỗi.
- Không log dữ liệu thừa, không log nguyên thân yêu cầu.

## Hoàn thành khi
Chạy `docker compose up -d postgres order-api`, rồi dùng `curl` kiểm tra và dán kết quả thật:
1. `GET /menu` trả `200` với 5 món.
2. `POST /orders` hợp lệ trả `201`, có header `Location`, trạng thái `Pending`.
3. `GET /orders/{id}` với mã vừa tạo trả `200`; với `Guid` ngẫu nhiên trả `404` + `ORDER_NOT_FOUND`.
4. `POST /orders` với tên rỗng trả `400` + `INVALID_ORDER`; với `MenuItemId` không tồn tại trả `400` + `INVALID_MENU_ITEM`.
5. `GET /orders` trả đơn vừa tạo.
6. Khởi động lại `order-api`: dữ liệu vẫn còn.
7. Commit: `Cài đặt Order API (REST)`. Báo cáo theo mục 7 của `project-guardrails`.
