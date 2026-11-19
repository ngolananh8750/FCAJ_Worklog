---
title: "Worklog Tuần 10"
weight: 10
date: "` Sys.Date()`"
chapter: false
pre : "<b> 1.10 </b>"
---


### Mục tiêu tuần 10:

* Tìm hiểu mạng phân phối nội dung Amazon CloudFront (CDN) và tường lửa ứng dụng web AWS WAF.
* Tăng tốc độ tải trang và bổ sung lớp bảo mật tại hạ tầng biên (Edge Location).

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| :--- | :--- | :--- | :--- | :--- |
| 2 | Tạo CloudFront Distribution trỏ máy chủ Nginx EC2 làm Origin giúp tăng tốc độ phân phối nội dung | | | |
| 4 | Cấu hình AWS WAF đính kèm CloudFront với các bộ Managed Rules cơ bản bảo vệ Web Server khỏi lỗ hổng web | | | |

### Kết quả đạt được tuần 10:

* Tăng tốc độ phân phối nội dung tĩnh tới người dùng nhờ mạng lưới lưu trữ cache của CloudFront.
* Thiết lập thêm một lớp tường lửa bảo vệ Web Server ngay từ hạ tầng biên.