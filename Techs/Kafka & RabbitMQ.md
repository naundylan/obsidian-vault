Tại sao nhìn chung Kafka & RabbitMQ lại trông giống nhau những lại được sử dụng ở các usecase hoàn toàn khác?

RabbitMQ:
- Thứ nhất là RMQ có cơ chế tự động xóa các message khi có các tín hiệu xác nhận đã xử lý tin nhắn thành công (đây là cơ chế tự nhiên chứ kp là xử lý logic của dev)
- Trong RMQ các messages nếu muốn được sử dụng bởi nhiều services khác nhau thì phải 