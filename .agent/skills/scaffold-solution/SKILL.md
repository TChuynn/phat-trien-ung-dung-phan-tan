---
name: scaffold-solution
description: Dựng khung ban đầu cho hệ thống đặt món và xử lý tại bếp khi thư mục dự án còn trống: solution .NET 10, 4 project (order-api, kitchen-worker, notification-service, client), Dockerfile riêng, docker-compose.yml với postgres và rabbitmq, .env.example, .gitignore, git init. Chỉ tạo khung, không viết logic nghiệp vụ.
---

# Dựng khung hệ thống

Trước khi làm: đọc skill `project-guardrails` và `coding-convention.md`.

Mục tiêu: một khung build được và `docker compose up` chạy được. Chưa có logic nghiệp vụ.

## Cấu trúc thư mục

```
.
├── coding-convention.md
├── docker-compose.yml
├── .env.example
├── .gitignore
├── <file solution do `dotnet new sln` tạo ra>
└── src/
    ├── OrderApi/
    ├── KitchenWorker/
    ├── NotificationService/
    └── Client/
```

Mỗi project có `Dockerfile` riêng nằm trong thư mục project.

## Các bước

1. `git init`. Tạo `.gitignore` chuẩn cho .NET (thêm `.env`, `bin/`, `obj/`).
2. Tạo solution bằng `dotnet new sln`, giữ nguyên định dạng file mà lệnh sinh ra (với .NET 10 có thể là `.slnx`), không chuyển đổi.
3. Tạo 4 project, tất cả target `net10.0`:
   - `OrderApi`: `dotnet new web` (không dùng template `webapi` để tránh OpenAPI và mã mẫu thừa).
   - `NotificationService`: `dotnet new web`.
   - `KitchenWorker`: `dotnet new worker`.
   - `Client`: `dotnet new console`.
4. Xóa mã mẫu do template sinh ra mà không dùng (ví dụ `Worker.cs` với vòng lặp log mỗi giây, endpoint "Hello World"). Giữ chương trình tối thiểu, build được:
   - `OrderApi` và `NotificationService`: `Program.cs` chỉ tạo app, lắng nghe theo `ASPNETCORE_URLS`, chạy.
   - `KitchenWorker`: host chạy được và không làm gì. Không thêm `BackgroundService` giả.
   - `Client`: `Program.cs` trống hợp lệ.
   - Bật `Nullable` và `ImplicitUsings`, giữ mặc định của template.
5. Thêm các project vào solution.
6. Viết `Dockerfile` đa tầng cho từng project:
   - Tầng build: `mcr.microsoft.com/dotnet/sdk:10.0`; chạy `dotnet publish -c Release -o /app`.
   - Tầng chạy: `mcr.microsoft.com/dotnet/aspnet:10.0` cho `OrderApi` và `NotificationService`; `mcr.microsoft.com/dotnet/runtime:10.0` cho `KitchenWorker` và `Client`.
   - Build context là thư mục gốc repo, trỏ đúng đường dẫn project. Không `EXPOSE` và không cài thêm gì ngoài việc cần thiết.
7. Viết `docker-compose.yml` ở thư mục gốc, khai báo đủ 6 service trên một network:
   - `postgres`: image `postgres` (phiên bản chính cố định, không dùng `latest`), biến `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` lấy từ `.env`, volume đặt tên, healthcheck bằng `pg_isready`.
   - `rabbitmq`: image `rabbitmq` (phiên bản chính cố định, không dùng `latest`), user/password lấy từ `.env`, healthcheck bằng `rabbitmq-diagnostics -q ping`. Không bật management UI.
   - `order-api`, `kitchen-worker`, `notification-service`: build từ Dockerfile, `restart: unless-stopped`, `depends_on` với `condition: service_healthy` cho `postgres`/`rabbitmq` nơi cần, biến môi trường theo bảng ở mục C9 của `coding-convention.md`.
   - `client`: build từ Dockerfile, `restart: "no"` vì là ứng dụng console chạy một lần rồi thoát (restart tự động sẽ làm nó chạy lặp). Ghi rõ lý do này bằng một comment ngắn trong file compose.
   - Các service gọi nhau bằng tên service. Không dùng `localhost` hay IP.
   - Cổng publish ra máy host chỉ cho `order-api` và `notification-service`.
8. Viết `.env.example` chứa mọi biến mà compose cần, giá trị mẫu rõ ràng là giá trị mẫu (không dùng mật khẩu thật). Copy sang `.env` để chạy cục bộ.
9. Commit: `Dựng khung solution và docker-compose`.

## Hoàn thành khi
- `dotnet build` toàn solution không lỗi, không cảnh báo.
- `docker compose build` thành công cho cả 4 project.
- `docker compose up -d postgres rabbitmq` rồi `docker compose ps` cho thấy cả hai ở trạng thái healthy.
- `order-api`, `kitchen-worker`, `notification-service` khởi động và không thoát bất thường.
- Báo cáo theo mục 7 của `project-guardrails`.
