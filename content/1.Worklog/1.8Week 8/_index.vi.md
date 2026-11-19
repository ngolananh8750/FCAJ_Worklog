---
title: "Worklog Tuần 8"
weight: 8
date: "` Sys.Date()`"
chapter: false
pre : "<b> 1.8 </b>"
---


### Mục tiêu tuần 8:

* Giám sát hiệu năng hệ thống và tập trung log dịch vụ bằng AWS CloudWatch.
* Thiết lập cảnh báo ngân sách tự động với AWS Budgets để kiểm soát chi phí Free Tier.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| :--- | :--- | :--- | :--- | :--- |
| 2 | Nghiên cứu dịch vụ giám sát AWS CloudWatch (Metrics, Logs, Alarms) | | | |
| 3 | Cài đặt CloudWatch Agent trên EC2 để đẩy Log Nginx và chỉ số RAM/Disk về CloudWatch | | | |
| 5 | Khởi tạo AWS Budget cài đặt ngưỡng $1.00 USD và gửi Email cảnh báo khi chi phí đạt $0.80 USD | | | |

### Kết quả đạt được tuần 8:

* Thu thập đầy đủ log và chỉ số vận hành của Web Server về dashboard tập trung.
* Đảm bảo an toàn tài chính, tránh phát sinh chi phí ngoài ý muốn trên tài khoản AWS.