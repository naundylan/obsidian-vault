Ví dụ bài toán ở đây là bài toán generate ghế từ 1 sơ đồ layout ghế có sẵn (thường được dùng cho các hệ thống booking vé xem phim, concert,...)

Các đối tượng liên quan đến bài toán này là:
- Event: đại diện cho sự kiện
- Seat: đại diện cho ghế
- TicketClass: đại diện cho loại ghế (bên trong sơ đồ ghế sẽ có phân loại ghế)
- Layout: Sơ đồ ghế

Mối quan hệ giữa các Entity này là:
- Event phải được tạo sẵn từ trước, tiếp theo là Layout
- Khi mà Layout được tạo ra thì được yêu cầu tạo TicketClass (Lưu ý TicketClass sẽ k được gắn chung với Layout thật trong DB)
- Sau đó, sau khi có ticketclass thì mới bắt đầu cho vẽ ghế dựa vào ticketclass đó
- Những ghế ở đây đều không được lưu lại trừ phi người dùng bấm Save, lúc đó mới bắt đầu chạy script để lưu và tạo mới list seat, còn trước đó mọi thứ chỉ dựa vào các fields trung gian mà mapping với nhau mà thôi
- Lấy ví dụ như ta có một vấn đề, nếu trong bảng TicketClass có class là VIP, id là 123, giá là 1.500.000, nếu ta gán cho Layout (1 thứ sẽ được dùng lại khá nhiều), thì ở 2 layout khác nhau (có ticketclass giống nhau, mà sự kiện có tầm cỡ khác nhau) thì có thể ở 2 sự kiện đó giá vé đều khác, ví dụ là 100.000 và 1.500.000
- Vậy nên nếu lưu thẳng thì nó bị dính cứng với 1 Event duy nhất
- Dựa vào đó thì chúng ta có giải phải là:
	- Layout dùng TicketClassKey
	- Event dùng TicketClassId
	  --> Khi apply layout vào event thì sẽ có cơ chế mapping key sang id thật

Đặc biệt là với bài toán auto generate dựa trên một template tự chọn:
- Ví dụ là bạn muốn tạo một sơ đồ ghế ngồi từ sân vận động 5000 người nhưng vẽ tay quá lâu
- Vậy thì trong ngoài entity của layout bạn nên một số fields cần thiết cho việc tính toán như sau

```
export interface LayoutData {

  workspaceRows: number

  workspaceCols: number

  templateType?: LayoutTemplateType | null

  usedBounds?: UsedBounds | null

  cells: LayoutCell[]

  decorations?: LayoutDecoration[]

}
```

- Giải thích:
	- Lúc vẽ thì sẽ dưới dạng lưới grid 50x50 (mặc định nên cần workspaceRows và workspaceCols) để tính toán
	- templateType là để xem thử bản layout hiện tại có dùng template nào hay k
	- UsedBounds là số lượng hàng cột, ghế thực tế đã sử dụng
	- Cells chính là mảng các ghế kèm thông tin tọa độ + key của ticketClass
	- Decoration là mảng các phần tử trang trí như stage hoặc screen