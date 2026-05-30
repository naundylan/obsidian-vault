Transaction không được gọi là 1 phiên mà là 1 vùng làm việc mà trong đó có một chuỗi công việc được thực thi, nếu 1 trong số các công việc bị lỗi thì toàn bộ chuỗi đó sẽ bị hủy (rollback), còn nếu hoàn thành tất cả thì sẽ đóng và lưu (commit)

Transaction tuân thủ 4 nguyên tắc ACID:
- Atomicity: hoặc tất cả thành công, hoặc tất cả rollback
- Consistency: dữ liệu sau transaction phải hợp lệ
- Isolation: transaction này không được phá dữ liệu của transaction khác
- Durability: commit rồi thì dữ liệu được lưu bền vững

Cú pháp sử dụng trong Java ở folder Repository:

```
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select s from Seat s where s.id in :ids")
List<Seat> lockSeats(@Param("ids") List<UUID> ids);

```


Đối với bài toán Lock Transaction: có 2 dạng lock
- **Pessimistic Lock**: Khóa ngay khi đọc
	- Tôi đang đọc những ghế này để sửa. Transaction khác muốn sửa phải chờ.
	- Hợp với bài toán ghế, tồn kho, ví, thanh toán, vì không được phép bán trùng.
	- Ưu điểm:
		- chắc chắn hơn trong cạnh tranh cao
		- dễ hiểu
	- Nhược điểm:
		- có thể chờ lock
		- nếu nhiều lock có thể ảnh hưởng performance
- **Optimistic Lock**: Không khóa ngay mà dùng version để phát hiện xung đột khi save
	- Entity phải có @Version private int version;
	- Ưu điểm:
		- không khóa sớm
		- tốt khi ít xung đột
	- Nhược điểm:
		- khi xung đột thì fail ở cuối, phải retry hoặc báo lỗi


Cách giải quyết bài toán thực tế của dạng này:
- Tầng 1: UI layer:
	- disable button, loading, chặn double click
- Tầng 2: API/controller layer:
	- validate req
	- Dùng idempotency key cho các req quan trọng
	- tránh xử lý trùng lặp nếu req đến từ 1 nguồn reqId
- Tầng 3: Service layer:
	- dùng keyword transaction để gom nhiều thao tác db vào chung 1 chổ
- Tầng 4: Repository layer:
	- Khai báo Pessimistic lock
- Tầng 5: DB layer:
	- Constraint unique + foreign key
- Tầng 6: OL layer:
	- Entity phải có version
- Tầng 7: Redis layer:
	- Dùng Redis để giữ ghế 5 phút
	- Tận dụng cache của Redis để tăng độ trãi ngh
- Tầng 8: Queue/event layer:
	- Đẩy vào queue để xử lý tuần tự
