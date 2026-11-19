---
title: "Worklog Tuần 9"
weight: 9
date: "` Sys.Date()`"
chapter: false
pre : "<b> 1.9 </b>"
---



### Mục tiêu tuần 9:

* Tự động hóa quy trình triển khai mã nguồn trang web (CI/CD Pipeline).
* Xây dựng GitHub Actions Workflow tự động đẩy code lên Nginx Web Server.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| :--- | :--- | :--- | :--- | :--- |
| 2 | Nghiên cứu quy trình CI/CD và công cụ tự động hóa GitHub Actions | | | |
| 3 | Tạo GitHub Repository và cấu hình GitHub Secrets lưu trữ SSH Credentials/Host IP an toàn | | | |
| 4 | Viết tệp workflow `.github/workflows/deploy.yml` tự động rsync mã nguồn lên EC2 khi push code | | | |
| 5 | Kiểm thử luồng CI/CD: Push thay đổi mã nguồn mới và kiểm tra cập nhật trên Web Server | | | |

### Kết quả đạt được tuần 9:

* Loại bỏ hoàn toàn thao tác kéo code thủ công lên máy chủ.
* Tự động hóa $100\%$ quy trình đưa phiên bản trang web mới lên môi trường chạy thực tế.