---
title: "Chuẩn bị tài khoản và quyền truy cập AWS"
weight: 1
chapter: false
pre: "<b>5.2.1. </b>"
---

Chuẩn bị một tài khoản AWS cho workshop. Quản trị viên tạo bốn IAM user dưới đây và cho mỗi user chuyển sang đúng role được phân công. Các thành viên dùng MFA khi đăng nhập AWS Console.

| IAM user | IAM Role | Quyền cần cấp trong phạm vi lab |
| --- | --- | --- |
| `vpc` | `WorkshopNetworkRole` | Tạo và quản lý VPC, subnet, route table, Internet Gateway, Security Group, Network ACL, S3 Gateway Endpoint và VPC Flow Logs. |
| `ec2` | `WorkshopComputeRole` | Tạo và quản lý EC2, EBS và key pair của lab. Xem và chọn mạng có sẵn. Gắn instance profile chứa role ứng dụng đã duyệt. |
| `s3` | `WorkshopStorageRole` | Tạo và quản lý bucket riêng tư của lab, bucket policy, mã hóa, Versioning, Lifecycle và dữ liệu mẫu. |
| `cloudwatch` | `WorkshopMonitoringRole` | Xem chỉ số EC2. Quản lý log group, thời hạn lưu log, dashboard, alarm và SNS topic của lab. |

**Role cho dịch vụ AWS**

| IAM Role | Quyền cần cấp |
| --- | --- |
| `WorkshopEC2AppRole` | EC2 dùng role này để liệt kê, đọc và tải tài liệu lên S3 trong phạm vi `portal/`, đồng thời gửi log ứng dụng vào CloudWatch Logs. |
| `WorkshopFlowLogsDeliveryRole` | VPC Flow Logs dùng role này để ghi log vào log group của workshop. |

Chỉ cấp quyền theo phần việc của từng role. Giới hạn `iam:PassRole` của role EC2 và role mạng vào role dịch vụ tương ứng. Giữ bucket S3 ở chế độ riêng tư và dùng IAM Role thay cho Access Key lưu trên EC2.

Tiếp tục sang [5.2.2. Thống nhất Region, quy ước đặt tên và nội dung demo](../5.2.2-project-settings/).
