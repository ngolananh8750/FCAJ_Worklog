---
title: "Workshop"
date: 2026-10-25
weight: 5
chapter: false
pre: "<b>5. </b>"
---

# 5. Xây dựng và bảo mật Web Server trên Amazon EC2

## Tổng quan

Trong hệ thống **Thiết kế và triển khai Web Server bảo mật trên AWS sử dụng VPC, EC2, S3 và CloudWatch**, Amazon EC2 đóng vai trò là máy chủ chính, chịu trách nhiệm vận hành website, xử lý các yêu cầu truy cập từ người dùng và kết nối với các dịch vụ AWS khác.

Workshop này tập trung vào việc **xây dựng, triển khai và bảo mật Web Server trên Amazon EC2**, bảo đảm hệ thống hoạt động ổn định, kiểm soát quyền truy cập và hỗ trợ sao lưu dữ liệu lên Amazon S3.

Máy chủ EC2 được triển khai trong Public Subnet thuộc Amazon VPC do thành viên 1 cấu hình, sử dụng Security Group và Network ACL để kiểm soát lưu lượng mạng.

Web Server sử dụng Nginx để cung cấp nội dung website thông qua HTTP và HTTPS. Đồng thời, hệ thống áp dụng các biện pháp bảo mật như giới hạn quyền quản trị SSH, sử dụng IAM Role, cập nhật hệ điều hành, mã hóa ổ đĩa và bảo vệ thông tin xác thực.

Workshop cũng thực hiện tích hợp Amazon S3 để hỗ trợ sao lưu dữ liệu website và phối hợp với Amazon CloudWatch trong việc giám sát hoạt động của EC2.

## Mục tiêu Workshop

Sau khi hoàn thành workshop, người thực hiện có thể:

* Khởi tạo và cấu hình Amazon EC2 trong môi trường VPC đã được thiết lập.
* Cài đặt Nginx và triển khai website HTML/CSS cơ bản.
* Cấu hình truy cập Web Server thông qua HTTP và HTTPS.
* Áp dụng Security Group để kiểm soát các cổng 22, 80 và 443.
* Tăng cường bảo mật hệ điều hành và dịch vụ Web Server.
* Sử dụng SSH Key Pair và hạn chế quyền truy cập quản trị từ Internet.
* Cấu hình IAM Role cho EC2 theo nguyên tắc phân quyền tối thiểu.
* Sử dụng IAM Role thay cho việc lưu trữ AWS Access Key trực tiếp trên máy chủ.
* Thiết lập kết nối giữa EC2 và Amazon S3 phục vụ sao lưu dữ liệu website.
* Kiểm thử khả năng truy cập, bảo mật và hoạt động của Web Server.
* Phối hợp với Amazon CloudWatch để giám sát trạng thái máy chủ.
* Tối ưu tài nguyên sử dụng nhằm giảm chi phí thực hành trên AWS.

## Giới thiệu mô hình triển khai

Hệ thống được xây dựng trên nền tảng AWS Cloud với các thành phần chính:

| Thành phần              | Vai trò                                                          |
| ----------------------- | ---------------------------------------------------------------- |
| Amazon VPC              | Cung cấp môi trường mạng riêng cho hệ thống                      |
| Public Subnet           | Chứa máy chủ EC2 và cho phép kết nối Internet theo cấu hình mạng |
| Internet Gateway        | Kết nối VPC với Internet                                         |
| Security Group          | Kiểm soát lưu lượng truy cập đến và đi từ EC2                    |
| Network ACL             | Bổ sung lớp kiểm soát lưu lượng tại cấp subnet                   |
| Amazon EC2              | Vận hành Web Server                                              |
| Nginx                   | Cung cấp nội dung website qua HTTP/HTTPS                         |
| Amazon EBS              | Lưu trữ hệ điều hành và dữ liệu máy chủ                          |
| AWS IAM                 | Quản lý quyền truy cập của người thực hành và máy chủ EC2        |
| Amazon S3               | Lưu trữ bản sao lưu dữ liệu website                              |
| S3 Gateway VPC Endpoint | Hỗ trợ truy cập S3 từ VPC thông qua tuyến mạng riêng             |
| Amazon CloudWatch       | Giám sát hoạt động và các chỉ số của EC2                         |

**Luồng hoạt động chính:**

1. Người dùng gửi yêu cầu HTTP hoặc HTTPS đến địa chỉ công khai của Web Server.
2. Lưu lượng đi qua Internet Gateway và được kiểm soát bởi Network ACL và Security Group.
3. EC2 tiếp nhận yêu cầu và Nginx cung cấp nội dung website.
4. EC2 sử dụng IAM Role để thực hiện các thao tác sao lưu dữ liệu được cho phép trên Amazon S3.
5. CloudWatch thu thập các chỉ số giám sát EC2 để hỗ trợ theo dõi trạng thái và hiệu năng.

### Cấu hình triển khai tối ưu chi phí

Workshop sử dụng một máy chủ EC2 duy nhất và tận dụng hạ tầng mạng dùng chung của dự án.

| Tài nguyên        | Cấu hình dự kiến                                                  |
| ----------------- | ----------------------------------------------------------------- |
| AWS Region        | Asia Pacific (Singapore) – ap-southeast-1                         |
| EC2 Instance Type | t4g.micro                                                         |
| Hệ điều hành      | Amazon Linux 2023 (ARM64)                                         |
| Số lượng EC2      | 1                                                                 |
| EBS Volume        | 8 GiB gp3, bật mã hóa                                             |
| Web Server        | Nginx                                                             |
| Subnet            | Public Subnet                                                     |
| IAM Instance Role | SecureWebServerEC2Role                                            |
| EC2 Metadata      | IMDSv2 Required                                                   |
| Monitoring        | CloudWatch Basic Monitoring                                       |
| Sao lưu           | Amazon S3                                                         |
| Quản trị          | SSH Key Pair, giới hạn IP hoặc Session Manager khi được cấp quyền |

Mô hình không sử dụng NAT Gateway, Application Load Balancer hoặc AWS WAF nhằm giảm chi phí trong phạm vi bài thực hành.

EC2 chỉ được bật khi cần triển khai, kiểm thử hoặc trình diễn. Các tài nguyên được tái sử dụng giữa các thành viên thay vì khởi tạo hệ thống riêng biệt.

## Chuẩn bị môi trường thực hành

Workshop được thực hiện trên AWS Management Console bằng danh tính IAM hiện hành đã được người quản trị cấp quyền phù hợp.

**Yêu cầu về IAM:**

* Người thực hành đăng nhập bằng IAM User hoặc danh tính IAM được cấp cho thành viên 2.
* Có quyền sử dụng IAM Role `SecureWebEC2Role` hoặc bộ quyền tương đương để triển khai và quản lý EC2 trong phạm vi dự án.
* Có quyền gắn IAM Instance Role `SecureWebServerEC2Role` vào EC2 theo phạm vi được cho phép.
* Không sử dụng Root User để thực hiện các thao tác quản trị hằng ngày.
* Áp dụng nguyên tắc Least Privilege và bật MFA cho tài khoản thực hành.

**Yêu cầu về hạ tầng:**

* VPC, Public Subnet, Internet Gateway, Route Table, Security Group và Network ACL do thành viên 1 chuẩn bị.
* IAM Instance Role và quyền truy cập S3 được thống nhất với thành viên 3.
* Các cấu hình giám sát được phối hợp với thành viên 4.
* Máy tính cá nhân có trình duyệt, công cụ SSH và kết nối Internet.

## Nội dung Workshop

Workshop được chia thành các phần sau:

1. **[5.1. Giới thiệu mô hình triển khai](5.1/)**

   Tìm hiểu kiến trúc Web Server trên Amazon EC2, các thành phần dịch vụ AWS, luồng kết nối và cơ chế bảo mật trong hệ thống.

2. **[5.2. Chuẩn bị môi trường thực hành](5.2/)**

   Kiểm tra tài khoản IAM, quyền truy cập, IAM Role, các tài nguyên VPC đã được tạo và những điều kiện cần thiết trước khi triển khai.

3. **[5.3. Khởi tạo và cấu hình Amazon EC2](5.3/)**

   Tạo EC2 Instance, lựa chọn hệ điều hành, cấu hình EBS, Key Pair, Security Group, IAM Instance Role và phương thức truy cập máy chủ.

4. **[5.4. Cài đặt Nginx và triển khai Website](5.4/)**

   Cập nhật hệ điều hành, cài đặt Nginx, xây dựng website HTML/CSS và kiểm tra khả năng truy cập Web Server.

5. **[5.5. Cấu hình bảo mật Web Server](5.5/)**

   Tăng cường bảo mật hệ điều hành, quản trị SSH an toàn, giới hạn các cổng truy cập, cấu hình HTTPS khi có tên miền phù hợp và kiểm tra IMDSv2.

6. **[5.6. Tích hợp IAM Role và sao lưu dữ liệu lên Amazon S3](5.6/)**

   Kiểm tra quyền IAM Role của EC2, kết nối với S3 Bucket riêng tư, triển khai script sao lưu và cấu hình lịch sao lưu bằng cron hoặc systemd timer.

7. **[5.7. Kiểm thử hoạt động và bảo mật hệ thống](5.7/)**

   Kiểm thử HTTP/HTTPS, SSH, Security Group, quyền truy cập S3, chức năng sao lưu và phối hợp kiểm tra chỉ số giám sát trên CloudWatch.

8. **[5.8. Dọn dẹp tài nguyên](5.8/)**

   Dừng hoặc xóa tài nguyên EC2 không còn cần thiết, kiểm tra các tài nguyên phát sinh chi phí và bảo đảm không ảnh hưởng đến hạ tầng dùng chung của các thành viên khác.

## Kết quả mong đợi

Sau khi hoàn thành workshop, hệ thống cần đạt được các kết quả:

* Web Server được triển khai thành công trên Amazon EC2.
* Website HTML/CSS có thể truy cập thông qua trình duyệt.
* Nginx hoạt động ổn định và hỗ trợ cấu hình HTTPS.
* Quyền truy cập máy chủ được kiểm soát bằng Security Group và cơ chế quản trị an toàn.
* EC2 sử dụng IAM Role phù hợp và không lưu AWS Access Key tĩnh.
* Dữ liệu website có thể được sao lưu lên Amazon S3 và kiểm tra khả năng khôi phục.
* Các chỉ số hoạt động của EC2 có thể được theo dõi qua Amazon CloudWatch.
* Có đầy đủ ảnh chụp màn hình minh chứng, kết quả kiểm thử và tài liệu cấu hình để phục vụ báo cáo workshop.

Workshop góp phần xây dựng một hệ thống Web Server cơ bản, bảo mật, dễ quản lý và có chi phí phù hợp với môi trường thực hành trên AWS.
