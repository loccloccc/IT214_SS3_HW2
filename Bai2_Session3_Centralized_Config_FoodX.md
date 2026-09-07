# Bài 2 — Tổ chức Centralized Configuration cho hệ thống FoodX

> **Session 02 — Từ Monolithic đến Microservice** | Cấp độ: Vận dụng cơ bản

---

## 1. Phân tích lỗi đặt tên file cấu hình

### 1.1 Quy ước đặt tên file trong Spring Cloud Config Server

Spring Cloud Config Server định vị file cấu hình dựa trên giá trị `{application}` — là giá trị của thuộc tính `spring.application.name` được khai báo trong từng service. Quy ước tên file như sau:

```
{application}.yml                  -- cau hinh cho tat ca moi truong (default)
{application}-{profile}.yml        -- cau hinh rieng cho tung moi truong

Vi du:
  restaurant-service.yml           -- default cho restaurant-service
  restaurant-service-dev.yml       -- chi cho moi truong dev
  restaurant-service-prod.yml      -- chi cho moi truong production
```

Ngoài ra, Config Server còn hỗ trợ file dùng chung cho tất cả service:

```
application.yml                    -- cau hinh mac dinh cho moi service
application-{profile}.yml          -- cau hinh mac dinh cho tung profile
```

### 1.2 Lỗi trong file hiện tại

File cấu hình hiện tại được đặt tên là `config.yml`. Trong khi đó, `restaurant-service` có:

```yaml
# Khai bao trong restaurant-service (bootstrap.yml hoac application.yml)
spring:
  application:
    name: restaurant-service
```

Khi `restaurant-service` khởi động và kết nối đến Config Server, nó sẽ gửi yêu cầu:

```
GET http://config-server/restaurant-service/default
```

Config Server sẽ tìm kiếm theo thứ tự:
1. `restaurant-service-default.yml`
2. `restaurant-service.yml`
3. `application.yml`

Tên `config.yml` không trùng với bất kỳ mẫu nào ở trên. Config Server sẽ **không tìm thấy** file này và `restaurant-service` sẽ khởi động thiếu cấu hình — không có `datasource.url`, không có `server.port` — dẫn đến lỗi kết nối database ngay khi khởi động.

### 1.3 Hậu quả cụ thể khi đặt sai tên

| Hậu quả | Mô tả chi tiết |
|---|---|
| Service khởi động thất bại | Thiếu `datasource.url` → Spring Boot không thể tạo DataSource bean → ứng dụng crash ngay khi start |
| Cấu hình fallback về default | Nếu có `application.yml` chung, service dùng cấu hình đó — có thể kết nối nhầm database của service khác |
| Khó debug | Không có exception rõ ràng nói "sai tên file" — chỉ thấy `DataSourceBeanCreationException` hoặc `Connection refused` |
| Khó phát hiện trong CI/CD | File vẫn tồn tại trong Git, CI/CD vẫn pass, nhưng service chỉ vỡ khi chạy thật |

### 1.4 Cach sua: doi ten file

```
Truoc: config.yml          (SAI)
Sau:   restaurant-service.yml   (DUNG)
```

---

## 2. Phân tích rủi ro mật khẩu plaintext và cách sửa

### 2.1 Rủi ro khi lưu mật khẩu dạng plaintext trong Git

```yaml
# Nguy hiem: mat khau lo ra bat ky ai co quyen doc Git repo
password: RestaurantPass123
```

| Rủi ro | Mức độ | Mô tả cụ thể với FoodX |
|---|---|---|
| Git history luu vinh vien | Nghiem trong | Du sau nay xoa dong nay, `git log` / `git reflog` van hien thi mat khau cu trong lich su commit |
| Ai co quyen read repo la biet mat khau | Nghiem trong | Dev moi, intern, bot CI/CD deu doc duoc mat khau production database |
| Ro qua log | Trung binh | Mot so tool log YAML config khi khoi dong — mat khau xuat hien trong log file |
| Vi pham tuan thu bao mat | Cao | He thong xu ly du lieu don hang, thanh toan — vi pham PCI-DSS neu lo mat khau DB |

### 2.2 Giai phap: Ma hoa bang Spring Cloud Config Cipher

Spring Cloud Config Server ho tro ma hoa gia tri bang khoa doi xung (AES) hoac khoa bat doi xung (RSA). Gia tri ma hoa duoc bao boc trong cu phap `{cipher}...`:

```yaml
# Dung: gia tri da ma hoa, chi Config Server moi giai ma duoc
spring:
  datasource:
    password: '{cipher}AQBXk3z9vP2mR7tL8nQwYcDfGhJsKpUoI1eN6bVxWaM...'
```

**Cach thuc hoat dong:**

```
+----------------+        +-------------------+        +---------------------+
| Git Repository |        |   Config Server   |        |  restaurant-service |
|                |        |                   |        |                     |
| password:      | -----> | Doc file          | -----> | Nhan gia tri da     |
| '{cipher}ABC'  |        | Phat hien {cipher}|        | giai ma:            |
|                |        | Giai ma bang key  |        | RestaurantPass123   |
+----------------+        +-------------------+        +---------------------+

Khoa giai ma chi nam tren Config Server, KHONG commit vao Git
```

**Minh hoa cu phap day du (khong can ma hoa that):**

```yaml
spring:
  datasource:
    url: jdbc:mysql://foodx-cluster.local:3306/restaurants_db
    username: restaurant_service_user
    password: '{cipher}AQBXk3z9vP2mR7tL8nQwYcDfGhJsKpUoI1eN6bVxWaM4rEt2sFuHdZ0qCjOy'
```

**Cach tao gia tri cipher trong thuc te:**

```bash
# Goi API cua Config Server de ma hoa gia tri
curl -X POST http://config-server:8888/encrypt \
  -H "Content-Type: text/plain" \
  -d "RestaurantPass123"

# Config Server tra ve chuoi da ma hoa:
# AQBXk3z9vP2mR7tL8nQwYcDfGhJsKpUoI1eN6bVxWaM4rEt2sFuHdZ0qCjOy

# Dan chuoi do vao file YAML voi tien to {cipher}
```

**Cau hinh khoa ma hoa tren Config Server (application.yml cua config-server):**

```yaml
# Luu trong Config Server, KHONG commit vao config Git repo
encrypt:
  key: ${ENCRYPT_KEY}   # lay tu bien moi truong, khong hardcode

# Hoac dung RSA (khuyen nghi cho production):
encrypt:
  key-store:
    location: classpath:foodx-config.jks
    password: ${KEYSTORE_PASSWORD}
    alias: foodx-config-key
```

---

## 3. Ba file cấu hình hoàn chỉnh

### 3.1 restaurant-service.yml

```yaml
# ============================================================
# FILE: restaurant-service.yml
# Phuc vu: restaurant-service (spring.application.name = restaurant-service)
# Mo ta: Cau hinh ket noi DB, port, va cac thong so noi bo
# ============================================================

spring:
  datasource:
    url: jdbc:mysql://foodx-cluster.local:3306/restaurants_db
    username: restaurant_service_user
    password: '{cipher}AQBXk3z9vP2mR7tL8nQwYcDfGhJsKpUoI1eN6bVxWaM4rEt2sFuHdZ0qCjOy'
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect

server:
  port: 8085

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when-authorized

logging:
  level:
    vn.foodx.restaurant: INFO
    org.springframework.web: WARN
  file:
    name: /var/log/foodx/restaurant-service.log

foodx:
  restaurant:
    max-menu-items-per-page: 50
    image-upload-max-size-mb: 5
    default-open-time: "07:00"
    default-close-time: "22:00"
```

### 3.2 order-service.yml

```yaml
# ============================================================
# FILE: order-service.yml
# Phuc vu: order-service (spring.application.name = order-service)
# Mo ta: Cau hinh ket noi DB, message queue, va cac thong so don hang
# ============================================================

spring:
  datasource:
    url: jdbc:mysql://foodx-cluster.local:3306/orders_db
    username: order_service_user
    password: '{cipher}BRCYl4a0wQ3nS8uM9oRxZdEgHiKtLqVpJ2fO7cWyXbN5sAt3uGvIeQ1rDkPz'
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect

  rabbitmq:
    host: foodx-mq.local
    port: 5672
    username: order_mq_user
    password: '{cipher}CSDZm5b1xR4oT9vN0pSyAeHjIlMuOrWqK3gP8dXzYcO6tBu4vHwJfR2sDlQa'
    virtual-host: /foodx

server:
  port: 8086

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when-authorized

logging:
  level:
    vn.foodx.order: INFO
    org.springframework.amqp: WARN
  file:
    name: /var/log/foodx/order-service.log

foodx:
  order:
    max-items-per-order: 20
    payment-timeout-minutes: 15
    auto-cancel-unpaid-minutes: 30
    commission-rate-percent: 15
```

### 3.3 delivery-service.yml

```yaml
# ============================================================
# FILE: delivery-service.yml
# Phuc vu: delivery-service (spring.application.name = delivery-service)
# Mo ta: Cau hinh ket noi DB, tracking, va thong so giao hang
# ============================================================

spring:
  datasource:
    url: jdbc:mysql://foodx-cluster.local:3306/deliveries_db
    username: delivery_service_user
    password: '{cipher}DTEAn6c2yS5pU0wO1qTzBfIkJmNvPsXrL4hQ9eYaZdP7uCv5wIxKgS3tEmRb'
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 15
      minimum-idle: 3
      connection-timeout: 30000

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect

  data:
    redis:
      host: foodx-redis.local
      port: 6379
      password: '{cipher}EUFBo7d3zT6qV1xP2rUaBgJlKnOwQtYsM5iR0fZbAeQ8vDw6xJyLhT4uFnSc'
      timeout: 3000ms
      lettuce:
        pool:
          max-active: 10
          min-idle: 2

server:
  port: 8087

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when-authorized

logging:
  level:
    vn.foodx.delivery: INFO
  file:
    name: /var/log/foodx/delivery-service.log

foodx:
  delivery:
    max-distance-km: 10
    base-fee: 15000
    fee-per-km: 5000
    estimated-time-per-km-minutes: 3
    driver-assignment-timeout-seconds: 60
    tracking-update-interval-seconds: 15
```

### 3.4 application.yml — cấu hình dùng chung cho tất cả service

```yaml
# ============================================================
# FILE: application.yml
# Phuc vu: TAT CA service (duoc load truoc file rieng tung service)
# Mo ta: Cau hinh mac dinh va cac thong so dung chung
# ============================================================

spring:
  cloud:
    config:
      fail-fast: true         # Service tu choi khoi dong neu khong ket noi duoc Config Server
      retry:
        initial-interval: 1000
        max-attempts: 6

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30

management:
  endpoints:
    web:
      base-path: /actuator
  health:
    diskspace:
      enabled: true

logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
  level:
    root: WARN
    org.springframework: WARN
```

---

## 4. README — Cấu trúc Git Repository cấu hình tập trung

```markdown
# foodx-config-repo

Git repository luu tru tap trung cau hinh cho toan bo he thong FoodX.
Repository nay duoc Spring Cloud Config Server doc va phuc vu cho cac service.

---

## Cau truc thu muc

foodx-config-repo/
├── application.yml                    # Cau hinh mac dinh dung chung cho MOI service
├── application-dev.yml                # Cau hinh chung cho moi truong Development
├── application-staging.yml            # Cau hinh chung cho moi truong Staging
├── application-prod.yml               # Cau hinh chung cho moi truong Production
│
├── restaurant-service.yml             # Cau hinh rieng cua restaurant-service (default)
├── restaurant-service-dev.yml         # Cau hinh restaurant-service trong moi truong dev
├── restaurant-service-staging.yml     # Cau hinh restaurant-service trong staging
├── restaurant-service-prod.yml        # Cau hinh restaurant-service trong production
│
├── order-service.yml                  # Cau hinh rieng cua order-service (default)
├── order-service-dev.yml
├── order-service-staging.yml
├── order-service-prod.yml
│
├── delivery-service.yml               # Cau hinh rieng cua delivery-service (default)
├── delivery-service-dev.yml
├── delivery-service-staging.yml
├── delivery-service-prod.yml
│
└── README.md                          # File nay

---

## Quy uoc dat ten file

| Mau ten file                   | Y nghia                                                 |
|--------------------------------|---------------------------------------------------------|
| application.yml                | Cau hinh chung, moi service deu nhan duoc               |
| application-{profile}.yml      | Cau hinh chung rieng cho tung profile (dev/staging/prod)|
| {service-name}.yml             | Cau hinh rieng cua service, ap dung cho moi profile     |
| {service-name}-{profile}.yml   | Cau hinh rieng cua service, chi trong profile cu the   |

Gia tri {service-name} PHAI khop chinh xac voi spring.application.name cua service do.

Thu tu uu tien (cao nhat trc): {service-name}-{profile}.yml > {service-name}.yml > application-{profile}.yml > application.yml

---

## Quy tac bao mat bat buoc

1. KHONG bao gio luu mat khau, secret key, API token o dang plaintext.
2. Moi gia tri nhay cam PHAI duoc ma hoa bang cu phap: '{cipher}<gia-tri-ma-hoa>'
3. Tao gia tri cipher bang lenh:
       curl -X POST http://config-server:8888/encrypt -d "gia-tri-can-ma-hoa"
4. Khoa ma hoa nam tren Config Server, lay tu bien moi truong (ENCRYPT_KEY),
   KHONG commit vao bat ky repository nao.
5. File .gitignore phai bao gom cac file key va keystore:
       *.jks
       *.p12
       .env

---

## Cach Config Server lay cau hinh

Config Server duoc cau hinh tro vao repository nay:

    # config-server/src/main/resources/application.yml
    spring:
      cloud:
        config:
          server:
            git:
              uri: https://github.com/foodx/foodx-config-repo
              default-label: main
              search-paths: '.'
              clone-on-start: true

---

## Cach service lay cau hinh

Moi service khai bao trong bootstrap.yml (hoac application.yml voi Spring Boot 2.4+):

    # Vi du: restaurant-service/src/main/resources/bootstrap.yml
    spring:
      application:
        name: restaurant-service        # PHAI khop voi ten file trong repo nay
      cloud:
        config:
          uri: http://config-server:8888
          profile: ${SPRING_PROFILES_ACTIVE:dev}
          fail-fast: true

---

## Lich su thay doi (Changelog)

| Ngay       | Nguoi thuc hien | Noi dung thay doi                           |
|------------|-----------------|---------------------------------------------|
| 2026-09-07 | devops-team     | Khoi tao repo, them cau hinh 3 service      |
| 2026-09-07 | devops-team     | Ma hoa toan bo mat khau bang {cipher}        |
```

---

## 5. Sơ đồ luồng hoạt động Centralized Configuration

```
+-------------------+         +-------------------+         +---------------------+
|   Config Git Repo  |         |   Config Server   |         |  FoodX Services     |
|                   |         |   :8888           |         |                     |
|  restaurant-      |         |                   |         |  restaurant-service |
|  service.yml      | <------ | Doc file tu Git   | ------> |  :8085              |
|  order-           |  clone  | Giai ma {cipher}  |         |                     |
|  service.yml      |         | Phuc vu qua HTTP  |         |  order-service      |
|  delivery-        |         |                   | ------> |  :8086              |
|  service.yml      |         |                   |         |                     |
|  application.yml  |         |                   | ------> |  delivery-service   |
+-------------------+         +-------------------+         |  :8087              |
                                       ^                    +---------------------+
                                       |
                              +--------+--------+
                              |  ENCRYPT_KEY    |
                              |  (bien moi      |
                              |   truong, khong |
                              |   luu trong Git)|
                              +-----------------+

Luong khi service khoi dong:
  1. Service doc bootstrap.yml -> biet dia chi Config Server va ten chinh minh
  2. Service gui GET http://config-server:8888/{ten-service}/{profile}
  3. Config Server doc file tuong ung tu Git repo
  4. Config Server giai ma tat ca gia tri '{cipher}...' bang ENCRYPT_KEY
  5. Config Server tra ve YAML da giai ma cho service
  6. Service nap cau hinh va tiep tuc khoi dong binh thuong
```

---

## 6. Tổng kết — Bảng so sánh trước và sau

| Tieu chi | Truoc (sai) | Sau (dung) |
|---|---|---|
| Ten file | `config.yml` | `restaurant-service.yml` |
| Khop voi spring.application.name | Khong khop | Khop chinh xac |
| Config Server tim thay file | Khong | Co |
| Mat khau | `RestaurantPass123` (plaintext) | `'{cipher}AQBXk3z9...'` (ma hoa) |
| Ai doc duoc mat khau | Bat ky ai co quyen doc Git | Chi Config Server co ENCRYPT_KEY |
| Mat khau trong git log | Lo vinh vien | Khong bao gio co mat khau that |
| Cau hinh dung chung | Khong co | `application.yml` phuc vu moi service |
| README mo ta cau truc | Khong co | Day du quy uoc va huong dan |

> **Nguyen tac then chot**: Ten file cau hinh trong Git repo PHAI khop chinh xac voi `spring.application.name` cua service can phuc vu. Moi gia tri nhay cam PHAI duoc ma hoa truoc khi commit — khoa giai ma chi ton tai tren Config Server, lay tu bien moi truong, khong bao gio commit vao bat ky repository nao.
