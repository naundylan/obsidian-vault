- VPC: dịch vụ cho phép tạo một hệ mạng riêng, mục đích của dịch vụ này là để phân lớp các IP public và các IP service nội bộ, có nghĩa là các dịch vụ như DB, Redis, Kafka... sẽ được đưa vào bên trong, client k thể truy cập những service này, thay vào đó phải đặt 1 "ông bảo vệ" đứng giữa để kiểm xoát
- Dịch vụ này sẽ được free trên AWS

---
- EC2: dịch vụ cung cấp máy ảo (có thể như VPS) nhưng cần cấu hình nhiều thứ hơn chứ k phải đơn giản như VPS
- Có bản free tier trong 12 tháng

---
- RDS: dịch vụ cung cấp nơi deploy DB (k rõ có ngon hơn superbase k, hoặc là ngon hơn render k), nó có các chức năng backup, restore, tự động nhiều thứ
- Maybe là có free tier