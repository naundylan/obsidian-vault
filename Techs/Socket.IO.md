Cơ chế hoạt động của Socket hoạt động theo các khái niệm sau:
- Room:
	- Server sẽ tự tạo các room dựa vào đường dẫn riêng
	- Ví dụ nếu bạn truy cập Url để xem chi tiết 1 sự kiện nào đó: /booking/:eventId
	- Thì lúc đó có nghĩa bạn đang join 1 room do websocket tạo ra
- Emit = gửi tin nhắn:
	- Client emit lên server: socket.emit('join-event', { eventId })
	- Server emit xuống room: io.to('event:abc').emit('seat-held', { seatIds })
- Events = tên của tin nhắn:
	- 'join-event'    → client nói với server "cho tôi vào room này"
	- 'seat-held'     → server nói với room "ghế này bị giữ rồi"
	- 'seat-released' → server nói với room "ghế này free rồi"
	- 'seat-sold'     → server nói với room "ghế này bán rồi"
- Luồng thực tế:

> [!NOTE]
> 1. User mở trang /booking/:eventId/seats
>    → FE kết nối socket server
>    → FE emit 'join-event' { eventId: "abc" }
>    → Server thêm user vào room "event:abc"
>    → Server emit 'seat-snapshot' về cho user đó
> 
> 2. User A checkout ghế A1, A2
>    → BE xử lý Lua script thành công
>    → BE gọi: io.to("event:abc").emit("seat-held", { seatIds: ["A1","A2"] })
>    → TẤT CẢ user trong room nhận được
>    → FE của mỗi user patch ghế A1, A2 → màu "đang bị giữ"
> 
> 3. User B đang chọn ghế A1
>    → FE nhận socket event seat-held có A1
>    → Tự động bỏ chọn A1
>    → Hiện thông báo "Ghế A1 vừa bị người khác giữ"
> 
> 4. User rời trang
>    → Socket disconnect
>    → Server tự remove khỏi room

# 1. Socket Là Gì Trong App Này?

REST API giống như:

text

`FE hỏi -> BE trả lời`

Ví dụ:

text

`GET /customer/events/{eventId}/catalog`

FE chủ động gọi, BE trả dữ liệu.

Socket thì giống như một đường dây mở liên tục:

text

`FE kết nối với BE BE có thể chủ động báo tin ngược lại FE FE cũng có thể gửi tín hiệu nhỏ cho BE`

Use case trong booking:

text

`User A giữ ghế A1 -> BE báo ngay cho User B đang xem cùng event -> User B thấy A1 chuyển sang "Đang giữ"`

Nếu chỉ dùng REST API, User B phải bấm reload hoặc polling liên tục mới biết.

---

# 2. Muốn Làm Socket Cần Những Gì?

Cần cả FE và BE.

## BE Cần

1. Dependency socket server.
2. File config để bật socket server.
3. DTO cho dữ liệu gửi/nhận.
4. Service/handler để:
    - nhận event từ FE
    - cho client vào room
    - emit event về FE

Trong project này BE có:

text

`SocketIoConfig.java SocketIoProperties.java SeatSocketService.java JoinEventRequest.java SeatHeldSocketEvent.java SeatChangedSocketEvent.java`

## FE Cần

1. Dependency socket client.
2. Hook/service để connect socket.
3. Code emit event.
4. Code listen event.
5. Page gọi hook đó khi cần realtime.

Trong project này FE có:

text

`useSeatsSocket.ts`

Và page gọi nó ở màn chọn ghế:

text

`/customer/booking/[eventId]/seats`

---

# 3. Socket BE Cấu Trúc Như Thế Nào?

## Dependency

BE có:

gradle

`implementation 'com.corundumstudio.socketio:netty-socketio:2.0.14'`

Ý nghĩa:

Spring Boot không tự chạy Socket.IO server được, nên cần thư viện này.

## Config File

SocketIoConfig.java (line 21)

Nhiệm vụ:

text

`Tạo Socket.IO server Chạy nó ở port 9092 Cho phép FE localhost:3000 connect Start server khi Spring Boot chạy Stop server khi Spring Boot tắt`

## Properties File

SocketIoProperties.java (line 15)

Nhiệm vụ:

text

`Đọc SOCKET_HOST Đọc SOCKET_PORT Đọc SOCKET_ORIGIN`

Để sau này deploy chỉ đổi env, không sửa code.

---

# 4. Socket Service Là Gì?

SeatSocketService.java (line 19)

Đây là file xử lý event socket của phần ghế.

Nó làm 2 nhóm việc.

## Nhóm 1: Nhận Event Từ FE

FE gửi:

ts

`socket.emit('join-event', { eventId })`

BE nhận:

java

`socketIOServer.addEventListener("join-event", ...)`

Sau đó BE:

text

`kiểm tra eventId cho client vào room event:{eventId} gửi seat-snapshot về client`

## Nhóm 2: Emit Event Về FE

BE có các hàm:

java

`emitSeatHeld(...) emitSeatReleased(...) emitSeatSold(...)`

Các hàm này không tự chạy. Chúng được gọi từ business service.

Ví dụ trong checkout:

java

`seatSocketService.emitSeatHeld(eventId, seatIds, expiresAt);`

---

# 5. Room Trong Socket Là Gì?

Room là nhóm client.

Ví dụ có 100 người đang xem nhiều event khác nhau:

text

`event:111 -> 20 người event:222 -> 40 người event:333 -> 40 người`

Khi ghế của event 111 thay đổi, BE chỉ emit cho room:

text

`event:111`

Không gửi nhầm sang event khác.

Trong code:

java

`client.joinRoom("event:" + eventId);`

Và emit:

java

`socketIOServer.getRoomOperations("event:" + eventId).sendEvent(...)`

---

# 6. Socket FE Hoạt Động Thế Nào?

useSeatsSocket.ts (line 20)

Hook này làm:

text

`connect tới NEXT_PUBLIC_SOCKET_URL khi connect xong thì emit join-event listen seat-snapshot listen seat-held listen seat-released listen seat-sold disconnect khi rời page`

Luồng:

text

`User vào trang ghế -> React render page -> page gọi useSeatsSocket -> hook connect socket -> hook emit join-event -> BE cho client vào room -> BE gửi snapshot`

FE không có route socket kiểu REST. Nó kích hoạt socket vì page có gọi hook.

---

# 7. Socket Có Phải URL Không?

Không giống REST.

REST dùng route:

java

`@GetMapping("/customer/events/{eventId}")`

Socket dùng event name:

text

`join-event seat-snapshot seat-held seat-released seat-sold`

Nên “route” của socket chính là tên event.

FE:

ts

`socket.emit('join-event', data)`

BE:

java

`addEventListener("join-event", ...)`

---

# 8. Công Thức Làm Socket Cho Tính Năng Khác

Ví dụ sau này muốn realtime notification.

BE:

text

`NotificationSocketService NotificationSocketDTO emitNotification(...) join-user-room`

FE:

text

`useNotificationsSocket socket.emit('join-user', { userId }) socket.on('notification-created', ...)`

Công thức chung:

text

`1. Cài dependency FE + BE 2. BE tạo socket config 3. BE tạo DTO event 4. BE tạo socket service 5. FE tạo hook/service socket 6. Page nào cần realtime thì gọi hook 7. Business service khi có thay đổi thì gọi emit`

---

# 9. Redis Là Gì Trong App Này?

Redis chỉ nằm ở BE.

FE không nói chuyện trực tiếp với Redis.

FE chỉ gọi API:

text

`POST /customer/checkout GET /customer/checkout/{paymentSessionId} DELETE /customer/checkout/{paymentSessionId} POST /customer/checkout/{paymentSessionId}/confirm-dev`

BE mới là bên đọc/ghi Redis.

Redis dùng để lưu dữ liệu tạm:

text

`checkout session ghế đang giữ`

---

# 10. Vì Sao Cần Redis?

Vì booking có khái niệm:

text

`giữ ghế 10 phút`

Nếu lưu vào DB thì có vấn đề:

text

`nhiều user chọn/hủy liên tục DB bị ghi nhiều phải tự viết logic dọn dữ liệu hết hạn trạng thái HELD chỉ là tạm thời`

Redis có TTL:

text

`lưu key 10 phút sau 10 phút tự hết hạn`

Nên rất hợp để giữ ghế tạm.

---

# 11. Muốn Dùng Redis Cần Những Gì?

BE cần:

1. Dependency Redis.
2. Config kết nối Redis.
3. RedisConfig để tạo template.
4. Redis service để đọc/ghi/xóa key.
5. Gắn Redis service vào business flow.

Trong project này có:

text

`RedisConfig.java CheckoutSessionRedisService.java CheckoutSessionRedisServiceImpl.java SeatHoldRedisService.java SeatHoldRedisServiceImpl.java`

---

# 12. Redis Dependency

server/build.gradle (line 58)

gradle

`implementation 'org.springframework.boot:spring-boot-starter-data-redis'`

Nó cho Spring Boot dùng Redis.

---

# 13. Redis Config

application.yml (line 65)

yaml

`spring: data: redis: host: ${SPRING_REDIS_HOST:localhost} port: ${SPRING_REDIS_PORT:6379} password: ${SPRING_REDIS_PASSWORD:} timeout: ${SPRING_REDIS_TIMEOUT:2s}`

Local:

text

`localhost:6379`

Docker:

text

`redis:6379`

---

# 14. RedisConfig File

RedisConfig.java (line 10)

Nhiệm vụ:

text

`Tạo RedisTemplate để service dùng đọc/ghi Redis`

Nó config:

text

`key dạng string value dạng JSON`

---

# 15. Redis Service Cho Checkout Session

CheckoutSessionRedisServiceImpl.java (line 1)

Service này có:

java

`save(...) findById(...) delete(...)`

Nó lưu key:

text

`customer:checkout:session:{paymentSessionId}`

Use case:

text

`User vào checkout -> BE tạo session -> lưu session vào Redis 10 phút`

Refresh trang:

text

`FE gửi paymentSessionId -> BE findById trong Redis -> trả session về FE`

Hủy/thanh toán xong:

text

`BE delete session`

---

# 16. Redis Service Cho Seat Hold

SeatHoldRedisServiceImpl.java (line 1)

Service này có:

java

`holdSeats(...) releaseSeats(...) getHeldSeatIds(...) hasAnyHeldSeat(...) ownsAllHeldSeats(...)`

Redis key chính:

text

`customer:seat-hold:{eventId}:{seatId}`

Value:

text

`paymentSessionId`

Nghĩa là:

text

`Ghế này đang được giữ bởi session nào`

---

# 17. setIfAbsent Là Gì?

Đây là điểm quan trọng nhất.

Khi giữ ghế:

java

`setIfAbsent(key, value, ttl)`

Nó nghĩa là:

text

`Nếu key chưa tồn tại -> tạo key -> giữ ghế thành công Nếu key đã tồn tại -> không ghi đè -> giữ ghế thất bại`

Ví dụ:

text

`User A giữ A1 -> Redis tạo key customer:seat-hold:event1:A1 User B cũng giữ A1 -> Redis thấy key đã tồn tại -> BE trả 409 Conflict`

Đây là cách tránh 2 người cùng giữ một ghế.

---

# 18. Redis Được Gắn Vào Business Flow Như Thế Nào?

Trước đây service chỉ kiểu:

text

`repo lấy data build DTO save DB return response`

Bây giờ flow có thêm Redis và socket:

text

`repo kiểm tra ghế thật trong DB Redis kiểm tra ghế đang bị hold không Redis hold ghế Redis lưu session Socket emit seat-held return session`

Tức là Redis nằm giữa bước kiểm tra và bước tạo kết quả.

---

# 19. Flow Checkout Hiện Tại

CustomerBookingServiceImpl.java (line 131)

text

`FE gọi POST /customer/checkout -> BE kiểm tra paymentMethod -> BE kiểm tra event ONSALE -> BE kiểm tra ghế có bị hold trong Redis không -> BE lock ghế DB để chắc ghế còn AVAILABLE -> BE tính tổng tiền -> BE tạo paymentSessionId -> BE lưu seat hold Redis 10 phút -> BE lưu checkout session Redis 10 phút -> BE emit socket seat-held -> FE nhận checkout session`

---

# 20. Flow Get Session

text

`FE refresh trang checkout -> FE gọi GET /customer/checkout/{paymentSessionId} -> BE lấy session từ Redis -> BE kiểm tra session thuộc customer hiện tại -> BE kiểm tra ghế vẫn đang được giữ bởi session đó -> trả session`

Nếu không còn:

text

`BE trả 410 Gone FE quay về chọn ghế`

---

# 21. Flow Release Session

text

`User hủy checkout -> FE gọi DELETE /customer/checkout/{paymentSessionId} -> BE lấy session -> BE xóa seat hold Redis -> BE xóa checkout session Redis -> BE emit socket seat-released`

---

# 22. Flow Confirm Dev Payment

text

`User bấm giả lập đã thanh toán -> FE gọi POST /customer/checkout/{paymentSessionId}/confirm-dev -> BE lấy session Redis -> BE kiểm tra session hợp lệ -> BE lock ghế DB -> BE tạo Order -> BE tạo Payment -> BE tạo Ticket -> BE đổi ghế SOLD -> BE xóa Redis session/hold -> BE emit socket seat-sold -> FE sang trang success`

---

# 23. Redis Và Socket Phối Hợp Như Thế Nào?

Redis lưu trạng thái tạm.

Socket báo trạng thái tạm đó cho FE.

Ví dụ:

text

`BE lưu hold vào Redis -> BE emit seat-held -> FE khác đổi UI sang HELD`

Nếu không có Redis:

text

`BE không biết ghế nào đang bị giữ`

Nếu không có socket:

text

`Redis vẫn giữ ghế, nhưng FE khác không biết ngay`

Nên:

text

`Redis = nguồn dữ liệu tạm Socket = kênh báo tin realtime DB = nguồn dữ liệu chính thức`

---

# 24. Khi Nào Dùng Redis?

Dùng Redis khi dữ liệu:

text

`tạm thời cần hết hạn cần nhanh không cần lưu vĩnh viễn dễ tính bằng key/value`

Ví dụ hợp:

text

`checkout session seat hold OTP reset password token rate limit cache danh sách hot online user presence`

Không nên dùng Redis làm nguồn chính cho:

text

`order thật payment thật ticket thật user thật event thật`

Các cái đó phải vào DB.

---

# 25. Khi Nào Dùng Socket?

Dùng socket khi FE cần biết thay đổi ngay.

Ví dụ hợp:

text

`ghế vừa bị giữ ghế vừa bán notification mới chat trạng thái đơn hàng realtime dashboard live`

Không cần socket nếu chỉ là:

text

`mở trang -> lấy dữ liệu một lần form submit bình thường admin chỉnh dữ liệu ít khi thay đổi`

---

# 26. Công Thức Làm Redis Cho Tính Năng Khác

Ví dụ muốn làm OTP:

text

`1. Thêm key pattern: otp:{phone} 2. Service saveOtp(phone, code, ttl) 3. Service getOtp(phone) 4. Service deleteOtp(phone) 5. AuthService gọi Redis service`

Ví dụ muốn làm rate limit:

text

`1. Key: rate-limit:login:{ip} 2. Increment count 3. Set TTL 4. Nếu count vượt giới hạn thì chặn`

Công thức chung:

text

`1. Xác định dữ liệu tạm là gì 2. Đặt key Redis rõ ràng 3. Xác định TTL 4. Viết service save/get/delete/check 5. Inject service đó vào business service 6. Không để controller gọi Redis trực tiếp nếu không cần`

---

# 27. Công Thức Kết Hợp Redis + Socket

Dùng khi có trạng thái tạm cần realtime.

Ví dụ booking ghế:

text

`1. Redis lưu trạng thái tạm 2. Socket báo trạng thái tạm 3. DB lưu kết quả cuối cùng`

Flow mẫu:

text

`User action -> REST API vào BE -> BE check DB -> BE check Redis -> BE update Redis -> BE emit socket -> FE khác cập nhật UI`

Khi hoàn tất:

text

`BE update DB -> BE xóa Redis -> BE emit socket`

---

# 28. Tổng Kết Ngắn

Socket:

text

`FE + BE đều cần code FE connect/listen/emit BE config server/listen/emit/room Dùng để báo realtime`

Redis:

text

`Chỉ BE dùng Cần dependency + config + service Dùng để lưu dữ liệu tạm có TTL`

Trong project này:

text

`Redis giữ ghế/session Socket báo ghế held/released/sold DB lưu order/payment/ticket/seat sold`