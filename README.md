# JobBeacon discovery

Service registry (Eureka server) của JobBeacon. Không có nghiệp vụ: chỉ khởi chạy Spring Boot với `@EnableEurekaServer`.

Các service đăng ký vào đây khi khởi động và gửi heartbeat định kỳ:

| Service | Tên trong Eureka (`spring.application.name`) |
|---|---|
| Jobbeacon-be | `jobbeacon-be` |
| Jobbeacon-recommendation | `jobbeacon-recommendation` |
| Jobbeacon-sync | `jobbeacon-sync` |
| job-posting-crawler | `job-posting-crawler` |

Lời gọi HTTP giữa các service đi qua Feign theo tên service, không dùng URL cố định. Hiện chỉ có một luồng như vậy: recommendation gọi API `/internal/**` của backend qua `@FeignClient(name = "${app.backend.service-id}")` (mặc định `jobbeacon-be`). Crawler, sync và backend không gọi HTTP sang nhau (chúng giao tiếp qua RabbitMQ và CDC), nên chỉ đăng ký để hiện trên registry.

## Chạy local

```bash
mvn spring-boot:run
```

Dashboard: `http://localhost:8761`. Danh sách instance dạng JSON/XML: `http://localhost:8761/eureka/apps`. Health: `http://localhost:8761/actuator/health`.

Các service khác mặc định trỏ tới `http://localhost:8761/eureka/`. Muốn chạy một service mà không cần discovery, đặt `EUREKA_ENABLED=false` (riêng recommendation khi đó không gọi được backend vì không còn URL cố định).

## Cấu hình

| Biến | Mặc định | Ý nghĩa |
|---|---|---|
| `PORT` | `8761` | Cổng HTTP |
| `EUREKA_HOSTNAME` | `localhost` | Hostname của server (trong Docker là `discovery`) |
| `EUREKA_SELF_PRESERVATION` | `false` | Bật self-preservation (giữ instance khi mất nhiều heartbeat cùng lúc) |
| `EUREKA_EVICTION_INTERVAL_MS` | `30000` | Chu kỳ dọn instance đã hết hạn |

Phía client (các service còn lại):

| Biến | Mặc định | Ý nghĩa |
|---|---|---|
| `EUREKA_URL` | `http://localhost:8761/eureka/` | Địa chỉ registry |
| `EUREKA_ENABLED` | `true` | Tắt để chạy độc lập |
| `BACKEND_SERVICE_ID` (recommendation) | `jobbeacon-be` | Tên backend trong Eureka mà Feign gọi tới |

Client đăng ký bằng IP (`prefer-ip-address: true`) để các container gọi được nhau trong mạng Docker.

## Docker

```bash
docker build -t jobbeacon/discovery .
docker run -p 8761:8761 jobbeacon/discovery
```

Trong `jobbeacon-deployment/docker-compose.yml`, service `discovery` khởi động trước; backend, crawler, sync và recommendation chờ nó `healthy`.

## Test

```bash
mvn test
```
