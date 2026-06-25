### 1. Tầng 1: UI Layer (Client-Side Protection)

- **Cách xử lý**:
    - Ngay khi người dùng nhấn nút "Thanh toán" (Checkout), button lập tức bị `disabled` và hiển thị trạng thái loading nhằm **chặn hoàn toàn Double Click** (gửi nhiều request đồng thời do người dùng sốt ruột).
    - Sử dụng **WebSockets (Socket.IO)** để truyền trạng thái ghế theo thời gian thực. Khi người dùng A vừa click giữ ghế, sự kiện `emitSeatHeld` được gửi đi, màn hình của người dùng B lập tức cập nhật màu ghế đó sang trạng thái khóa (màu xám), ngăn chặn việc B click chọn trùng.
- **Ý nghĩa**: Giảm thiểu tối đa lượng request rác/trùng lặp gửi lên server ngay từ đầu phễu.

---

### 2. Tầng 2: API / Controller Layer (Gatekeeping & Idempotency)

- **Cách xử lý**:
    - **Request Validation**: Sử dụng bộ thư viện `@Valid` của Spring Boot để kiểm tra tính hợp lệ của dữ liệu đầu vào (UUID đúng định dạng, danh sách ghế không rỗng).
    - **Khử trùng lặp giao dịch (Idempotency Lock)**:
        - Trong luồng nhận Webhook thanh toán tự động (như từ cổng SePay/VietQR), hệ thống lấy `sePayId` (mã giao dịch duy nhất từ ngân hàng) làm khóa tạm thời trên Redis thông qua markSePayWebhookProcessed:
            
            java
            
            boolean marked = vietQrService.markSePayWebhookProcessed(payload.getId());
            
        - Hàm này thực hiện lệnh ghi khóa atomic `setIfAbsent` trên Redis. Nếu request Webhook bị gửi trùng lặp (do mạng chập chờn hoặc ngân hàng retry), luồng thứ 2 sẽ bị từ chối ngay ở bước này, ngăn chặn việc tạo 2 đơn hàng cho cùng một giao dịch chuyển khoản.

---

### 3. Tầng 3: Service Layer (Transactional Boundary)

- **Cách xử lý**:
    - Sử dụng annotation **`@Transactional`** của Spring Framework cho các phương thức quan trọng như `checkout` hay `completePaidCheckoutSession`.
    - Tất cả các hành động: Kiểm tra ghế -> Khóa ghế trong DB -> Lưu Order -> Lưu Payment -> Sinh Ticket được gộp chung vào một transaction. Nếu có bất kỳ lỗi nào xảy ra ở bất kỳ dòng code nào, hệ thống sẽ thực hiện **Rollback** toàn bộ transaction về trạng thái cũ, đảm bảo dữ liệu không bao giờ bị rơi vào trạng thái lấp lửng (ví dụ: tạo order thành công nhưng ghế vẫn trống, hoặc ngược lại).
    - **Transaction Synchronization**: Hệ thống đăng ký đồng bộ hóa transaction qua `TransactionSynchronizationManager`. Các hoạt động xóa session và bắn WebSocket thông báo ghế đã bán (`emitSeatSold`) chỉ được kích hoạt **sau khi Database đã commit thành công** (`afterCommit`), tránh tình trạng báo ảo cho client khi DB bị lỗi rollback.

---

### 4. Tầng 4: Repository Layer (Pessimistic Locking & Deadlock Prevention)

- **Cách xử lý**:
    - Hệ thống sử dụng **Pessimistic Write Lock (Khóa bi quan - `FOR UPDATE`)** tại SeatRepository:
        
        java
        
        @Lock(LockModeType.PESSIMISTIC_WRITE)
        
        @Query("SELECT s FROM Seat s WHERE s.id IN :seatIds ORDER BY s.id")
        
        List<Seat> findAllByIdForUpdate(List<UUID> seatIds);
        
    - Khi luồng xử lý gọi hàm này, Database PostgreSQL sẽ khóa chặt các dòng ghế tương ứng. Không một luồng song song nào khác được phép đọc để cập nhật hoặc sửa đổi các bản ghi ghế này cho tới khi transaction hiện tại kết thúc.
    - **Ngăn chặn Deadlock (Cực kỳ quan trọng)**:
        
        IMPORTANT
        
        **Điểm nhấn chuyên môn:** Khi khóa nhiều dòng dữ liệu, nếu Luồng 1 khóa ghế A trước rồi đến ghế B, còn Luồng 2 khóa ghế B trước rồi đến ghế A, hệ thống sẽ rơi vào trạng thái **Deadlock** (chờ vòng lặp vô hạn). Để giải quyết, dự án bắt buộc **sắp xếp tăng dần các ID ghế (`ORDER BY s.id` hoặc `sorted()`)** trước khi thực hiện khóa. Vì các ID được khóa theo một thứ tự duy nhất và nhất quán, deadlock hoàn toàn bị triệt tiêu về mặt toán học.
        

---

### 5. Tầng 5: Optimistic Locking Layer (Double-Check & Lost Updates)

- **Cách xử lý**:
    - Các Class Entity chính như Seat hay `SeatLayout` đều khai báo thuộc tính `@Version int version;` được quản lý bởi Hibernate.
    - Khi Hibernate thực hiện cập nhật trạng thái ghế: `UPDATE seats SET status = 'SOLD', version = version + 1 WHERE id = ? AND version = <current_version>`
    - Nếu một tiến trình khác đã âm thầm cập nhật bản ghi này trước đó (khiến số version tăng lên), câu lệnh UPDATE sẽ trả về số dòng bị tác động là `0`. Hibernate sẽ lập tức ném ra `OptimisticLockException`, và Spring Boot sẽ rollback transaction để bảo vệ dữ liệu.

---

### 6. Tầng 6: DB Layer (Database Engine Integrity Constraints)

- **Cách xử lý**:
    - Đây là chốt chặn vật lý cuối cùng nằm trực tiếp trong Schema của cơ sở dữ liệu.
    - **Unique Constraint trên tọa độ ghế (`uk_seat_event_position` trong bảng `seats`)**: Đảm bảo không bao giờ có 2 hàng ghế trùng tọa độ dòng/cột (`event_id, grid_row, grid_column`) cho cùng một sự kiện. Nếu một lỗi logic phần mềm nào đó cố ghi đè, DB sẽ từ chối ghi và trả lỗi `ConstraintViolationException`.
    - **Unique Constraint trên mã giao dịch (`uk_payment_transaction_ref` trong bảng `payments`)**: Đảm bảo một mã giao dịch chuyển khoản ngân hàng (`transaction_ref`) chỉ có thể áp dụng cho duy nhất một Payment record, triệt tiêu nguy cơ trùng lặp thanh toán.

---

### 7. Tầng 7: Redis Layer (Distributed Cache & Lock)

- **Cách xử lý**:
    - Khi khách hàng tạo phiên checkout, hệ thống thực hiện giữ ghế tạm thời trong vòng 10 phút trên Redis thông qua holdSeats:
        - Tạo key: `customer:seat-hold:<eventId>:<seatId>` lưu value là `paymentSessionId`.
        - Sử dụng lệnh `setIfAbsent` kèm TTL (Time-To-Live). Nếu ghi khóa thất bại (ghế đã bị người khác giữ), hệ thống sẽ báo lỗi ngay lập tức.
    - **Lợi ích**: Check nhanh trên bộ nhớ RAM của Redis trước giúp ngăn chặn hàng ngàn request tranh chấp chạm tới Database PostgreSQL, giúp tăng khả năng chịu tải (throughput) của hệ thống lên gấp nhiều lần.

---

### 8. Tầng 8: Queue / Event Layer (Kafka & Auto Load-Shedding)

- **Cách xử lý**:
    - Sau khi thanh toán thành công, hệ thống cần thực hiện hành động gửi email vé (Ticket Mail). Quá trình gửi email qua SMTP bên thứ ba rất chậm (thường mất 2-5 giây/email). Nếu làm đồng bộ, thread pool của server sẽ nhanh chóng bị nghẽn (Thread Exhaustion) khi có lượng mua lớn.
    - Hệ thống thiết lập class TicketDeliveryService với cơ chế **Auto Load-Shedding (Tự động giảm tải)**:
        - **Khi tải bình thường**: Email được gửi trực tiếp (`sendDirect`).
        - **Khi tải cao** (Số lượng request active vượt ngưỡng an toàn): Hệ thống tự động chuyển sang chế độ bất đồng bộ, đẩy sự kiện `BookingPaidEvent` vào **Kafka Queue** thông qua `bookingEventPublisher.publishBookingPaid()`.
        - Một Consumer độc lập BookingPaidConsumer sẽ tiêu thụ sự kiện từ Kafka và thực hiện gửi email ở background.
    - **Lợi ích**: Bảo vệ luồng thanh toán chính luôn mượt mà và phản hồi khách hàng ngay lập tức mà không bị nghẽn bởi các tác vụ gửi mail bên ngoài.