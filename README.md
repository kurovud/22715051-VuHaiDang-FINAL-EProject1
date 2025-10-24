
# EProject — Microservices demo (Node.js, MongoDB, RabbitMQ)

Một project demo gồm 4 microservice: `auth`, `product`, `order`, và một `api-gateway`. Mục tiêu là minh họa kiến trúc microservices, giao tiếp HTTP + AMQP (RabbitMQ), và quy trình CI/CD với GitHub Actions.

## Tổng quan chức năng

- `auth`: Đăng ký/đăng nhập người dùng, cấp JWT, lưu người dùng vào MongoDB.
- `product`: Quản lý catalogue sản phẩm; publish event liên quan sản phẩm lên RabbitMQ.
- `order`: Nhận đơn hàng, consume event từ RabbitMQ, lưu đơn vào MongoDB.
- `api-gateway`: Cổng vào (reverse proxy) định tuyến tới các service.

## Kiến trúc & ports mặc định

| Service | Mô tả | Port mặc định |
| --- | --- | --- |
| `auth` | Authentication (MongoDB + JWT) | 3000 |
| `product` | Product catalogue (publishes events) | 3001 |
| `order` | Order processing (consumes events) | 3002 |
| `api-gateway` | Gateway/proxy cho các endpoint | 3003 |
| `mongo` | MongoDB (data store) | 27017 |
| `rabbitmq` | RabbitMQ (message broker) | 5672 (AMQP) / 15672 (UI) |

Các service giao tiếp qua REST và qua queues trên RabbitMQ. Cấu hình mặc định có thể ghi đè bằng biến môi trường (.env).

Dự án được tổ chức như sau:

```
.
├── api-gateway
├── auth
├── product
├── order
├── utils
├── docker-compose.yml
└── .github/workflows
```

## Biến môi trường

Mỗi service đọc cấu hình từ các environment variables. Một số biến thường dùng:

- `PORT`
- `MONGODB_*_URI` (ví dụ `MONGODB_AUTH_URI`, `MONGODB_PRODUCT_URI`, `MONGODB_ORDER_URI`)
- `JWT_SECRET`
- `RABBITMQ_URI` (ví dụ `amqp://user:pass@rabbitmq:5672`)
- `RABBITMQ_ORDER_QUEUE`, `RABBITMQ_PRODUCT_QUEUE`
- `RABBITMQ_CONNECT_DELAY_MS` (nếu cần delay khi service kết nối tới RabbitMQ)

Lưu ý: Docker Compose mặc định provision RabbitMQ với user/password `app:app`. Nếu dùng .env cá nhân, đảm bảo cập nhật URI tương ứng.

## Chạy toàn bộ hệ thống (Docker Compose)

Yêu cầu: Docker (Engine) chạy được trên máy.

1. Build và khởi động stack:

```powershell
docker compose up --build
```

2. Truy cập API gateway: http://localhost:3003 (gateway sẽ proxy tới `/auth`, `/products`, `/orders`).

3. Dừng stack:

```powershell
docker compose down
```

## Chạy từng service tại máy (local, không dùng Docker)

1. Cài dependencies root (tùy chọn):

```powershell
npm install
```

2. Cài dependencies và khởi chạy service riêng:

```powershell
# Cài dependencies cho service cụ thể
npm install --prefix auth
npm install --prefix product
npm install --prefix order
npm install --prefix api-gateway

# Khởi động service (ví dụ auth)

```

Lưu ý: Nếu chạy local, cần có MongoDB và RabbitMQ sẵn sàng (ở `localhost` hoặc update các biến môi trường tương ứng).

## Kiểm thử (Tests)

- Chạy toàn bộ test ở root (mocha pattern đã cấu hình):

```powershell
npm test
```

- Chạy test riêng cho service:

```powershell
npm test --prefix auth
npm test --prefix product
```

Chú ý về dependency: `npm ci` tại root chỉ cài packages ở `package.json` gốc. Các service có `package.json` riêng (ví dụ `auth`, `product`) cần `npm install --prefix <service>` nếu test import các dependency riêng.

## CI/CD (GitHub Actions)

Pipeline nằm ở `.github/workflows/ci.yml`. Hiện workflow thực hiện:

- Triggers: `push` (nhánh `main`/`master`) và `pull_request`.
- Job `build-and-test`: checkout, setup Node.js 18, `npm ci`, `npm test`, và build Docker images cho từng service (kiểm tra Dockerfile).
- Job `push-docker-images` (chỉ trên push vào main/master): login Docker Hub từ secrets `DOCKER_NAME`/`DOCKER_TOKEN`, build + push images tagged bằng SHA và `latest`.

Lưu ý quan trọng: hiện workflow chạy `npm ci` ở root. Nếu tests nằm trong từng service và phụ thuộc vào packages trong `auth/package.json` hoặc `product/package.json`, workflow cần cài dependencies cho từng service hoặc cấu hình workspace để đảm bảo tests có đủ package.

Nếu muốn, mình có thể giúp cập nhật workflow để:

- cài `npm ci` trong từng service trước khi chạy test, hoặc
- thiết lập npm workspaces để cài đồng bộ, hoặc
- chạy MongoDB / RabbitMQ dưới dạng services trong job nếu test cần cơ sở dữ liệu/queue thật.

## Vấn đề thường gặp & gợi ý khắc phục

- Nếu tests fail vì không tìm module: chạy `npm install --prefix <service>` trong workflow trước khi test.
- Nếu tests cần MongoDB/RabbitMQ: thêm `services:` vào job hoặc chạy `docker compose up -d mongo rabbitmq` trước khi test và sử dụng `wait-on` để chờ.
- Nếu Docker push thất bại: kiểm tra secrets `DOCKER_NAME` và `DOCKER_TOKEN` trong Settings -> Secrets.

## Đóng góp

- Mở issue hoặc pull request.
- Trước khi PR: chạy tests cục bộ và đảm bảo pipeline CI chạy (nếu có thể).

---

Nếu bạn muốn mình cập nhật file `.github/workflows/ci.yml` để cài deps per-service hoặc thêm DB services cho test, cho mình biết lựa chọn (ví dụ: "install per-service" hoặc "use docker-compose services") — mình sẽ cập nhật workflow sẵn sàng.

Sinh viên thực hiện : Vũ Hải Đăng
MSSV : 22715051
