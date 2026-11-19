---
title: "Tạo VPC và Public Subnet"
date: 2026-10-29
weight: 1
chapter: false
pre: "<b>5.3.1. </b>"
---



## Mục tiêu

Tạo VPC **vpc-secure-web** và subnet **subnet-web-public** để chuẩn bị triển khai EC2 cho website tài liệu. Thành viên 1 thực hiện trên AWS Management Console.


## Chuẩn bị

- IAM user phụ trách mạng có quyền tạo VPC, subnet, gắn tag và thay đổi thuộc tính DNS.
- Region **Asia Pacific (Singapore) — ap-southeast-1**.
- Nếu đã có tài nguyên đúng cấu hình, kiểm tra và sử dụng lại; không tạo bản trùng.
- Chuẩn bị thư mục ảnh để lưu các ảnh minh chứng theo số thứ tự bên dưới.

| Thông số | Giá trị |
| --- | --- |
| VPC Name | vpc-secure-web |
| VPC IPv4 CIDR | 10.0.0.0/16 |
| Tenancy | Default |
| IPv6 | Không cấp trong bài thực hành này |
| Subnet Name | subnet-web-public |
| Subnet IPv4 CIDR | 10.0.0.0/24 |
| Availability Zone | ap-southeast-1a nếu khả dụng |
| DNS resolution / DNS hostnames | Bật cả hai |
| Tags | Project=SecureWebWorkshop |



## Bước 1: Mở VPC Console

1. Đăng nhập AWS Management Console.
2. Tìm **VPC** trên thanh tìm kiếm và mở dịch vụ.
3. Kiểm tra Region ở góc trên, chọn **Singapore — ap-southeast-1**.
4. Chọn **Your VPCs** ở menu bên trái.



![vpc1](/FCAJ_Worklog/images/005/vpc1.png)


## Bước 2: Tạo VPC

1. Chọn **Create VPC**.
2. Trong **Resources to create**, chọn **VPC only**.
3. Nhập **Name tag: vpc-secure-web**.
4. Chọn nhập IPv4 CIDR thủ công và điền **10.0.0.0/16**.
5. Chọn **No IPv6 CIDR block**, giữ **Tenancy: Default**.
6. Thêm các tag Project, Environment và Owner như bảng chuẩn bị; giữ tag Name.
7. Kiểm tra thông tin trước khi tạo.

Dùng **VPC only** để thực hiện riêng từng thành phần theo mục lục; không dùng trình tạo tự động VPC and more trong hướng dẫn này.


![vpc2](/FCAJ_Worklog/images/005/vpc2.png)


1. Chọn **Create VPC**.
2. Mở VPC vừa tạo, kiểm tra trạng thái **Available** và CIDR.
3.  Sao chép **VPC ID** thực tế vào bảng bàn giao.

![vpc3](/FCAJ_Worklog/images/005/vpc3.png)


## Bước 3: Bật DNS cho VPC

1. Tại **Your VPCs**, chọn đúng **vpc-secure-web**.
2. Chọn **Actions → Edit VPC settings**.
3. Bật **Enable DNS resolution** và **Enable DNS hostnames**.
4. Chọn **Save changes**.
5. Kiểm tra lại phần Details để xác nhận cả hai thuộc tính đã bật.

![vpc4](/FCAJ_Worklog/images/005/vpc4.png)


## Bước 4: Tạo subnet

1. Chọn **Subnets → Create subnet**.
2. Trong **VPC ID**, chọn VPC **vpc-secure-web** vừa tạo; đối chiếu ID ở Bước 2.
3. Nhập các giá trị sau:

| Trường | Giá trị nhập |
| --- | --- |
| Subnet name | subnet-web-public |
| Availability Zone | ap-southeast-1a |
| IPv4 VPC CIDR block, nếu có | 10.0.0.0/16 |
| IPv4 subnet CIDR block | 10.0.0.0/24 |

4. Nếu AZ đề xuất không khả dụng, chọn một AZ khác trong Singapore và ghi lại.
5. Thêm tag Project=SecureWebWorkshop cho subnet.
6. Chỉ tạo **một subnet** trong bước này.

![](/FCAJ_Worklog/images/005/subnet1.png)

7. Chọn **Create subnet**.
8. Mở subnet mới, kiểm tra trạng thái **Available**.
9. Ghi **Subnet ID**, **VPC ID**, **AZ** và **CIDR** vào bảng bàn giao.

![](/FCAJ_Worklog/images/005/subnet2.png)


## Bước 5: Kiểm tra và bàn giao



| Nội dung | Kết quả mong đợi / giá trị cần ghi |
| --- | --- |
| Region | ap-southeast-1 |
| VPC | vpc-secure-web, trạng thái Available |
| VPC ID | Điền ID thực tế |
| VPC CIDR | 10.0.0.0/16 |
| DNS resolution | Enabled |
| DNS hostnames | Enabled |
| Subnet | subnet-web-public, trạng thái Available |
| Subnet ID | Điền ID thực tế |
| Subnet CIDR | 10.0.0.0/24 |
| AZ | Điền AZ đã chọn |
| Subnet thuộc đúng VPC | VPC ID khớp với vpc-secure-web |
| Kết nối Internet | Chưa cấu hình trong mục này |

**Không cần bật auto-assign Public IPv4 cho toàn subnet ở bước này.** Theo phần 5.2.2, thành viên 2 sẽ bật **Auto-assign public IP: Enable** khi khởi tạo EC2 ở phần 5.4.1. Việc tạo subnet chưa cấp Public IPv4 cho một máy chủ vì chưa có EC2.

VPC mới có các thành phần mặc định như main route table, default Security Group và default Network ACL. Không xóa chúng trong bước này. Route Table riêng, Security Group và Network ACL cho workshop được xử lý ở các mục sau.

## Lỗi thường gặp

| Hiện tượng | Cách xử lý |
| --- | --- |
| Không thấy VPC vừa tạo | Kiểm tra AWS account, Region và bộ lọc danh sách |
| AccessDenied khi tạo hoặc gắn tag | Gửi action và thông báo lỗi cho quản trị; kiểm tra quyền tạo, gắn tag và điều kiện policy |
| CIDR subnet không hợp lệ | Kiểm tra 10.0.0.0/24 nằm trong VPC 10.0.0.0/16 |
| CIDR subnet bị trùng | Kiểm tra subnet đã tồn tại; không tạo thêm nếu nhóm đã có subnet đúng cấu hình |
| Không thấy AZ ap-southeast-1a | Chọn AZ khả dụng khác cùng Region và cập nhật bảng bàn giao |
| Subnet chưa truy cập được Internet | Đây là trạng thái dự kiến; tiếp tục cấu hình Internet Gateway và Route Table ở 5.3.2 |




Tiếp tục: [5.3.2. Tạo Internet Gateway và Route Table](../5.3.2-gateway-and-route-table/).

## Tài liệu tham khảo

- [Create a VPC](https://docs.aws.amazon.com/vpc/latest/userguide/create-vpc.html)
- [Create a subnet](https://docs.aws.amazon.com/vpc/latest/userguide/create-subnets.html)
- [View and update DNS attributes for your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns-updating.html)
