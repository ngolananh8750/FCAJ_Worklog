---
title: "Worklog Tuần 4"
weight: 4
date: "` Sys.Date()`"
chapter: false
pre : "<b> 1.4 </b>"
---




### Mục tiêu tuần 4:

* Khởi tạo máy chủ EC2 trong hạ tầng VPC riêng.
* Cấu hình tường lửa ảo Security Group tuân thủ nguyên tắc phân quyền tối thiểu (Least Privilege).

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| :--- | :--- | :--- | :--- | :--- |
| 2 | Nghiên cứu cơ chế hoạt động của tường lửa ảo Security Group (Stateful Firewall) | | | |
| 3 | Tạo Security Group với quy tắc Inbound: mở Port 80, 443 và siết chặt Port 22 (SSH) chỉ cho địa chỉ `My IP` | | | |
| 4 | Launch máy chủ EC2 vào Public Subnet của VPC riêng và kiểm tra kết nối mạng | | | |

### Kết quả đạt được tuần 4:

* Triển khai thành công máy chủ EC2 trên hạ tầng VPC tự dựng.
* Thiết lập tường lửa bảo vệ máy chủ khỏi các cuộc tấn công dò quét cổng từ toàn mạng.