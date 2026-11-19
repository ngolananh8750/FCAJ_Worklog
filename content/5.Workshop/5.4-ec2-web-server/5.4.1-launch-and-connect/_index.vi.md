---
title: "Khởi tạo EC2 và kết nối SSH"
date: 2026-10-30
weight: 1
chapter: false
pre: "<b>5.4.1. </b>"
---



## Mục tiêu

Thành viên 2 tạo **workshop-web-01** trong mạng đã chuẩn bị, sau đó kết nối SSH từ máy cá nhân. Phần này chỉ khởi tạo và kiểm tra máy chủ; cài đặt dịch vụ web được thực hiện ở 5.4.2.

## Chuẩn bị

- Hoàn thành phần 5.3 và đối chiếu ID của vpc-secure-web, subnet-web-public, sg-secure-web.
- Region: **Singapore — ap-southeast-1**.
- IAM user có quyền tạo EC2, key pair, EBS, gắn tag và dùng instance profile; iam:PassRole chỉ dành cho role đã được phép.
- Quản trị viên đã chuẩn bị **WorkshopEC2AppRole** và instance profile **WorkshopEC2AppRole**. Tên profile thực tế có thể khác nếu tạo bằng Console; kiểm tra profile chứa đúng role.
- Máy cá nhân có OpenSSH client: Terminal trên Linux/macOS hoặc PowerShell trên Windows. Kiểm tra bằng `ssh -V`.
- Public IPv4/32 của máy SSH đã có trong cả Security Group và NACL. Nếu đổi mạng/VPN, cập nhật nguồn trước.

## Bước 1: Đặt tên, chọn AMI và loại máy

Mở **EC2 → Instances → Launch instances**, cấu hình:

| Trường | Giá trị cho bài lab |
| --- | --- |
| Name | workshop-web-01 |
| AMI | Amazon Linux 2023, bản tiêu chuẩn do Amazon cung cấp |
| Architecture | 64-bit (x86) / x86_64 |
| Instance type | t3.micro |
| Number of instances | 1 |
| Tags | Project=SecureWebWorkshop |

Cấu hình trên là lựa chọn triển khai cho các phần tiếp theo, sử dụng tài khoản hệ điều hành **ec2-user** và trình quản lý gói **dnf**. Không chọn nhầm AMI Marketplace hoặc Arm khi dùng t3.micro. Ghi AMI ID thực tế vì ID thay đổi theo Region và phiên bản.

Kiểm tra chi phí và quyền lợi tài khoản trước khi khởi tạo; không mặc định nhãn Free tier đồng nghĩa toàn bộ bài lab miễn phí. Nếu tài khoản không cho phép t3.micro, nhờ quản trị thống nhất loại máy x86_64 thay thế trước khi tiếp tục.

![](/FCAJ_Worklog/images/54/ec21.png)




## Bước 2: Tạo hoặc chọn key pair

1. Tại **Key pair (login)**, chọn **Create new key pair** nếu chưa có khóa hợp lệ.
2. Nhập **workshop-web-key**, chọn **RSA**, định dạng **.pem**.
3. Chọn **Create key pair**, lưu file tải xuống ở thư mục riêng trên máy SSH.
4. Xác nhận workshop-web-key đang được chọn trong biểu mẫu khởi tạo.

Nếu key pair đã tồn tại, chỉ dùng lại khi đang giữ đúng private key. AWS không cho tải lại private key của key pair đã tạo. Không chọn tiếp tục không có key pair với phương thức SSH của bài này. Không đưa file .pem lên Git, website workshop hoặc ảnh chụp.

![](/FCAJ_Worklog/images/54/ec23.png)


## Bước 3: Chọn mạng và Security Group

Trong **Network settings → Edit**, điền:

| Trường | Giá trị |
| --- | --- |
| VPC | vpc-secure-web |
| Subnet | subnet-web-public |
| Auto-assign public IP | Enable |
| Firewall | Select existing security group |
| Security group | sg-secure-web |

Đối chiếu các ID thực tế, không chỉ tên. Không tạo Security Group mới hoặc mở SSH cho 0.0.0.0/0. Không cần Elastic IP cho bước thực hành này.

![](/FCAJ_Worklog/images/54/ec22.png)


## Bước 4: Cấu hình lưu trữ và bảo mật

1. Trong **Configure storage**, dùng một root volume **8 GiB, gp3** nếu AMI đã chọn cho phép; không giảm dưới kích thước tối thiểu của AMI.
2. Mở phần nâng cao của volume, bật **Encrypted**, dùng khóa mặc định **aws/ebs** cho lab; giữ **Delete on termination** để volume gốc được xóa khi terminate instance.
3. Mở **Advanced details → IAM instance profile**, chọn profile chứa **WorkshopEC2AppRole**.
4. Giữ metadata endpoint Enabled; đặt **Metadata version: V2 only (token required)** để yêu cầu IMDSv2.
5. Với **Credit specification**, chọn **Standard** để tránh phí surplus CPU credit của chế độ Unlimited. Khi hết credit, hiệu năng có thể giảm về mức nền.
6. Giữ detailed CloudWatch monitoring tắt trong bước này; để **User data** trống. Không nhập mật khẩu, private key hay access key vào User data.

Nếu không thấy instance profile, nhờ quản trị kiểm tra profile, role và quyền liệt kê/PassRole; không gắn một role có quyền rộng tùy ý. Quyền S3 cụ thể sẽ được hoàn thiện ở 5.5.3.

![](/FCAJ_Worklog/images/54/ec24.png)


![](/FCAJ_Worklog/images/54/ec26.png)


## Bước 5: Khởi tạo và kiểm tra instance

1. Đối chiếu toàn bộ cấu hình, Number of instances = **1**, rồi chọn **Launch instance**.
2. Chọn **View all instances**, mở workshop-web-01 vừa tạo.
3. Chờ **Instance state: Running** và các status checks hiển thị đều Passed.
4. Ghi Instance ID, Public IPv4, Private IPv4, Availability Zone và AMI ID.
5. Kiểm tra tab **Security** có sg-secure-web và IAM Role đúng; tab **Networking** có đúng VPC/subnet.

Nếu chưa có Public IPv4, kiểm tra cấu hình launch; không dùng Private IPv4 để SSH trực tiếp từ Internet. Chưa thử mở website vì chưa cài dịch vụ web.

![](/FCAJ_Worklog/images/54/ec27.png)


## Bước 6: Kết nối SSH từ máy cá nhân

Các lệnh dưới đây chạy **trên máy cá nhân**, trong thư mục chứa YOUR_KEY_PAIR.pem. Thay **PUBLIC_IPV4** bằng Public IPv4 thực tế của instance.

**Linux/macOS:**

```bash
chmod 400 YOUR_KEY_PAIR.pem
ssh -i "YOUR_KEY_PAIR.pem" ec2-user@PUBLIC_IPV4
```

**Windows PowerShell có OpenSSH:**

```powershell
ssh -i ".\YOUR_KEY_PAIR.pem" ec2-user@PUBLIC_IPV4
```

Nếu Windows báo private key quá rộng quyền, mở Properties → Security → Advanced của file .pem, tắt kế thừa và chỉ giữ quyền đọc cần thiết cho tài khoản Windows đang dùng; không cấp quyền cho Everyone/Users. Sau đó kết nối lại.

Lần đầu SSH hỏi xác nhận host fingerprint. Đối chiếu với fingerprint của **instance**, lấy từ EC2 → Actions → Monitor and troubleshoot → Get system log, tìm phần SSH HOST KEY FINGERPRINTS nếu có. Không nhầm với fingerprint của key pair. Chỉ nhập `yes` sau khi xác nhận đúng máy; nếu không lấy được fingerprint, nhờ quản trị hỗ trợ xác minh.

Sau khi đăng nhập, chạy các lệnh không thay đổi cấu hình:

```bash
whoami
cat /etc/os-release
hostname
```

Kết quả cần có: user **ec2-user**, hệ điều hành **Amazon Linux 2023**. Không in thông tin xác thực từ metadata để chụp ảnh. Gõ `exit` khi muốn thoát SSH.

![](/FCAJ_Worklog/images/54/ec28.png)


## Lỗi thường gặp

| Hiện tượng | Kiểm tra/xử lý |
| --- | --- |
| Connection timed out | Public IPv4, route IGW, nguồn SSH /32 trong SG/NACL và outbound ephemeral của NACL; kiểm tra mạng cá nhân có chặn TCP 22 |
| Permission denied (publickey) | Đúng ec2-user, đúng private key của key pair đã gắn khi launch và đúng instance |
| Unprotected private key file | Sửa quyền file trên máy cá nhân; Linux/macOS dùng chmod 400 |
| Host key changed | Kiểm tra có đổi IP hoặc tạo lại máy không; xác minh fingerprint trước khi cập nhật known_hosts, không tắt kiểm tra host key |
| AccessDenied khi Launch | Ghi action bị từ chối; kiểm tra quyền EC2/EBS/tag/PassRole và điều kiện policy |
| Instance dừng/terminate ngay | Kiểm tra State transition reason, quyền mã hóa EBS và cấu hình AMI/instance |
| Browser EC2 Instance Connect thất bại | Bài này dùng SSH trực tiếp từ IP cá nhân; browser connect có nguồn mạng khác, không mở SSH toàn Internet để xử lý |

## Kết quả và bàn giao

Ghi Instance ID, AMI ID, Public/Private IPv4, VPC/Subnet ID, Security Group ID, role/profile và key pair name. Không ghi private key vào tài liệu. Dán ảnh ở **7 vị trí** đã đánh dấu; một vị trí có thể cần nhiều ảnh để đủ thông tin.

Nếu tạm dừng thực hành, có thể Stop instance để giảm thời gian chạy; EBS vẫn tồn tại và có thể tính phí. Public IPv4 tự cấp có thể đổi sau Stop/Start: kiểm tra lại trước khi SSH. Dọn dẹp toàn bộ tài nguyên ở 5.8.

Tiếp tục: [5.4.2. Cài đặt máy chủ web](../5.4.2-install-web-server/).

## Tài liệu tham khảo

- [Launch an EC2 instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-launch-instance-wizard.html)
- [Instance configuration parameters](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-launch-parameters.html)
- [SSH connection prerequisites](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connection-prereqs-general.html)
- [Connect using an SSH client](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-linux-inst-ssh.html)
- [Configure CPU credit mode](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances-how-to.html)
- [Configure IMDSv2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-IMDS-new-instances.html)
