# Coding Convention – Đề tài 04: Hệ thống đặt món và xử lý tại bếp

Tài liệu này là quy định kỹ thuật bắt buộc dành cho AI coding agent (Antigravity) và lập trình viên.

- **Phần A** là yêu cầu kỹ thuật chung, quan trọng nhất, bắt buộc tuân thủ đầy đủ và chính xác.
- **Phần B** là yêu cầu của đề tài 04.
- **Phần C** là các quyết định kỹ thuật cụ thể để thực hiện Phần A và Phần B. Làm đúng theo, không tự đổi sang lựa chọn khác.
- Không tự ý thêm công nghệ, thành phần hay chức năng ngoài tài liệu này.

---

# PHẦN A. YÊU CẦU KỸ THUẬT CHUNG (BẮT BUỘC)

## A1. Ngôn ngữ và nền tảng
- Ngôn ngữ: C#.
- Nền tảng: .NET 10.

## A2. API và Client
- API: ASP.NET Core Minimal API.
- Client: Console App hoặc Web App.

## A3. Công nghệ giao tiếp và middleware
- Sử dụng các công nghệ giao tiếp và middleware theo yêu cầu cụ thể của từng đề tài, chẳng hạn REST, SignalR, gRPC, RabbitMQ hoặc các công nghệ tương đương được chấp nhận.

## A4. Cơ sở dữ liệu
- Sinh viên tự lựa chọn cơ sở dữ liệu phù hợp với bài toán.

## A5. Quản lý mã nguồn
- Sử dụng Git để quản lý mã nguồn.

## A6. Kiến trúc service
- Các service phải được tổ chức và chạy độc lập theo kiến trúc của đề tài.

## A7. Docker
- Mỗi service .NET phải có Dockerfile riêng.
- Có `docker-compose.yml` để khởi chạy toàn bộ hệ thống.
- Các container giao tiếp với nhau qua Docker network và service name.

## A8. Cấu hình
- Các cấu hình phụ thuộc môi trường như connection string, địa chỉ message broker, URL service và port phải được tách khỏi mã nguồn.

## A9. Logging
- Có logging đủ để theo dõi luồng xử lý qua các thành phần chính của hệ thống.

## A10. Khả năng phục hồi
- Hệ thống phải có khả năng tiếp tục hoạt động hoặc phục hồi hợp lý khi một thành phần tạm thời dừng hoạt động, phù hợp với cơ chế giao tiếp được sử dụng trong đề tài.

---

# PHẦN B. YÊU CẦU CỦA ĐỀ TÀI 04

Hệ thống cho phép khách hàng gửi yêu cầu đặt món. Đơn món được chuyển qua hàng đợi đến bộ phận bếp để xử lý. Khi trạng thái món thay đổi, khách hàng nhận được thông báo thời gian thực.

Kiến trúc tối thiểu:

```
Client -> Order API -> RabbitMQ -> Kitchen Worker -> RabbitMQ -> Notification Service -> Client
              |                         |
              +------> Database <-------+
```

Chức năng chính:
1. Xem danh sách món ăn.
2. Tạo đơn đặt món.
3. Đưa đơn món vào hàng đợi xử lý của bếp.
4. Kitchen Worker nhận và xử lý đơn theo thứ tự phù hợp.
5. Cập nhật trạng thái tối thiểu `Pending`, `Preparing`, `Ready`, `Cancelled`.
6. Phát sự kiện khi trạng thái đơn thay đổi.
7. Gửi thông báo realtime đến client.
8. Tra cứu chi tiết và trạng thái đơn món.

API tối thiểu:

| Method | Endpoint | Chức năng |
|---|---|---|
| GET | `/menu` | Xem danh sách món ăn |
| POST | `/orders` | Tạo đơn đặt món |
| GET | `/orders/{id}` | Tra cứu chi tiết và trạng thái đơn món |
| GET | `/orders` | Tra cứu danh sách đơn món |

Dữ liệu tối thiểu gồm khách hàng, món ăn, số lượng, thời điểm đặt và trạng thái xử lý.

---

# PHẦN C. QUYẾT ĐỊNH KỸ THUẬT CỤ THỂ

## C1. Thành phần hệ thống (thực hiện A6)

| Thành phần | Loại | Vai trò |
|---|---|---|
| `client` | .NET Console App | Gọi Order API, nhận thông báo realtime từ Notification Service |
| `order-api` | .NET, ASP.NET Core Minimal API | Nhận yêu cầu từ client, ghi Database, đẩy đơn vào RabbitMQ |
| `kitchen-worker` | .NET Worker Service | Nhận đơn từ RabbitMQ, xử lý, cập nhật trạng thái trong Database, phát sự kiện đổi trạng thái |
| `notification-service` | .NET, ASP.NET Core Minimal API + SignalR | Nhận sự kiện đổi trạng thái từ RabbitMQ, gửi thông báo realtime đến client |
| `rabbitmq` | RabbitMQ | Message broker |
| `postgres` | PostgreSQL | Cơ sở dữ liệu |

## C2. API (thực hiện A2)
- Định dạng dữ liệu trao đổi qua API: JSON.
- `POST /orders` nhận: tên khách hàng và danh sách món gồm mã món, số lượng. Trả về mã đơn và trạng thái `Pending`.

## C3. Client (thực hiện A2)
- Dùng **Console App** (.NET).

## C4. Giao tiếp và middleware (thực hiện A3)
- Công nghệ realtime: **SignalR**. Notification Service chứa SignalR Hub, Client kết nối vào Hub để nhận thông báo.
- Thư viện RabbitMQ: `RabbitMQ.Client` (gói NuGet chính thức).
- Hai hàng đợi tách biệt:
  - `kitchen.orders`: Order API gửi, Kitchen Worker nhận (đơn mới cần xử lý).
  - `order.status.changed`: Kitchen Worker gửi, Notification Service nhận (sự kiện đổi trạng thái).
- Thứ tự xử lý của Kitchen Worker: đơn đến trước xử lý trước (FIFO).
- Nội dung sự kiện đổi trạng thái gồm tối thiểu: mã đơn, trạng thái mới, thời điểm thay đổi.

## C5. Nghiệp vụ
- Luồng trạng thái: đơn mới tạo ở `Pending` → Kitchen Worker chuyển sang `Preparing` → chuyển sang `Ready`. Trạng thái `Cancelled` dùng cho đơn bị hủy.

## C6. Cơ sở dữ liệu (thực hiện A4)
- Dùng **PostgreSQL**, chạy trong container, truy cập bằng **Entity Framework Core** với provider `Npgsql.EntityFrameworkCore.PostgreSQL`.
- Các bảng:
  - `MenuItems`: mã món, tên món, giá.
  - `Orders`: mã đơn, tên khách hàng, thời điểm đặt, trạng thái.
  - `OrderItems`: mã đơn, mã món, số lượng.
- Order API và Kitchen Worker cùng truy cập database này. Notification Service không truy cập database.
- Dữ liệu món ăn mẫu được seed khi khởi động để `GET /menu` có dữ liệu.

## C7. Quản lý mã nguồn (thực hiện A5)
- Không commit các giá trị cấu hình thật; chỉ commit file mẫu `.env.example`.

## C8. Docker (thực hiện A7)
- Dockerfile riêng cho từng service .NET: `order-api`, `kitchen-worker`, `notification-service`, `client`.
- `docker-compose.yml` ở thư mục gốc khai báo đủ 6 thành phần ở C1 trên cùng một Docker network.
- Các container gọi nhau bằng service name (`postgres`, `rabbitmq`, `order-api`, `notification-service`), không dùng `localhost` hay IP cố định.

## C9. Cấu hình (thực hiện A8)
- Đọc cấu hình từ biến môi trường, giá trị được khai báo trong `docker-compose.yml` và `.env`. Không hard-code trong mã nguồn.
- Tên biến môi trường:

| Biến | Ý nghĩa | Dùng bởi |
|---|---|---|
| `ConnectionStrings__Default` | Connection string PostgreSQL | order-api, kitchen-worker |
| `RabbitMq__Host` | Địa chỉ RabbitMQ (service name) | order-api, kitchen-worker, notification-service |
| `RabbitMq__Port` | Port RabbitMQ | order-api, kitchen-worker, notification-service |
| `RabbitMq__User` / `RabbitMq__Password` | Tài khoản RabbitMQ | order-api, kitchen-worker, notification-service |
| `Services__OrderApiUrl` | URL Order API | client |
| `Services__NotificationHubUrl` | URL SignalR Hub | client |
| `ASPNETCORE_URLS` | URL/port lắng nghe | order-api, notification-service |

## C10. Logging (thực hiện A9)
- Dùng `ILogger` có sẵn của .NET, ghi ra console.
- Ghi log tại các điểm sau, kèm mã đơn để lần theo luồng:
  - Order API: nhận yêu cầu tạo đơn, lưu đơn, gửi đơn vào `kitchen.orders`.
  - Kitchen Worker: nhận đơn, đổi trạng thái, gửi sự kiện vào `order.status.changed`.
  - Notification Service: nhận sự kiện, gửi thông báo đến client.
  - Client: gửi yêu cầu, nhận thông báo.

## C11. Khả năng phục hồi (thực hiện A10)
- Hàng đợi và tin nhắn ở chế độ bền (durable queue, persistent message) để không mất đơn khi RabbitMQ hoặc worker khởi động lại.
- Kitchen Worker và Notification Service chỉ xác nhận (ack) tin nhắn sau khi xử lý xong.
- Các service tự thử kết nối lại RabbitMQ và PostgreSQL khi khởi động hoặc khi mất kết nối, thay vì thoát ngay.
- Container cấu hình tự khởi động lại (`restart: unless-stopped`) và `depends_on` kèm healthcheck cho `postgres` và `rabbitmq`.

---

# CHECKLIST TUÂN THỦ

**Yêu cầu chung (Phần A)**
- [ ] A1: C#, .NET 10
- [ ] A2: API dùng ASP.NET Core Minimal API; Client là Console App
- [ ] A3: Giao tiếp và middleware theo đề tài (RabbitMQ, SignalR)
- [ ] A4: Cơ sở dữ liệu PostgreSQL, phù hợp với bài toán
- [ ] A5: Mã nguồn quản lý bằng Git
- [ ] A6: Các service tổ chức và chạy độc lập theo kiến trúc của đề tài
- [ ] A7: Mỗi service .NET có Dockerfile riêng; có `docker-compose.yml` chạy toàn bộ hệ thống; container giao tiếp qua Docker network và service name
- [ ] A8: Connection string, địa chỉ RabbitMQ, URL service, port tách khỏi mã nguồn
- [ ] A9: Logging đủ để theo dõi luồng xử lý qua các thành phần chính
- [ ] A10: Hệ thống tiếp tục hoạt động hoặc phục hồi hợp lý khi một thành phần tạm thời dừng

**Đề tài 04 (Phần B)**
- [ ] Luồng: Client → Order API → RabbitMQ → Kitchen Worker → RabbitMQ → Notification Service → Client
- [ ] Đủ 4 endpoint: `GET /menu`, `POST /orders`, `GET /orders/{id}`, `GET /orders`
- [ ] Trạng thái tối thiểu: `Pending`, `Preparing`, `Ready`, `Cancelled`
- [ ] Phát sự kiện khi đổi trạng thái; thông báo realtime đến Client
- [ ] Lưu tối thiểu: khách hàng, món ăn, số lượng, thời điểm đặt, trạng thái xử lý
