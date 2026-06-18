Tại sao nhìn chung Kafka & RabbitMQ lại trông giống nhau những lại được sử dụng ở các usecase hoàn toàn khác?

RabbitMQ:
- Thứ nhất là RMQ có cơ chế tự động xóa các message khi có các tín hiệu xác nhận đã xử lý tin nhắn thành công (đây là cơ chế tự nhiên chứ kp là xử lý logic của dev)
- Trong RMQ các messages nếu muốn được sử dụng bởi nhiều services khác nhau ví dụ như Email, phân tích, thống kê, ... Thì phải được nhân bản các tin nhắn đó ra với số lượng của services và được nhét vào các hàng đợi riêng biệt, khi đọc xong sẽ tự xóa
Kafka:
- Kafka thì khác, nó tạo ra không gian riêng trên disk, nó chỉ có nhiệm vụ là có message nào là sẽ ghi tuần tự liên tục ở trong không gian đó (không phải random), ghi nối tiếp nhau
- Các service khi cùng nhảy vào Kafka để đọc dữ liệu thì sẽ có các offset khác nhau, vậy nên k cần phải copy ra nhiều bản như RMQ (các offset này đã được kafka và spring boot tự động quản lý)