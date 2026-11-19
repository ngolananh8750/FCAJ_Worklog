---
title: "Thống nhất Region, quy ước đặt tên và nội dung demo"
weight: 2
chapter: false
pre: "<b>5.2.2. </b>"
---

Chọn Region **Asia Pacific (Singapore)** cho workshop. Dùng các tên dưới đây khi tạo tài nguyên.

| Tài nguyên | Tên quy ước |
| --- | --- |
| VPC | `vpc-secure-web` |
| Public subnet | `subnet-web-public` |
| Internet Gateway | `igw-secure-web` |
| Route table | `rtb-web-public` |
| Security Group | `sg-secure-web` |
| Network ACL | `nacl-web-public` |
| S3 Gateway Endpoint | `s3-gateway-endpoint` |
| VPC Flow Log | `flow-web-public` |
| Máy chủ EC2 | `workshop-web-01` |
| Key pair | `workshop-web-key` |
| S3 bucket | `secureweb-docs-<hau-to-rieng>` |
| Log group mạng | `/workshop/vpc/flow-logs` |
| Log group ứng dụng | `/workshop/webserver` |
| CloudWatch Alarm | `workshop-web-01-cpu-high` |
| CloudWatch Dashboard | `workshop-web-01` |
| SNS topic | `workshop-alerts` |

Với S3, thay `<hau-to-rieng>` bằng hậu tố riêng viết thường, ví dụ `secureweb-docs-nhom07-a4k` nếu tên còn khả dụng.

**Tag dự án:** Gắn `Project=SecureWebWorkshop` khi tạo tài nguyên có hỗ trợ tag để dễ nhận diện tài nguyên của lab.

**Demo:** Chuẩn bị một tệp TXT hoặc PDF mẫu để thử đăng nhập, tải lên và tải xuống.

Tiếp tục sang [5.3. Thiết lập mạng và Firewall](../../5.3-network-and-firewall/).
