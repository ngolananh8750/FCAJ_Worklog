---
title: "Tạo Dashboard và gửi cảnh báo qua SNS"
weight: 4
chapter: false
pre: "<b>5.6.4. </b>"
---

1. Dùng `cloudwatch` chuyển sang `WorkshopMonitoringRole` trong Region Singapore. Mở **CloudWatch → Dashboards → Create dashboard**, đặt tên `workshop-web-01`.
2. Thêm widget **Line** cho `AWS/EC2 → CPUUtilization` với Instance ID `i-0c233adcff2a5a919`, thống kê Average, chu kỳ 5 phút. Thêm widget **Alarm status** cho `workshop-web-01-cpu-high` rồi lưu.
3. Trong **SNS → Topics**, tạo topic **Standard** `workshop-alerts` nếu chưa có. Tạo subscription **Email** đến hộp thư demo và xác nhận qua email. Đối chiếu trạng thái Confirmed.
4. Mở **CloudWatch → All alarms → workshop-web-01-cpu-high → Edit**. Thêm notification khi **In alarm** tới topic trên và lưu. Giữ điều kiện Average > 80%, 5 phút, 2/2. Nếu topic đã có policy tùy chỉnh, kiểm tra quyền CloudWatch publish.
5. Thực hiện bài thử tải có giới hạn ở 5.6.2. Đối chiếu timestamp trên dashboard, history và email. Nếu alarm đã ở ALARM trước khi gắn SNS, chờ về OK rồi thử lại để có chuyển trạng thái mới.



[CloudWatch và SNS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Notify_Users_Alarm_Changes.html)
