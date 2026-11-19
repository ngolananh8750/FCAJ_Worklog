---
title: "Worklog Tuần 7"
weight: 7
date: "` Sys.Date()`"
chapter: false
pre : "<b> 1.7 </b>"
---


### Mục tiêu tuần 7:

* Quản lý phân quyền máy chủ với IAM Role (Instance Profile).
* Nâng cao bảo mật bằng cách truy cập EC2 qua AWS SSM Session Manager không cần mở Port 22.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| :--- | :--- | :--- | :--- | :--- |
| 2 | Tạo IAM Role với quyền `AmazonSSMManagedInstanceCore` và gán vào EC2 Instance Profile | | | |
| 4 | Kiểm thử kết nối quản trị EC2 qua SSM Session Manager và khóa hoàn toàn Port 22 trên Security Group | | | |

### Kết quả đạt được tuần 7:

* Khóa hoàn toàn cổng SSH (Port 22) ra công khai, triệt tiêu nguy cơ bị tấn công Brute-force.
* Quản trị máy chủ an toàn trực tiếp từ giao diện AWS Console hoặc AWS CLI qua kênh kết nối bảo mật của AWS.