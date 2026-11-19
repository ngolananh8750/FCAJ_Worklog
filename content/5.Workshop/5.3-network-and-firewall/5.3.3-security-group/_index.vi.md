---
title: "Cấu hình Security Group"
date: 2026-10-30
weight: 3
chapter: false
pre: "<b>5.3.3. </b>"
---


## Mục tiêu

Tạo **sg-secure-web** trong **vpc-secure-web** để kiểm soát kết nối đến Web Server lưu trữ tài liệu. Thành viên 1 cấu hình trên AWS Management Console, sau đó bàn giao Security Group ID cho thành viên 2 khi tạo EC2 ở phần 5.4.1.

Security Group áp dụng cho tài nguyên được gắn với nó, không gắn trực tiếp vào subnet. Security Group có trạng thái (stateful): lưu lượng phản hồi của kết nối đã được cho phép không cần quy tắc chiều ngược lại riêng. Vì vậy, không mở dải cổng tạm thời trong Security Group chỉ để nhận phản hồi; phần Network ACL ở 5.3.4 sẽ xử lý riêng.

## Chuẩn bị

- Hoàn thành [5.3.2. Tạo Internet Gateway và Route Table](../5.3.2-gateway-and-route-table/).
- Chọn **Singapore — ap-southeast-1** và đối chiếu VPC ID của **vpc-secure-web**.
- Tài khoản thực hiện có quyền tạo Security Group, quản lý inbound/outbound rules và gắn tag theo phần 5.2.1.
- Xác định Public IPv4 của máy quản trị sẽ thực hiện SSH. Không dùng địa chỉ LAN như 192.168.x.x.
- Nếu sg-secure-web đã tồn tại, kiểm tra VPC ID và quy tắc trước khi dùng lại; không tạo trùng.

| Thuộc tính | Giá trị |
| --- | --- |
| Security group name | sg-secure-web |
| Description | Security group for workshop document web server |
| VPC | vpc-secure-web — 10.0.0.0/16 |
| Name tag | sg-secure-web |
| Tags khác | Project=SecureWebWorkshop |

## Bước 1: Xác định quy tắc truy cập

Sử dụng cấu hình inbound sau cho website của workshop:

| Type | Protocol | Port range | Source | Mục đích |
| --- | --- | --- | --- | --- |
| SSH | TCP | 22 | PUBLIC_IP_QUAN_TRI/32 | Quản trị EC2 từ máy đã được cho phép |
| HTTPS | TCP | 443 | 0.0.0.0/0 | Người dùng truy cập website qua HTTPS |
| HTTP | TCP | 80 | 0.0.0.0/0 | Trang Nginx ban đầu, sau đó chuyển sang HTTPS |

**PUBLIC_IP_QUAN_TRI/32 là chỗ giữ chỗ, phải thay bằng Public IPv4 thực tế với hậu tố /32.** Không nhập nguyên chuỗi này vào Console. Mỗi quản trị viên ở một địa chỉ Internet khác nhau cần một quy tắc SSH /32 riêng.

Quy tắc HTTPS cho phép người dùng kết nối đến máy chủ; ứng dụng vẫn phải kiểm tra đăng nhập và quyền truy cập tài liệu. Việc mở cổng 443 không tự cài chứng chỉ hoặc bật HTTPS.

Cho phép 80 để thử Nginx và chuyển hướng HTTPS, giới hạn SSH vào IP quản trị /32, giữ cổng 81 và 8000 ngoài danh sách cho phép.

## Bước 2: Tạo Security Group

1. Mở **VPC → Security groups → Create security group**.
2. Điền tên và Description theo bảng chuẩn bị.
3. Ở **VPC**, chọn **vpc-secure-web** theo đúng VPC ID; bỏ lựa chọn VPC mặc định nếu đang được chọn.
4. Thêm các tag Name, Project, Environment và Owner.
5. Tạm giữ phần inbound trống và outbound mặc định; chọn **Create security group**. Các quy tắc inbound sẽ được bổ sung ngay ở bước tiếp theo.
6. Ghi lại Security Group ID có dạng `sg-...`.

![](/FCAJ_Worklog/images/005/sg1.png)


## Bước 3: Cấu hình inbound rules

1. Chọn **sg-secure-web → Inbound rules → Edit inbound rules**.
2. Chọn **Add rule**, Type **SSH**, TCP **22**.
3. Chọn Source **My IP** nếu đang thao tác từ đúng máy và đường mạng sẽ dùng để SSH. Kiểm tra CIDR /32 được điền; nếu cấu hình cho thành viên khác, chọn **Custom** và nhập Public IPv4/32 của người đó.
4. Nhập Description: `SSH from administrator public IP`.
5. Thêm quy tắc Type **HTTPS**, TCP **443**, Source **Anywhere-IPv4 — 0.0.0.0/0**; Description: `HTTPS document portal`.
6. Kiểm tra không có quy tắc SSH rộng hoặc quy tắc thừa rồi chọn **Save rules**.
7. Kiểm tra lại tab Inbound rules sau khi lưu.

Nếu trình duyệt và SSH đi qua hai VPN/proxy khác nhau, My IP có thể không trùng địa chỉ nguồn SSH. Xác nhận địa chỉ Internet của đường kết nối SSH. Khi IP quản trị thay đổi, cập nhật quy tắc /32 và xóa địa chỉ cũ không còn sử dụng.

![](/FCAJ_Worklog/images/005/sg2.png)

![](/FCAJ_Worklog/images/005/sg4.png)


## Bước 4: Kiểm tra outbound rules

Workshop giữ quy tắc outbound IPv4 mặc định để EC2 có thể tải gói cài đặt và kết nối tới S3, CloudWatch cùng các dịch vụ cần thiết:

| Type | Protocol | Port range | Destination |
| --- | --- | --- | --- |
| All traffic | All | All | 0.0.0.0/0 |

1. Mở tab **Outbound rules** của sg-secure-web.
2. Kiểm tra quy tắc trong bảng. Nếu đã đúng, giữ nguyên.
3. Nếu đang thiếu, chọn **Edit outbound rules → Add rule**, nhập giá trị trên rồi **Save rules**.

Đây là lựa chọn đơn giản cho bài lab, chưa giới hạn chặt lưu lượng đi ra. Cho phép outbound không mở thêm cổng inbound và không cấp quyền đọc/ghi S3. Quyền dữ liệu sẽ do IAM Role và bucket policy kiểm soát. S3 Gateway Endpoint được tạo ở 5.3.5; Security Group không thay thế cấu hình định tuyến.

![](/FCAJ_Worklog/images/005/sg3.png)


## Bước 5: Kiểm tra và bàn giao

| Nội dung | Kết quả cần đạt |
| --- | --- |
| Tên và VPC | sg-secure-web thuộc vpc-secure-web |
| SSH | TCP 22 chỉ từ Public IPv4/32 của quản trị viên |
| HTTPS | TCP 443 từ 0.0.0.0/0 |
| HTTP | TCP 80 |
| Cổng backend | Không có inbound từ Internet |
| Outbound | All traffic đến 0.0.0.0/0 theo phạm vi lab |
| Thông tin bàn giao | Region, VPC ID, Security Group ID và nguồn SSH đã thống nhất |

Ghi ID thực tế vào bảng bàn giao. Ở **5.4.1**, thành viên 2 chọn **Select existing security group → sg-secure-web** khi tạo **workshop-web-01** trong subnet-web-public. Không để trình khởi tạo tự tạo nhóm khác với SSH mở rộng. Nếu gắn nhiều Security Group, các quy tắc cho phép được cộng dồn; nhóm khác có thể mở rộng truy cập ngoài dự kiến.

Chưa thực hiện kiểm thử SSH hoặc HTTPS khi chưa có EC2 và dịch vụ web. Sau khi triển khai, kiểm tra ở phần 5.7: SSH từ nguồn được phép, SSH từ nguồn không được phép và truy cập HTTPS. Nếu có lỗi, kiểm tra cả Public IPv4, route, NACL và dịch vụ đang lắng nghe.

## Lỗi thường gặp

| Hiện tượng | Cách xử lý |
| --- | --- |
| Không tìm thấy vpc-secure-web | Kiểm tra Region và quyền xem VPC |
| Tên Security Group đã tồn tại | Đối chiếu ID/VPC và dùng lại nhóm đúng của workshop |
| My IP khác IP máy quản trị | Kiểm tra VPN/proxy và nhập đúng Public IPv4/32 của kết nối SSH |
| SSH bị mất kết nối sau khi đổi mạng | Cập nhật nguồn /32, không mở SSH cho toàn Internet |
| Quy tắc bị báo trùng | Giữ quy tắc đúng hiện có, không thêm bản sao |
| AccessDenied khi tạo hoặc sửa | Ghi lại action bị từ chối và nhờ quản trị kiểm tra policy theo 5.2.1 |
| Sau này vẫn truy cập được cổng ngoài dự kiến | Kiểm tra toàn bộ Security Group đang gắn vào network interface của EC2 |




Tiếp tục: [5.3.4. Cấu hình Network ACL](../5.3.4-network-acl/).

## Tài liệu tham khảo

- [Create a security group](https://docs.aws.amazon.com/vpc/latest/userguide/creating-security-groups.html)
- [Configure security group rules](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-security-group-rules.html)
- [Security group rules](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html)
- [Security group basics](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
