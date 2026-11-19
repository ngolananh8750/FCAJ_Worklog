---
title: "Worklog Tuần 3"
weight: 3
date: "` Sys.Date()`"
chapter: false
pre : "<b> 1.3 </b>"
---

### Mục tiêu tuần 3:

* Nghiên cứu và thiết lập hạ tầng mạng riêng ảo AWS VPC.
* Khởi tạo các phân vùng mạng Subnet, Internet Gateway và Route Table định tuyến lưu lượng.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| :--- | :--- | :--- | :--- | :--- |
| 2 | Tìm hiểu lý thuyết mạng AWS: VPC, CIDR Notation, Public & Private Subnets | | | |
| 3 | Thực hành khởi tạo VPC riêng biệt (`10.0.0.0/16`) và Public Subnet (`10.0.1.0/24`) | | | |
| 4 | Tạo Internet Gateway (IGW) và đính kèm vào VPC vừa khởi tạo | | | |
| 5 | Cấu hình Route Table định tuyến lưu lượng `0.0.0.0/0` ra Internet Gateway | | | |

### Kết quả đạt được tuần 3:

* Làm chủ kiến thức mạng căn bản trên đám mây AWS.
* Thiết lập hoàn chỉnh môi trường mạng riêng ảo VPC cô lập phục vụ triển khai Web Server.