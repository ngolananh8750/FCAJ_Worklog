---
title: "Worklog Tuần 6"
weight: 6
date: "` Sys.Date()`"
chapter: false
pre : "<b> 1.6 </b>"
---


### Mục tiêu tuần 6:

* Triển khai mã hóa giao thức HTTPS cho Web Server bằng chứng chỉ SSL/TLS miễn phí.
* Tự động hóa quá trình chuyển hướng và gia hạn chứng chỉ bằng Certbot (Let's Encrypt).

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| :--- | :--- | :--- | :--- | :--- |
| 2 | Tìm hiểu giao thức SSL/TLS và cơ chế mã hóa lưu lượng truyền tải trên Web | | | |
| 3 | Cài đặt Certbot và plugin `python3-certbot-nginx` trên máy chủ EC2 | | | |
| 4 | Cấp phát chứng chỉ SSL/TLS miễn phí từ Let's Encrypt cho tên miền | | | |
| 5 | Cấu hình Nginx tự động chuyển hướng HTTP (301) sang HTTPS và đặt lịch tự động gia hạn (`certbot renew`) | | | |

### Kết quả đạt được tuần 6:

* Mã hóa $100\%$ lưu lượng truy cập Web Server qua giao thức HTTPS bảo mật.
* Tự động hóa quy trình gia hạn chứng chỉ SSL/TLS định kỳ.