Khái niệm về Luồng và Chạy ngầm:

Nhân CPU: mỗi thiết bị đều có các thông số cơ bản, chúng ta cần phải hiểu về cpu và số nhân của nó, Nhân cpu tương tự như 1 cánh tay của người, có thể làm 1 việc cùng 1 thời điểm, nhưng nếu là 2 nhân hoặc nhiều hơn thì có thể làm được n công việc cùng 1 thời điểm (paralelled)

Luồng:
- Luồng có thể hiểu là 1 công việc chạy từ đầu tới cuối, ví dụ 1 request của client khi truy cập vào server là 1 luồng, khi truy cập, Server Tomcat của chúng ta sẽ cử 1 luồng ra tiếp đón req này (mặc định có 200 luồng) và nó sẽ được giữ cho đến khi client rời khỏi hoặc kết thúc, vd: truy cập sự kiện -> mua vé -> thanh toán -> gửi mail....
- Mỗi 1 luồng đều có sử dụng các dữ liệu chung của toàn server, ngoài ra nó được cấp riêng space để làm việc, vì vậy mỗi luồng thường ăn mất 1MB RAM của máy chủ, nếu tạo ra quá nhiều luồng và vượt mức cho phép -> Crash server
- Luồng khác với connection của DB, luồng xuất hiện từ khi client truy cập server, còn connection db xuất hiện khi user yêu cầu data, và tất nhiên cũng bị hold cho đến khi kết thúc công việc đọc/ghi dữ liệu
- 