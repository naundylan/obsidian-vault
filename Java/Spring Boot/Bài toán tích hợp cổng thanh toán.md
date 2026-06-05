# Thanh Toán VietQR + SePay

## 1. Mục Tiêu

Ta muốn làm chức năng:

`Customer chọn ghế -> hệ thống giữ ghế tạm thời -> customer chuyển khoản qua QR -> SePay phát hiện tiền vào tài khoản ngân hàng -> SePay gọi webhook về BE -> BE xác nhận thanh toán -> BE tạo order/payment/ticket -> ghế chuyển SOLD -> FE chuyển sang trang thành công`

VietQR và SePay có vai trò khác nhau:

`VietQR = tạo mã QR để customer chuyển tiền SePay = đọc biến động số dư ngân hàng và báo về BE BE = kiểm tra giao dịch và tạo vé FE = hiển thị QR, chờ kết quả Redis = giữ session/ghế tạm thời DB = lưu order/payment/ticket thật`

## 2. Các Khái Niệm Chính

### VietQR Là Gì?

VietQR là chuẩn QR chuyển khoản ngân hàng.

Trong dự án này, ta dùng URL public:

`https://img.vietqr.io/image/{bankId}-{accountNo}-{template}.png`

Ví dụ:

`https://img.vietqr.io/image/ICB-103879953916-compact2.png?amount=10000&addInfo=SEVQR%20MCB-12345678&accountName=NGO%20LE%20NAM%20QUYEN`

QR này chứa:

- ngân hàng
- số tài khoản
- tên chủ tài khoản
- số tiền
- nội dung chuyển khoản

### SePay Là Gì?

SePay là bên trung gian đọc biến động số dư tài khoản ngân hàng.

Khi có tiền vào, SePay gửi HTTP request về backend của mình.

Ví dụ:

`POST /api/v1/customer/payments/vietqr/webhook`

### Webhook Là Gì?

Webhook là một URL của BE để bên thứ ba gọi vào.

Trong bài này:

`SePay đọc được tiền vào -> SePay gọi webhook URL của mình -> BE nhận payload giao dịch -> BE xác nhận thanh toán`

Webhook URL local qua tunnel:

`https://your-tunnel.trycloudflare.com/api/v1/customer/payments/vietqr/webhook`

Webhook URL production:

`https://api.yourdomain.com/api/v1/customer/payments/vietqr/webhook`

### Cloudflare Tunnel Là Gì?

Khi BE chạy local:

`http://localhost:8081`

SePay không thể gọi localhost của máy bạn.

Cloudflare Tunnel tạo một URL public:

`https://xxx.trycloudflare.com`

rồi chuyển request vào máy bạn:

`SePay -> https://xxx.trycloudflare.com -> cloudflared -> localhost:8081 -> BE`

Chạy tunnel:

`cloudflared tunnel --url http://localhost:8081`

### CORS Có Liên Quan Webhook Không?

Không.

CORS chỉ liên quan browser, ví dụ:

`FE localhost:3000 gọi BE localhost:8081`

Webhook là server gọi server:

`SePay server gọi BE server`

nên CORS không quyết định SePay có gọi được hay không.

### Payment Reference Là Gì?

Payment reference là mã thanh toán nội bộ để nối giao dịch ngân hàng với checkout session.

Ví dụ:

`paymentSessionId = 52956727-xxxx-... reference = MCB-52956727`

Customer chuyển khoản với nội dung:

`SEVQR MCB-52956727`

SePay gửi payload có nội dung đó về BE. BE parse MCB-52956727 để tìm session trong Redis.

### Vì Sao Cần SEVQR?

Với VietinBank trên SePay, SePay yêu cầu nội dung chuyển khoản bắt đầu bằng:

`SEVQR`

Nếu chỉ ghi:

`MCB-52956727`

thì tiền vẫn vào tài khoản nhưng SePay có thể không nhận diện/gửi webhook.

Vì vậy nội dung QR phải là:

`SEVQR MCB-XXXXXXXX`

BE vẫn parse được MCB-XXXXXXXX.

## 3. Env Cần Cấu Hình

### Backend Local: server/.env

Cần có:

`SERVER_PORT=8081 SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/ticket_booking_db SPRING_DATASOURCE_USERNAME=admin SPRING_DATASOURCE_PASSWORD=your_password SPRING_REDIS_HOST=localhost SPRING_REDIS_PORT=6379 SPRING_REDIS_PASSWORD= SPRING_REDIS_TIMEOUT=2s VIETQR_BANK_ID=ICB VIETQR_ACCOUNT_NO=your_real_bank_account VIETQR_ACCOUNT_NAME=YOUR_ACCOUNT_NAME VIETQR_TEMPLATE=compact2 SEPAY_API_KEY=your_sepay_api_key SEPAY_PAYMENT_PREFIX=SEVQR`

Ý nghĩa:

`SERVER_PORT -> port BE chạy, hiện dùng 8081 VIETQR_BANK_ID -> mã ngân hàng, VietinBank là ICB VIETQR_ACCOUNT_NO -> số tài khoản nhận tiền thật VIETQR_ACCOUNT_NAME -> tên chủ tài khoản VIETQR_TEMPLATE -> kiểu ảnh QR, dùng compact2 SEPAY_API_KEY -> API key SePay dùng để xác thực webhook SEPAY_PAYMENT_PREFIX -> tiền tố bắt buộc với VietinBank, hiện là SEVQR`

### Frontend Local: client/.env.local

`NEXT_PUBLIC_API_URL=http://localhost:8081/api/v1 NEXT_PUBLIC_SOCKET_URL=http://localhost:9092`

Ý nghĩa:

`NEXT_PUBLIC_API_URL -> FE gọi REST API BE NEXT_PUBLIC_SOCKET_URL -> FE kết nối socket realtime`

### Docker Compose Root .env

Nếu chạy bằng Docker Compose, cần:

`BACKEND_PORT=8081 NEXT_PUBLIC_API_URL=http://localhost:8081/api/v1 VIETQR_BANK_ID=ICB VIETQR_ACCOUNT_NO=your_real_bank_account VIETQR_ACCOUNT_NAME=YOUR_ACCOUNT_NAME VIETQR_TEMPLATE=compact2 SEPAY_API_KEY=your_sepay_api_key SEPAY_PAYMENT_PREFIX=SEVQR`

## 4. Các File Config Quan Trọng

### application.yml

File:

`server/src/main/resources/application.yml`

Cấu hình payment:

`application: payment: vietqr: bankId: ${VIETQR_BANK_ID:ICB} accountNo: ${VIETQR_ACCOUNT_NO:} accountName: ${VIETQR_ACCOUNT_NAME:} template: ${VIETQR_TEMPLATE:compact2} sepay: apiKey: ${SEPAY_API_KEY:} merchantAccount: ${VIETQR_ACCOUNT_NO:} paymentPrefix: ${SEPAY_PAYMENT_PREFIX:SEVQR}`

Cấu hình server port:

`server: port: ${SERVER_PORT:8081}`

Cấu hình Redis:

`spring: data: redis: host: ${SPRING_REDIS_HOST:localhost} port: ${SPRING_REDIS_PORT:6379} password: ${SPRING_REDIS_PASSWORD:} timeout: ${SPRING_REDIS_TIMEOUT:2s}`

### docker-compose.yml

Backend expose port:

`ports: - "${BACKEND_PORT:-8081}:8081"`

Backend env:

`SERVER_PORT: 8081 VIETQR_BANK_ID: ${VIETQR_BANK_ID:-ICB} VIETQR_ACCOUNT_NO: ${VIETQR_ACCOUNT_NO:-} VIETQR_ACCOUNT_NAME: ${VIETQR_ACCOUNT_NAME:-} VIETQR_TEMPLATE: ${VIETQR_TEMPLATE:-compact2} SEPAY_API_KEY: ${SEPAY_API_KEY:-} SEPAY_PAYMENT_PREFIX: ${SEPAY_PAYMENT_PREFIX:-SEVQR}`

FE env:

`NEXT_PUBLIC_API_URL: ${NEXT_PUBLIC_API_URL:-http://localhost:8081/api/v1} NEXT_PUBLIC_SOCKET_URL: ${NEXT_PUBLIC_SOCKET_URL:-http://localhost:9092}`

### AppSecurityConfig.java

File:

`server/src/main/java/com/concert/booking/config/AppSecurityConfig.java`

Webhook SePay phải public:

`static String[] PUBLIC_ENDPOINTS = { "/", "/api/v1/orders/webhooks/**", "/api/v1/customer/payments/vietqr/webhook", "/api/v1/payments/webhook" };`

Vì sao public?

Vì SePay là server bên ngoài, không đăng nhập bằng JWT customer được.

Nhưng public không có nghĩa là không bảo vệ. Bên trong webhook vẫn check:

`Authorization: Apikey {SEPAY_API_KEY}`

CORS:

`cors.setAllowedOriginPatterns( List.of( "http://localhost:3000", "http://localhost:*", "http://172.21.240.1:*", "https://*.app.github.dev"));`

CORS phục vụ FE gọi BE, không phục vụ webhook SePay.

## 5. Các File BE Cần Có

### PaymentMethod.java

File:

`server/src/main/java/com/concert/booking/modules/order/enums/PaymentMethod.java`

Cần có:

`public enum PaymentMethod { CASH, BANK_TRANSFER, VIETQR }`

Ý nghĩa:

`VIETQR là phương thức thanh toán online qua QR + SePay.`

### VietQrProperties.java

File:

`server/src/main/java/com/concert/booking/common/constants/VietQrProperties.java`

Nhiệm vụ:

- đọc env VIETQR_*
- cung cấp thông tin tạo QR

Nội dung chính:

`@ConfigurationProperties(prefix = "application.payment.vietqr") public class VietQrProperties { String bankId; String accountNo; String accountName; String template; }`

### SePayProperties.java

File:

`server/src/main/java/com/concert/booking/common/constants/SePayProperties.java`

Nhiệm vụ:

- đọc env SEPAY_*
- chứa API key và prefix

Nội dung chính:

`@ConfigurationProperties(prefix = "application.payment.sepay") public class SePayProperties { String apiKey; String merchantAccount; String paymentPrefix = "SEVQR"; }`

### VietQrService.java

File:

`server/src/main/java/com/concert/booking/modules/customerbooking/vietqr/VietQrService.java`

Nhiệm vụ:

- tạo mã MCB-XXXXXXXX
- lưu mapping Redis
- build QR URL
- chống trùng webhook SePay

Các hàm chính:

`String createPaymentReference(UUID paymentSessionId); void savePaymentReference(String reference, UUID paymentSessionId, Duration ttl); Optional<UUID> findPaymentSessionId(String reference); boolean markSePayWebhookProcessed(Long sePayId); String buildQrUrl(BigDecimal amount, String content);`

### VietQrServiceImpl.java

File:

`server/src/main/java/com/concert/booking/modules/customerbooking/vietqr/VietQrServiceImpl.java`

Redis keys:

`customer:payment-reference:{reference} customer:sepay:webhook:{sePayId}`

Tạo reference:

`return "MCB-" + paymentSessionId.toString().substring(0, 8).toUpperCase();`

Lưu mapping:

`customer:payment-reference:MCB-52956727 -> paymentSessionId`

Tạo QR URL:

`https://img.vietqr.io/image/{bankId}-{accountNo}-{template}.png ?amount={amount} &addInfo={content} &accountName={accountName}`

Trong đó content hiện là:

`SEVQR MCB-XXXXXXXX`

### SePayWebhookVerifier.java

File:

`server/src/main/java/com/concert/booking/modules/customerbooking/sepay/SePayWebhookVerifier.java`

Nhiệm vụ:

- định nghĩa cách verify webhook

`public interface SePayWebhookVerifier { boolean verify(String authorizationHeader); }`

### SePayApiKeyVerifier.java

File:

`server/src/main/java/com/concert/booking/modules/customerbooking/sepay/SePayApiKeyVerifier.java`

Nhiệm vụ:

- kiểm tra header SePay gửi về

SePay gửi:

`Authorization: Apikey YOUR_API_KEY`

Code check:

`return ("Apikey " + apiKey).equals(authorizationHeader);`

Nếu sai API key:

- reject request
- không tạo order

### SePayWebhookDTO.java

File:

`server/src/main/java/com/concert/booking/modules/customerbooking/dto/SePayWebhookDTO.java`

Payload SePay gửi:

`Long id; String gateway; String transactionDate; String accountNumber; String subAccount; String code; String content; String transferType; String description; BigDecimal transferAmount; BigDecimal accumulated; String referenceCode;`

Ví dụ payload thật:

`{ "gateway": "VietinBank", "transactionDate": "2026-06-04 10:40:11", "accountNumber": "103879953916", "code": null, "content": "CT DEN:460T266056BKMFM2 SEVQR MCBAD06373C", "transferType": "in", "description": "BankAPINotify CT DEN:460T266056BKMFM2 SEVQR MCBAD06373C", "transferAmount": 10000, "referenceCode": "460T266056BKMFM2", "accumulated": 80000, "id": 61782030 }`

Lưu ý:

- SePay/VietinBank có thể trả MCBAD06373C, không có dấu -.
- BE phải parse được cả:
    
    `MCBAD06373C MCB-AD06373C`
    

### VietQrPaymentDTO.java

File:

`server/src/main/java/com/concert/booking/modules/customerbooking/dto/VietQrPaymentDTO.java`

BE trả cho FE:

`String qrUrl; String bankId; String accountNo; String accountName; BigDecimal amount; String content; Instant expiredAt;`

FE dùng để hiển thị QR.

### CheckoutPaymentStatusDTO.java

File:

`server/src/main/java/com/concert/booking/modules/customerbooking/dto/CheckoutPaymentStatusDTO.java`

FE polling nhận trạng thái:

`String status; PaymentMethod paymentMethod; UUID orderId;`

Status:

`PENDING PAID EXPIRED FAILED`

## 6. Controller Cần Có

File:

`server/src/main/java/com/concert/booking/modules/customerbooking/CustomerBookingV1Controller.java`

### Tạo QR VietQR

`@PostMapping("/checkout/{paymentSessionId}/vietqr") @PreAuthorize("hasRole('CUSTOMER')") public DataApiResponse<VietQrPaymentDTO> createVietQrPayment( @PathVariable UUID paymentSessionId) { UUID customerId = AuthUtils.getCurrentUserId(); return DataApiResponse.success( customerBookingService.createVietQrPayment(paymentSessionId, customerId), "Tao thong tin thanh toan VietQR thanh cong"); }`

FE gọi API này khi customer bấm:

`Thanh toán qua VietQR`

### Polling Payment Status

`@GetMapping("/checkout/{paymentSessionId}/payment-status") @PreAuthorize("hasRole('CUSTOMER')") public DataApiResponse<CheckoutPaymentStatusDTO> getCheckoutPaymentStatus( @PathVariable UUID paymentSessionId) { UUID customerId = AuthUtils.getCurrentUserId(); return DataApiResponse.success( customerBookingService.getCheckoutPaymentStatus(paymentSessionId, customerId), "Lay trang thai thanh toan thanh cong"); }`

FE gọi mỗi 3 giây để hỏi:

`Thanh toán xong chưa?`

### Webhook SePay

`@PostMapping("/payments/vietqr/webhook") public Map<String, Boolean> handleSePayWebhook( @RequestHeader(value = "Authorization", required = false) String authorizationHeader, @RequestBody SePayWebhookDTO payload) { return customerBookingService.handleSePayWebhook(authorizationHeader, payload); }`

SePay gọi API này khi có tiền vào.

## 7. Logic Service Quan Trọng

File:

`server/src/main/java/com/concert/booking/modules/customerbooking/CustomerBookingServiceImpl.java`

### Tạo VietQR

Logic:

`1. Lấy checkout session theo paymentSessionId 2. Check session thuộc đúng customer 3. Check ghế vẫn do session này hold 4. Tạo reference MCB-XXXXXXXX 5. Lưu Redis reference -> paymentSessionId 6. Build nội dung SEVQR MCB-XXXXXXXX 7. Build QR URL 8. Trả về FE`

Code ý chính:

`String reference = vietQrService.createPaymentReference(paymentSessionId); vietQrService.savePaymentReference(reference, paymentSessionId, CHECKOUT_TTL); String paymentContent = buildSePayPaymentContent(reference); return VietQrPaymentDTO.builder() .qrUrl(vietQrService.buildQrUrl(session.getTotalAmount(), paymentContent)) .content(paymentContent) .build();`

### Build Nội Dung Chuyển Khoản

`private String buildSePayPaymentContent(String reference) { String prefix = sePayProperties.getPaymentPrefix(); if (prefix == null || prefix.isBlank()) { return reference; } return prefix.trim() + " " + reference; }`

Kết quả:

`SEVQR MCB-52956727`

### Nhận Webhook SePay

Logic:

`1. Verify API key 2. Bỏ qua nếu payload null 3. Chỉ xử lý transferType = in 4. Check đúng accountNumber 5. Parse MCB reference từ code/content/description 6. Check payment đã tồn tại chưa 7. Lookup Redis reference -> paymentSessionId 8. Check session còn hạn 9. Check số tiền khớp 10. Gọi completePaidCheckoutSession 11. Mark webhook processed 12. Return {"success": true}`

### Parse Reference

Do SePay có thể trả:

`MCB-52956727 MCB52956727`

nên regex cần nhận cả hai:

`private static final Pattern VIETQR_REFERENCE_PATTERN = Pattern.compile("MCB-?[A-Z0-9]{8}");`

Chuẩn hóa:

`String rawReference = matcher.group(); return rawReference.startsWith("MCB-") ? rawReference : "MCB-" + rawReference.substring("MCB".length());`

### Complete Paid Checkout Session

Hàm này là trung tâm tạo vé:

`completePaidCheckoutSession(paymentSessionId, customerId, PaymentMethod.VIETQR, reference)`

Nó làm:

`1. Check payment transactionRef đã tồn tại chưa 2. Lấy checkout session 3. Check session còn hạn 4. Check session còn giữ ghế 5. Lock ghế trong DB 6. Tạo Order 7. Tạo Payment 8. Tạo Ticket 9. Đổi Seat thành SOLD 10. Xóa Redis hold/session 11. Emit socket seat-sold`

## 8. Redis Cần Dùng

### Checkout Session

`customer:checkout:session:{paymentSessionId}`

Lưu toàn bộ thông tin checkout tạm thời.

TTL:

`10 phút`

### Seat Hold

`customer:seat-hold:{eventId}:{seatId}`

Giữ ghế tạm thời.

Value:

`paymentSessionId`

### Payment Reference

`customer:payment-reference:{reference}`

Ví dụ:

`customer:payment-reference:MCB-52956727`

Value:

`paymentSessionId`

Dùng để webhook tìm lại checkout session.

### SePay Webhook Processed

`customer:sepay:webhook:{sePayId}`

Dùng để tránh xử lý lặp webhook cùng id.

## 9. DB Cần Có

### payments.transaction_ref

Cần lưu:

`MCB-XXXXXXXX`

Mục đích:

- một giao dịch chỉ tạo một payment
- webhook retry không tạo order trùng

Nên có unique constraint:

`ALTER TABLE payments ADD CONSTRAINT uk_payment_transaction_ref UNIQUE (transaction_ref);`

### payments.payment_method Constraint

DB phải cho phép:

`VIETQR`

Nếu gặp lỗi:

`violates check constraint payments_payment_method_check`

thì sửa DB:

`ALTER TABLE payments DROP CONSTRAINT IF EXISTS payments_payment_method_check; ALTER TABLE payments ADD CONSTRAINT payments_payment_method_check CHECK (payment_method IN ('CASH', 'BANK_TRANSFER', 'VIETQR'));`

### tickets.seat_id

Nên unique:

`seat_id UNIQUE`

Mục đích:

- một ghế chỉ có một ticket
- tránh double booking ở tầng DB

## 10. File FE Cần Có

### client/lib/axios.ts

`baseURL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8081/api/v1'`

### customer-booking.type.ts

Cần type:

`export type CustomerPaymentMethod = 'BANK_TRANSFER' | 'VIETQR' export interface VietQrPaymentDTO { qrUrl: string bankId: string accountNo: string accountName: string amount: number content: string expiredAt: string } export interface CheckoutPaymentStatusDTO { status: 'PENDING' | 'PAID' | 'EXPIRED' | 'FAILED' paymentMethod: CustomerPaymentMethod orderId?: string | null }`

### customer-booking.service.ts

Cần API:

`createVietQrPayment(paymentSessionId) getCheckoutPaymentStatus(paymentSessionId)`

Ví dụ:

``createVietQrPayment: async (paymentSessionId: string): Promise<VietQrPaymentDTO> => { const response = await axiosClient.post<{ data: VietQrPaymentDTO }>( `/customer/checkout/${paymentSessionId}/vietqr` ) return response.data }``

``getCheckoutPaymentStatus: async ( paymentSessionId: string ): Promise<CheckoutPaymentStatusDTO> => { const response = await axiosClient.get<{ data: CheckoutPaymentStatusDTO }>( `/customer/checkout/${paymentSessionId}/payment-status` ) return response.data }``

### Checkout Page

File:

`client/app/customer/booking/[eventId]/checkout/page.tsx`

Cần có:

1. State lưu QR:

`const [vietQrPayment, setVietQrPayment] = useState<VietQrPaymentDTO | null>(null) const [waitingVietQr, setWaitingVietQr] = useState(false)`

2. Hàm tạo QR:

`const payWithVietQr = async () => { const result = await customerBookingService.createVietQrPayment(session.paymentSessionId) setVietQrPayment(result) setWaitingVietQr(true) }`

3. Polling:

``useEffect(() => { if (!session || !waitingVietQr || countdown.isExpired) return const timer = window.setInterval(async () => { const status = await customerBookingService.getCheckoutPaymentStatus(session.paymentSessionId) if (status.status === 'PAID' && status.orderId) { window.clearInterval(timer) router.replace(`/customer/booking/${eventId}/success?orderId=${status.orderId}`) } if (status.status === 'EXPIRED' || status.status === 'FAILED') { window.clearInterval(timer) setWaitingVietQr(false) } }, 3000) return () => window.clearInterval(timer) }, [session, waitingVietQr])``

4. UI hiển thị:
    - QR image
    - ngân hàng
    - số tài khoản
    - tên tài khoản
    - số tiền
    - nội dung chuyển khoản
    - nút copy

Nội dung phải hiện:

`SEVQR MCB-XXXXXXXX`

Không chỉ MCB-XXXXXXXX.

## 11. Cấu Hình SePay Dashboard

Webhook URL local qua Cloudflare:

`https://your-tunnel.trycloudflare.com/api/v1/customer/payments/vietqr/webhook`

Webhook URL production:

`https://api.yourdomain.com/api/v1/customer/payments/vietqr/webhook`

Cấu hình:

- Sự kiện: Có tiền vào
- Tài khoản: tài khoản ngân hàng nhận tiền
- Bảo mật: API Key
- API Key: giống SEPAY_API_KEY trong BE
- Nếu có filter prefix: dùng SEVQR hoặc không filter trong giai đoạn test

Với VietinBank, cần chú ý:

- nội dung chuyển khoản phải bắt đầu bằng SEVQR
- QR nên có addInfo=SEVQR MCB-XXXXXXXX

## 12. Luồng Hoàn Chỉnh

`1. Customer chọn ghế 2. FE gọi POST /checkout 3. BE tạo paymentSessionId 4. BE giữ ghế bằng Redis 10 phút 5. Customer bấm Thanh toán VietQR 6. FE gọi POST /checkout/{paymentSessionId}/vietqr 7. BE tạo reference MCB-XXXXXXXX 8. BE lưu Redis: reference -> paymentSessionId 9. BE tạo QR với content: SEVQR MCB-XXXXXXXX 10. FE hiển thị QR 11. Customer chuyển khoản 12. SePay nhận biến động số dư 13. SePay gọi webhook về BE 14. BE verify API Key 15. BE parse MCB-XXXXXXXX từ content/description 16. BE tìm paymentSessionId trong Redis 17. BE check session còn hạn 18. BE check amount đúng 19. BE tạo order/payment/ticket 20. BE đổi ghế SOLD 21. BE emit socket 22. FE polling thấy PAID 23. FE redirect success page`

## 13. Các Lỗi Thường Gặp

### VietQR chua duoc cau hinh

Thiếu env:

`VIETQR_BANK_ID VIETQR_ACCOUNT_NO VIETQR_ACCOUNT_NAME VIETQR_TEMPLATE`

### SePay không thấy giao dịch

Với VietinBank, nội dung không bắt đầu bằng:

`SEVQR`

Cần QR content:

`SEVQR MCB-XXXXXXXX`

### SePay có webhook nhưng FE vẫn PENDING

Kiểm tra:

- payload có MCB... không
- Redis còn mapping không
- session còn hạn không
- amount có đúng không
- accountNumber có đúng không
- DB constraint có cho VIETQR không

### Lỗi DB payments_payment_method_check

DB constraint chưa cho phép VIETQR.

Sửa:

`ALTER TABLE payments DROP CONSTRAINT IF EXISTS payments_payment_method_check; ALTER TABLE payments ADD CONSTRAINT payments_payment_method_check CHECK (payment_method IN ('CASH', 'BANK_TRANSFER', 'VIETQR'));`

### Lỗi tunnel 502

Tunnel trỏ sai port.

Sau khi đổi BE sang 8081, chạy:

`cloudflared tunnel --url http://localhost:8081`

## 14. Những Việc Nên Làm Thêm Nếu Production

1. Migration chính thức bằng Flyway/Liquibase.
2. Bảng sepay_webhooks để lưu payload webhook.
3. Admin screen xử lý giao dịch lỗi.
4. Processing lock cho webhook.
5. Queue/Kafka cho payment event.
6. Monitoring/reconciliation đối soát giao dịch ngân hàng với order/payment.

## 15. Checklist Khi Làm Lại Chức Năng Tương Tự

`[ ] Có phương thức payment enum [ ] Có env payment gateway [ ] Có properties class đọc env [ ] Có service tạo payment info/QR [ ] Có webhook DTO [ ] Có verifier cho webhook [ ] Có webhook endpoint public [ ] Có API tạo payment cho FE [ ] Có API polling/payment status [ ] Có Redis mapping reference -> session [ ] Có transactionRef trong Payment [ ] Có unique constraint transactionRef [ ] Có logic completePaidCheckoutSession [ ] Có FE hiển thị payment info [ ] Có FE polling status [ ] Có tunnel/domain public cho webhook [ ] Có cấu hình dashboard bên thứ ba [ ] Có test webhook giả lập [ ] Có test chuyển khoản thật`