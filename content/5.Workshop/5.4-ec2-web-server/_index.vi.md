---
title: "Triển khai Web Server trên EC2"
date: 2026-10-30
weight: 4
chapter: false
pre: "<b>5.4. </b>"
---

**Thứ tự chạy:** 5.4.1–5.4.2 → 5.5.1–5.5.2 → 5.4.3 → 5.5.3 (bước 1–5) → 5.4.4 → 5.5.3 (bước 6) → 5.5.4. Dùng gói mã nguồn trong thư mục `demo/` của ZIP workshop.

## Tổng quan

Trong phần này, nhóm triển khai **Web Server trên Amazon EC2** để vận hành website đăng nhập, tải lên và tải xuống tài liệu. Máy chủ sử dụng hạ tầng mạng và Firewall đã chuẩn bị ở phần 5.3; tài liệu được lưu trong bucket S3 riêng tư.

**Thành viên 2** phụ trách tạo EC2, cài đặt máy chủ web, triển khai ứng dụng và cấu hình HTTPS. Thành viên 1 hỗ trợ mạng/Firewall, thành viên 3 phối hợp quyền truy cập S3, thành viên 4 tiếp nhận thông tin nhật ký để cấu hình CloudWatch ở phần 5.6.

Trang này giới thiệu phạm vi và thứ tự thực hiện. Các thao tác chi tiết nằm trong bốn mục con bên dưới.

## Mục tiêu

- Khởi tạo một EC2 trong subnet-web-public và kết nối quản trị bằng SSH từ nguồn được phép.
- Cài đặt máy chủ web và môi trường chạy ứng dụng.
- Triển khai đăng nhập, đăng xuất, danh sách tài liệu, tải lên và tải xuống.
- Cấu hình HTTPS và kiểm tra phiên đăng nhập trước khi sử dụng tài khoản thật của bài lab.
- Chuẩn bị tích hợp S3 bằng IAM Role và ghi nhật ký phục vụ giám sát.

## Chuẩn bị trước khi triển khai

| Thành phần | Thông tin cần có |
| --- | --- |
| Region | Singapore — ap-southeast-1 |
| VPC | vpc-secure-web — 10.0.0.0/16 |
| Subnet | subnet-web-public — 10.0.0.0/24 |
| Route Table | rtb-web-public, có tuyến Internet qua igw-secure-web và tuyến S3 qua s3-gateway-endpoint |
| Security Group | sg-secure-web, SSH từ Public IPv4/32 quản trị và HTTPS từ Internet |
| Network ACL | nacl-web-public, đã cấu hình lưu lượng yêu cầu và phản hồi |
| Tên EC2 dự kiến | workshop-web-01 |
| Key pair dự kiến | workshop-web-key |
| IAM Role / Instance profile | WorkshopEC2AppRole / WorkshopEC2AppRole, theo cấu hình thực tế quản trị viên đã chuẩn bị |
| S3 | Tên bucket thực tế theo 5.2.2, cùng Region; prefix portal/ |

Hoàn thành [5.3. Thiết lập mạng và cấu hình Firewall](../5.3-network-and-firewall/) và đối chiếu ID tài nguyên từ bảng bàn giao. Người tạo EC2 cần quyền khởi tạo tài nguyên và quyền iam:PassRole đối với role được phép sử dụng. Không thay thế quyền còn thiếu bằng access key gắn trong mã nguồn.

Ở 5.4.1, bật **Auto-assign public IP** khi tạo EC2 trong subnet đã chuẩn bị. Dùng Security Group hiện có; không để trình khởi tạo tạo nhóm mới với SSH mở cho toàn Internet.

Nếu bucket và quyền S3 chưa hoàn tất, có thể triển khai máy chủ và giao diện trước. Kiểm thử upload/download đầy đủ phải chờ phần 5.5. Không coi trang đăng nhập hiển thị thành công là bằng chứng đã lưu được dữ liệu lên S3.

## Phạm vi ứng dụng

Website sử dụng một tài khoản dùng chung được tạo sẵn cho bài lab. Tài khoản website độc lập với IAM user dùng để thao tác AWS Console. Không triển khai đăng ký, quên mật khẩu, phân quyền nhiều người dùng hoặc cơ sở dữ liệu trong phạm vi này.

| Chức năng | Yêu cầu |
| --- | --- |
| Đăng nhập | Kiểm tra thông tin tài khoản, mật khẩu được lưu dưới dạng băm |
| Phiên đăng nhập | Chặn trang tài liệu và các API nếu chưa đăng nhập; đăng xuất làm mất hiệu lực phiên |
| Danh sách tài liệu | Hiển thị tài liệu thuộc prefix portal/ của bucket đã chỉ định |
| Tải lên | Chấp nhận PDF/TXT, tối đa 5 MB mỗi tệp; kiểm tra tại máy chủ |
| Tải xuống | Máy chủ kiểm tra phiên, đọc tài liệu từ S3 và trả nội dung về người dùng |
| Dữ liệu mẫu | Chỉ sử dụng tài liệu thực hành, không chứa dữ liệu cá nhân hoặc bí mật |

## Luồng xử lý và yêu cầu bảo mật

Người dùng truy cập EC2 qua HTTPS. Ứng dụng kiểm tra phiên đăng nhập trước khi xử lý yêu cầu tài liệu, sau đó gọi S3 bằng quyền của IAM Role gắn với EC2. Lưu lượng S3 cùng Region sử dụng Gateway Endpoint đã tạo ở 5.3.5. Bucket vẫn riêng tư; trình duyệt nhận tài liệu thông qua ứng dụng.

IAM Role cung cấp thông tin xác thực tạm thời cho ứng dụng thông qua instance profile. Dùng cơ chế lấy thông tin xác thực của AWS SDK; không lưu access key dài hạn trong mã nguồn, Git hoặc file cấu hình công khai.

Mật khẩu băm và khóa bí mật phiên phải nằm ngoài thư mục web công khai. Không ghi mật khẩu, cookie phiên hoặc nội dung tài liệu vào log. Chỉ công khai cổng web cần thiết; backend nội bộ không mở trực tiếp ra Internet. Cổng 80 dùng để kiểm tra Nginx ban đầu và chuyển hướng sang HTTPS.

Nếu dùng chứng chỉ tự ký cho lab, phải nêu rõ giới hạn tin cậy của trình duyệt và cách kiểm tra chứng chỉ ở 5.4.4. Không mô tả chứng chỉ tự ký là chứng chỉ được trình duyệt công khai tin cậy.

## Nội dung thực hiện

- [5.4.1. Khởi tạo EC2 và kết nối SSH](./5.4.1-launch-and-connect/)
- [5.4.2. Cài đặt máy chủ web](./5.4.2-install-web-server/)
- [5.4.3. Triển khai đăng nhập và chức năng tải lên, tải xuống tài liệu](./5.4.3-document-web-app/)
- [5.4.4. Cấu hình HTTPS và kiểm tra phiên đăng nhập](./5.4.4-https-and-session/)

Thực hiện theo thứ tự trên. Phần 5.4.3 chuẩn bị ứng dụng; hoàn thành HTTPS ở 5.4.4 trước khi sử dụng đăng nhập qua mạng. Sau đó hoàn thiện bucket và quyền dữ liệu tại 5.5, giám sát tại 5.6, kiểm thử tổng thể tại 5.7.

## Kết quả cần đạt

| Hạng mục | Bằng chứng cần có |
| --- | --- |
| Máy chủ | EC2 hoạt động trong đúng VPC/subnet và gắn sg-secure-web |
| Quản trị | Kết nối SSH thành công từ nguồn /32 được cho phép |
| Dịch vụ web | Dịch vụ chạy và phục hồi sau khi khởi động lại máy chủ |
| HTTPS và phiên | Đăng nhập/đăng xuất đúng; yêu cầu chưa xác thực bị từ chối |
| Tích hợp dữ liệu | Upload/download hoạt động sau khi bucket và quyền ở 5.5 hoàn tất |
| Bàn giao | Instance ID, địa chỉ truy cập, role/profile và vị trí log; không bàn giao bí mật trong tài liệu công khai |

![](/FCAJ_Worklog/images/54/ec2tong.png)


Ảnh thao tác chi tiết sẽ được đặt tại từng mục con. Giữ quy mô một EC2 cho bài lab; theo dõi thời gian sử dụng và dọn dẹp ở 5.8. Không mặc định EC2, EBS và Public IPv4 đều miễn phí.

Bắt đầu: [5.4.1. Khởi tạo EC2 và kết nối SSH](./5.4.1-launch-and-connect/).

## Tài liệu tham khảo

- [Use an IAM role for applications on EC2](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html)
