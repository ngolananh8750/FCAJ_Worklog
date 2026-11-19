---
title: "Workshop"
date: 2026-10-26
weight: 5
chapter: false
pre: "<b>5. </b>"
---

## Tổng quan

Workshop **Thiết kế và triển khai hệ thống Web Server bảo mật trên AWS sử dụng VPC, EC2, S3 và CloudWatch** giúp nhóm thực hành triển khai một trang web đơn giản, kiểm soát truy cập mạng, bảo vệ dữ liệu và theo dõi hoạt động của máy chủ.

Các dịch vụ chính được sử dụng gồm:

| Dịch vụ | Vai trò trong workshop |
| --- | --- |
| **Amazon VPC** | Tạo môi trường mạng cho máy chủ; sử dụng Security Group và Network ACL để kiểm soát lưu lượng truy cập. |
| **Amazon EC2** | Chạy ứng dụng web, xử lý đăng nhập và các yêu cầu tải lên, xem danh sách, tải xuống tài liệu. |
| **Amazon S3** | Lưu tài liệu trong bucket riêng tư; thực hành phân quyền, mã hóa và bảo vệ dữ liệu. |
| **Amazon CloudWatch** | Theo dõi chỉ số hoạt động của EC2, thiết lập cảnh báo cơ bản và thu thập nhật ký ứng dụng khi được cấu hình. |

**Bài toán thực tế: Xây dựng website lưu trữ và chia sẻ tài liệu nội bộ đơn giản trên AWS.**

Nhóm thực tập cần một website nhỏ để lưu và tải xuống tài liệu học tập mẫu. Website gồm trang đăng nhập và trang quản lý tài liệu. Sau khi đăng nhập, người dùng có thể tải tài liệu lên, xem danh sách tài liệu và tải tài liệu về máy.

Để dễ triển khai, hệ thống chỉ sử dụng **một tài khoản dùng chung được tạo sẵn cho bài thực hành**, không triển khai đăng ký, quên mật khẩu, phân quyền nhiều người dùng hay cơ sở dữ liệu riêng. Mật khẩu được lưu dưới dạng băm; thông tin bí mật không được đưa vào mã nguồn công khai. Chỉ sử dụng tài liệu mẫu, không chứa dữ liệu cá nhân.

Nhóm triển khai một EC2 trong Public Subnet của VPC để chạy ứng dụng web và xử lý đăng nhập. Tài liệu được lưu trong S3 bucket riêng tư, bật Block Public Access và mã hóa dữ liệu. EC2 sử dụng IAM Role với quyền tối thiểu để liệt kê, tải lên và đọc tài liệu qua S3 Gateway VPC Endpoint. Trình duyệt gửi yêu cầu đến ứng dụng trên EC2; ứng dụng kiểm tra phiên đăng nhập trước mỗi thao tác và truyền tài liệu giữa người dùng với S3. CloudWatch theo dõi CPU của EC2, thử nghiệm cảnh báo và tiếp nhận nhật ký ứng dụng sau khi cấu hình thu thập log.

**Luồng sử dụng:** Đăng nhập → Mở trang tài liệu → Tải lên hoặc chọn tài liệu để tải xuống → Đăng xuất.

Yêu cầu hoàn thành bài toán:

- Truy cập website qua HTTPS, đăng nhập bằng tài khoản được tạo sẵn và đăng xuất được. Với chứng chỉ tự ký trong môi trường thực hành, cần nêu rõ giới hạn tin cậy của chứng chỉ.
- Người chưa đăng nhập không được xem danh sách, tải lên hoặc tải xuống tài liệu, kể cả khi truy cập trực tiếp đường dẫn chức năng.
- Tải một tài liệu mẫu lên website, kiểm tra tệp được lưu trong S3 và tải lại thành công sau khi đăng nhập.
- Giới hạn loại tệp và dung lượng tải lên ở mức nhỏ, chẳng hạn PDF/TXT tối đa 5 MB; xử lý tên tệp an toàn.
- Cấu hình Security Group và Network ACL phù hợp; chỉ cho phép SSH từ IP quản trị được chỉ định. Cổng HTTP chỉ dùng để chuyển hướng sang HTTPS nếu có.
- Giữ S3 bucket riêng tư; EC2 chỉ được cấp quyền trên bucket hoặc thư mục tài liệu cần dùng, còn truy cập công khai bị từ chối.
- Xem được chỉ số CPU, kiểm thử cảnh báo và kiểm tra nhật ký ứng dụng trên CloudWatch; không ghi mật khẩu hoặc mã phiên đăng nhập vào log.
- Dùng một EC2 cho bài thực hành, giới hạn dung lượng tài liệu và thời gian lưu log; dọn dẹp tài nguyên sau khi hoàn thành để hạn chế chi phí.

Mỗi thành viên phụ trách một phần: **Thành viên 1 — VPC và Firewall; Thành viên 2 — EC2 và Web Server; Thành viên 3 — S3 và bảo mật dữ liệu; Thành viên 4 — CloudWatch và giám sát.** Cả nhóm sử dụng chung bài toán và phối hợp kiểm thử trên cùng hệ thống.

## Nội dung

- [5.1. Giới thiệu](./5.1-introduction/)
- [5.2. Các bước chuẩn bị](./5.2-preparation/)
  - [5.2.1. Chuẩn bị tài khoản và quyền truy cập AWS](./5.2-preparation/5.2.1-account-and-permissions/)
  - [5.2.2. Thống nhất Region, tên tài nguyên và nội dung trang web](./5.2-preparation/5.2.2-project-settings/)
- [5.3. Thiết lập mạng và cấu hình Firewall](./5.3-network-and-firewall/)
  - [5.3.1. Tạo VPC và Public Subnet](./5.3-network-and-firewall/5.3.1-vpc-and-subnet/)
  - [5.3.2. Tạo Internet Gateway và Route Table](./5.3-network-and-firewall/5.3.2-gateway-and-route-table/)
  - [5.3.3. Cấu hình Security Group](./5.3-network-and-firewall/5.3.3-security-group/)
  - [5.3.4. Cấu hình Network ACL](./5.3-network-and-firewall/5.3.4-network-acl/)
  - [5.3.5. Tạo S3 Gateway VPC Endpoint](./5.3-network-and-firewall/5.3.5-s3-endpoint/)
  - [5.3.6. Flow Logs và kiểm thử cổng 80/81](./5.3-network-and-firewall/5.3.6-flow-logs/)
- [5.4. Triển khai Web Server trên EC2](./5.4-ec2-web-server/)
  - [5.4.1. Khởi tạo EC2 và kết nối SSH](./5.4-ec2-web-server/5.4.1-launch-and-connect/)
  - [5.4.2. Cài đặt máy chủ web](./5.4-ec2-web-server/5.4.2-install-web-server/)
  - [5.4.3. Triển khai đăng nhập và chức năng tải lên, tải xuống tài liệu](./5.4-ec2-web-server/5.4.3-document-web-app/)
  - [5.4.4. Cấu hình HTTPS và kiểm tra phiên đăng nhập](./5.4-ec2-web-server/5.4.4-https-and-session/)
- [5.5. Lưu trữ và bảo mật dữ liệu trên S3](./5.5-s3-data-security/)
  - [5.5.1. Tạo bucket riêng tư và kiểm tra mã hóa](./5.5-s3-data-security/5.5.1-private-bucket/)
  - [5.5.2. Chuẩn bị tài liệu mẫu và kiểm tra lưu trữ trên S3](./5.5-s3-data-security/5.5.2-upload-documents/)
  - [5.5.3. Cấp quyền IAM Role và kiểm tra EC2 đọc, ghi tài liệu trên S3](./5.5-s3-data-security/5.5.3-ec2-access/)
  - [5.5.4. Versioning, Lifecycle và phục hồi](./5.5-s3-data-security/5.5.4-versioning-lifecycle-recovery/)
- [5.6. Giám sát và cảnh báo với CloudWatch](./5.6-cloudwatch/)
  - [5.6.1. Theo dõi chỉ số CPU của EC2](./5.6-cloudwatch/5.6.1-cpu-metrics/)
  - [5.6.2. Tạo và kiểm thử cảnh báo CPU](./5.6-cloudwatch/5.6.2-cpu-alarm/)
  - [5.6.3. Thu thập và kiểm tra nhật ký ứng dụng](./5.6-cloudwatch/5.6.3-application-logs/)
  - [5.6.4. Dashboard và thông báo SNS](./5.6-cloudwatch/5.6.4-dashboard-and-sns/)
- [5.7. Kiểm thử hệ thống](./5.7-testing/)
  - [5.7.1. Kiểm tra đăng nhập, tải lên, tải xuống và Firewall](./5.7-testing/5.7.1-website-and-firewall/)
  - [5.7.2. Kiểm tra quyền truy cập S3 và kết quả giám sát](./5.7-testing/5.7.2-s3-and-monitoring/)
- [5.8. Dọn dẹp tài nguyên](./5.8-cleanup/)
