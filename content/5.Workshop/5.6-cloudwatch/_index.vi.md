---
title: "Giám sát và cảnh báo với CloudWatch"
date: 2026-10-31
weight: 6
chapter: false
pre: "<b>5.6. </b>"
---

## Tổng quan

Sau khi hoàn thành triển khai website và kiểm tra quyền truy cập S3, nhóm sử dụng **Amazon CloudWatch** để theo dõi hoạt động của Web Server, phát hiện CPU vượt ngưỡng và xem nhật ký ứng dụng khi xảy ra lỗi.

Trong phần này, **thành viên 4** phụ trách giám sát, phối hợp với thành viên 2 để cấu hình thu thập log trên EC2. Khi cần thay đổi quyền của **WorkshopEC2AppRole**, phối hợp với người quản trị IAM và giữ nguyên quyền S3 đã cấu hình ở phần 5.5.3.

Đây là trang tổng quan của phần 5.6. Các thao tác Console, lệnh cài đặt và kiểm thử chi tiết nằm trong ba mục con bên dưới.

## Mục tiêu

- Xem chỉ số CPU của đúng EC2 **workshop-web-01** tại **Singapore — ap-southeast-1**.
- Tạo cảnh báo CPU và kiểm tra sự thay đổi trạng thái dựa trên dữ liệu thực tế.
- Thu thập nhật ký ứng dụng vào CloudWatch Logs, đối chiếu với thao tác thử nghiệm trên website.
- Lưu bằng chứng giám sát để sử dụng trong phần kiểm thử hệ thống 5.7.

CloudWatch cung cấp khả năng quan sát; dịch vụ này không thay thế Security Group, Network ACL, cơ chế đăng nhập hoặc quyền truy cập S3. CPU thấp cũng không đủ để kết luận website đang hoạt động bình thường.

## Phạm vi và cấu hình thống nhất

| Thành phần | Cấu hình của workshop |
| --- | --- |
| Region | ap-southeast-1 |
| EC2 | workshop-web-01 — ghi lại Instance ID thực tế |
| Namespace / Metric | AWS/EC2 / CPUUtilization |
| Dimension | InstanceId của workshop-web-01 |
| Giám sát EC2 | Basic monitoring; CPU có dữ liệu theo chu kỳ 5 phút |
| Tên cảnh báo | workshop-web-01-cpu-high |
| Cấu hình cảnh báo dự kiến | Average, period 5 phút, CPU > 80%, 2/2 datapoints |
| Xử lý dữ liệu thiếu | missing; không coi thiếu dữ liệu là bằng chứng hệ thống khỏe |
| Log group | /workshop/webserver |
| Thời gian giữ log | 7 ngày cho bài lab |
| Công cụ gửi log | Amazon CloudWatch Agent trên EC2 |
| Thông báo email | Tùy chọn qua SNS topic workshop-alerts |

Basic monitoring đủ cho phần theo dõi CPU này và không cần cài CloudWatch Agent để có chỉ số CPUUtilization của EC2. Agent được dùng ở 5.6.3 để gửi log. Không bật Detailed monitoring hoặc thu thập thêm custom metrics khi chưa cần thiết.

Cấu hình 2/2 datapoints yêu cầu hai điểm dữ liệu vi phạm trong hai kỳ đánh giá. Với period 5 phút, việc kiểm thử cần dành thời gian cho dữ liệu được gửi và đánh giá; không mong đợi cảnh báo ngay khi bắt đầu tạo tải. Ngưỡng trên là lựa chọn cho bài lab, không phải mức chuẩn cho mọi ứng dụng.

## Chuẩn bị

1. Hoàn thành [5.5.3. Cấp quyền IAM Role và kiểm tra EC2 đọc, ghi tài liệu trên S3](../5.5-s3-data-security/5.5.3-ec2-access/).
2. Kiểm tra EC2 đang chạy, ghi lại Instance ID và chọn đúng Region trên Console.
3. Xác nhận website truy cập được qua HTTPS, đăng nhập và tải lên/tải xuống tài liệu mẫu thành công.
4. Chuẩn bị quyền xem metrics, tạo/sửa cảnh báo và xem logs cho thành viên 4. Người tạo log group cần quyền tạo nhóm và đặt retention; quyền chỉ đọc log không đủ để triển khai.
5. Khi cấu hình Agent, cấp quyền ghi log cho IAM Role của EC2 trong phạm vi log group của workshop. Quyền của IAM user thao tác Console và quyền của EC2 là hai phạm vi khác nhau; không lưu access key lên máy chủ.
6. Kiểm tra EC2 có DNS và kết nối outbound HTTPS đến dịch vụ CloudWatch Logs. S3 Gateway Endpoint chỉ phục vụ S3; log dùng đường Internet hiện có qua Internet Gateway và Public IPv4 của EC2.

Nếu tài khoản thiếu quyền, ghi lại action bị từ chối và nhờ quản trị bổ sung đúng phạm vi; không gắn AdministratorAccess chỉ để vượt lỗi.

## Nội dung thực hiện

### 5.6.1. Theo dõi chỉ số CPU của EC2

Mở CloudWatch Metrics, chọn CPUUtilization theo InstanceId, đặt Statistic là Average và Period là 5 phút. Quan sát dữ liệu trước khi tạo tải để có mốc so sánh. Kiểm tra Region, khoảng thời gian và Instance ID nếu đồ thị chưa có dữ liệu.

### 5.6.2. Tạo và kiểm thử cảnh báo CPU

Tạo **workshop-web-01-cpu-high**, cấu hình ngưỡng đã thống nhất và kiểm tra các trạng thái **OK**, **ALARM**, **INSUFFICIENT_DATA**. Thực hiện tải thử có giới hạn thời gian, theo dõi CPU và Alarm history, sau đó dừng tải và quan sát phục hồi.

EC2 t3.micro dùng CPU credits; ở chế độ Standard, thiếu credits có thể giới hạn hiệu năng và làm tải thử không đạt ngưỡng dự kiến. Không chạy tải vô hạn hoặc chuyển sang Unlimited chỉ để kích hoạt cảnh báo. Chi tiết kiểm thử và cách xử lý sẽ nằm ở 5.6.2.

Cảnh báo có thể được kiểm tra trực tiếp trên Console. Nếu cần email, cấu hình thêm SNS và xác nhận subscription. Trạng thái ALARM trên Console không đồng nghĩa email đã được gửi; phải kiểm tra riêng. Không cấu hình hành động tự dừng, terminate hoặc reboot EC2 cho bài lab này.

### 5.6.3. Thu thập và kiểm tra nhật ký ứng dụng

Chuẩn bị log group, retention, quyền EC2 và CloudWatch Agent. Ứng dụng hiện ghi log qua systemd journal, nên cần cấu hình đầu ra file hoặc cơ chế chuyển log sang file trước khi Agent đọc; chỉ cài Agent sẽ không tự thu thập nhật ký ứng dụng.

Sau đó thực hiện thao tác thử như đăng nhập thành công, đăng nhập sai hoặc tải tài liệu, đối chiếu các sự kiện ứng dụng thực sự ghi nhận với log trên CloudWatch. Chỉ kết luận thành công khi tìm thấy sự kiện mới khớp thời điểm kiểm thử.

Không ghi mật khẩu, password hash, cookie, session token, access key hoặc nội dung tài liệu vào log. Khi chuyển sang file, cần quyền truy cập phù hợp và xoay vòng log để tránh đầy đĩa EC2.



## Kiểm soát chi phí

Giữ Basic monitoring, chỉ tạo cảnh báo cần thiết, thu thập các file log phục vụ kiểm thử và đặt retention 7 ngày. CloudWatch Logs, alarms, SNS hoặc các tính năng bổ sung có thể phát sinh phí theo mức sử dụng và điều kiện tài khoản; không mặc định toàn bộ phần này miễn phí. Xóa tài nguyên giám sát không còn cần thiết trong phần 5.8 sau khi lưu bằng chứng.

## Kết quả cần đạt

- Có đồ thị CPU của đúng máy chủ.
- Có cảnh báo được kiểm thử và bằng chứng phục hồi sau khi dừng tải.
- Có log ứng dụng mới trên CloudWatch, retention đúng và không chứa bí mật.
- Ghi lại tên alarm, log group, Instance ID và thời gian thử nghiệm để bàn giao.

## Nội dung

- [5.6.1. Theo dõi chỉ số CPU của EC2](./5.6.1-cpu-metrics/)
- [5.6.2. Tạo và kiểm thử cảnh báo CPU](./5.6.2-cpu-alarm/)
- [5.6.3. Thu thập và kiểm tra nhật ký ứng dụng](./5.6.3-application-logs/)
- [5.6.4. Dashboard và thông báo SNS](./5.6.4-dashboard-and-sns/)

## Tài liệu tham khảo

- [EC2 basic và detailed monitoring](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/manage-detailed-monitoring.html)
- [CloudWatch alarm evaluation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-evaluation.html)
- [CloudWatch Agent configuration](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)
- [Log groups, streams và retention](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html)
