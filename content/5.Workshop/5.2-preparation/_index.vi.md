---
title: "Các bước chuẩn bị"
date: 2026-10-27
weight: 2
chapter: false
pre: "<b>5.2. </b>"
---

## Tổng quan

Trước khi triển khai hệ thống, nhóm cần chuẩn bị tài khoản và quyền truy cập AWS, thống nhất Region, cách đặt tên tài nguyên và nội dung website. Việc chuẩn bị giúp bốn thành viên thao tác trên cùng một môi trường, phối hợp triển khai và tránh tạo tài nguyên trùng lặp.

Workshop sử dụng bài toán **website đăng nhập để tải lên, lưu trữ và tải xuống tài liệu**. Hệ thống gồm một EC2 chạy ứng dụng web trong Public Subnet, một S3 bucket riêng tư để lưu tài liệu và CloudWatch để giám sát. VPC, Security Group và Network ACL đảm nhận phần mạng và kiểm soát lưu lượng.

## Yêu cầu chuẩn bị

### Tài khoản và quyền truy cập AWS

- Có tài khoản AWS đang hoạt động và truy cập được AWS Management Console.
- Mỗi thành viên sử dụng danh tính IAM riêng để thực hiện phần việc được phân công; không chia sẻ tài khoản root cho cả nhóm.
- Người quản trị chuẩn bị quyền phù hợp cho từng thành viên, bao gồm quyền tạo và cấu hình tài nguyên thuộc phần việc được giao, không chỉ quyền xem.
- Chuẩn bị việc cấp IAM Role cho EC2 để ứng dụng truy cập S3 và gửi nhật ký đến CloudWatch theo phạm vi cần thiết. IAM Role của EC2 khác với quyền đăng nhập Console của các thành viên.
- Thống nhất người quản lý các quyền IAM dùng chung và xử lý yêu cầu bổ sung quyền khi phối hợp giữa các phần.

| Thành viên | Phạm vi công việc cần chuẩn bị quyền |
| --- | --- |
| **Thành viên 1 — VPC và Firewall** | Tạo, cấu hình và dọn dẹp VPC, subnet, Internet Gateway, Route Table, Security Group, Network ACL và S3 Gateway VPC Endpoint trong phạm vi workshop. |
| **Thành viên 2 — EC2 và Web Server** | Khởi tạo, quản lý EC2; sử dụng các tài nguyên mạng được bàn giao; gắn IAM Role đã được phê duyệt; triển khai và kiểm thử ứng dụng web. |
| **Thành viên 3 — S3 và bảo mật dữ liệu** | Tạo và cấu hình bucket, Block Public Access, mã hóa, chính sách truy cập và tài liệu mẫu trong phạm vi workshop. |
| **Thành viên 4 — CloudWatch và giám sát** | Xem chỉ số EC2; tạo, cấu hình và dọn dẹp cảnh báo, log group cùng các thiết lập giám sát thuộc workshop. |

Chi tiết chuẩn bị tài khoản và quyền truy cập được trình bày tại [5.2.1. Chuẩn bị tài khoản và quyền truy cập AWS](./5.2.1-account-and-permissions/).

### Máy tính và công cụ thực hành

- Máy tính có kết nối Internet và trình duyệt để thao tác trên AWS Management Console.
- Công cụ SSH như OpenSSH hoặc Bitvise SSH Client để kết nối đến EC2.
- Trình soạn thảo như Visual Studio Code để chỉnh sửa mã nguồn và tài liệu Markdown.
- Công cụ chụp màn hình để lưu bằng chứng cấu hình và kết quả kiểm thử.
- Một tệp PDF hoặc TXT mẫu có dung lượng không quá 5 MB, không chứa dữ liệu cá nhân.

Các bước tạo và cấu hình tài nguyên AWS được thực hiện trên Console. Việc cài đặt máy chủ web và triển khai ứng dụng trên EC2 được thực hiện qua kết nối SSH theo hướng dẫn ở phần 5.4.

### Thông tin dùng chung của nhóm

Trước khi tạo tài nguyên, cả nhóm cần thống nhất:

- **AWS Region:** sử dụng cùng một Region cho các tài nguyên của hệ thống; S3 bucket và EC2 đặt trong cùng Region để phù hợp với kết nối qua S3 Gateway VPC Endpoint.
- **Tên tài nguyên:** thống nhất tiền tố và cách đặt tên để nhận biết tài nguyên thuộc workshop.
- **Thông số mạng:** thống nhất dải địa chỉ VPC, subnet và địa chỉ IP quản trị được phép SSH.
- **Thông tin bàn giao:** ghi lại các ID cần dùng chung như VPC ID, Subnet ID, Security Group ID; bổ sung tên bucket, IAM Role và EC2 ID khi các tài nguyên được tạo ở những phần sau.
- **Website:** gồm trang đăng nhập và trang tài liệu, sử dụng một tài khoản thực hành được tạo sẵn; hỗ trợ xem danh sách, tải lên và tải xuống tài liệu sau khi đăng nhập.
- **Bảo mật:** lưu mật khẩu dưới dạng băm; không đưa mật khẩu, khóa bí mật hoặc khóa SSH lên kho mã nguồn công khai; giữ S3 bucket riêng tư.
- **HTTPS:** thống nhất phương án chứng chỉ cho bài thực hành. Nếu sử dụng chứng chỉ tự ký, ghi rõ giới hạn tin cậy và cảnh báo của trình duyệt.
- **Chi phí:** kiểm tra điều kiện ưu đãi thực tế của tài khoản, giới hạn thời gian chạy EC2, dung lượng tài liệu và thời gian lưu log; không mặc định toàn bộ workshop được miễn phí.

Chi tiết các thông số và quy ước được trình bày tại [5.2.2. Thống nhất Region, quy ước đặt tên và nội dung demo](./5.2.2-project-settings/).

## Kết quả cần đạt

Sau khi hoàn thành phần chuẩn bị:

- Cả bốn thành viên đăng nhập được vào AWS Management Console bằng danh tính của mình.
- Quyền truy cập đã được chuẩn bị theo nhiệm vụ; nhóm xác định được người hỗ trợ khi gặp lỗi thiếu quyền.
- Cả nhóm thống nhất Region, quy ước đặt tên, thông số mạng và cách bàn giao thông tin tài nguyên.
- Máy tính đã có công cụ SSH, trình soạn thảo và tệp tài liệu mẫu phục vụ kiểm thử.
- Phạm vi website, yêu cầu bảo mật và cách kiểm soát chi phí đã được thống nhất.



## Nội dung

1. [5.2.1. Chuẩn bị tài khoản và quyền truy cập AWS](./5.2.1-account-and-permissions/)
2. [5.2.2. Thống nhất Region, quy ước đặt tên và nội dung demo](./5.2.2-project-settings/)
