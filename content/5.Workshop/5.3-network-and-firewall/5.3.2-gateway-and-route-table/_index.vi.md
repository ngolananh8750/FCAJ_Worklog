---
title: "Tạo Internet Gateway và Route Table"
date: 2026-10-29
weight: 2
chapter: false
pre: "<b>5.3.2. </b>"
---



## Mục tiêu

Tạo **igw-secure-web**, gắn vào **vpc-secure-web**, sau đó tạo **rtb-web-public** với tuyến Internet và liên kết với **subnet-web-public**. Thành viên 1 thực hiện trên AWS Management Console.

Sau bước này, subnet có tuyến trực tiếp đến Internet Gateway và trở thành Public Subnet. EC2 vẫn cần Public IPv4, Security Group, Network ACL phù hợp và dịch vụ đang chạy để kết nối được; chưa thể kết luận website hoạt động.

## Chuẩn bị

- Hoàn thành [5.3.1. Tạo VPC và Public Subnet](../5.3.1-vpc-and-subnet/).
- Đăng nhập bằng IAM user phụ trách mạng có quyền tạo/gắn Internet Gateway, tạo Route Table, thêm route, liên kết subnet và gắn tag.
- Chọn **Singapore — ap-southeast-1**.
- Đối chiếu VPC ID và Subnet ID thực tế từ bảng bàn giao.
- Nếu tài nguyên đã tồn tại, kiểm tra và dùng lại; không tạo trùng hoặc tháo gateway đang sử dụng.

| Thành phần | Giá trị |
| --- | --- |
| VPC | vpc-secure-web — 10.0.0.0/16 |
| Subnet | subnet-web-public — 10.0.0.0/24 |
| Internet Gateway | igw-secure-web |
| Route Table | rtb-web-public |
| Tuyến Internet IPv4 | 0.0.0.0/0 → Internet Gateway igw-secure-web |
| Tags | Project=SecureWebWorkshop |

## Bước 1: Tạo Internet Gateway

1. Mở **VPC → Internet gateways → Create internet gateway**.
2. Điền **Name tag: igw-secure-web**.
3. Thêm các tag dùng chung trong bảng trên.
4. Kiểm tra thông tin rồi chọn **Create internet gateway**.
![](/FCAJ_Worklog/images/005/igw1.png)
## Bước 2: Gắn Internet Gateway vào VPC

1. Chọn **igw-secure-web → Actions → Attach to VPC**.
![](/FCAJ_Worklog/images/005/igw2.png)
2. Chọn **vpc-secure-web**, đối chiếu đúng VPC ID.
![](/FCAJ_Worklog/images/005/igw3.png)
3. Chọn **Attach internet gateway**.
4. Kiểm tra trạng thái **Attached** và VPC được gắn.

Một VPC chỉ có một Internet Gateway được gắn tại một thời điểm. Nếu vpc-secure-web không xuất hiện trong danh sách, kiểm tra Region và gateway đã gắn sẵn; không tháo gateway của hệ thống khác để tiếp tục.

![](/FCAJ_Worklog/images/005/igw4.png)


## Bước 3: Tạo Route Table riêng

1. Trong menu VPC, chọn **Route tables → Create route table**.
2. Nhập **Name: rtb-web-public**.
3. Chọn **VPC: vpc-secure-web** theo ID đã ghi.
4. Thêm tag Project, Environment và Owner như bảng chuẩn bị.
5. Chọn **Create route table**.
6. Ghi Route Table ID và kiểm tra VPC ID.

Giữ nguyên main route table mặc định. Không chọn thao tác thay thế main route table; chỉ liên kết subnet của workshop với bảng mới.

![](/FCAJ_Worklog/images/005/rtb1.png)


## Bước 4: Thêm tuyến Internet

1. Chọn **rtb-web-public → Routes → Edit routes → Add route**.
2. Nhập **Destination: 0.0.0.0/0**.
3. Ở **Target**, chọn **Internet Gateway**, sau đó chọn ID của **igw-secure-web**.
4. Giữ nguyên tuyến **10.0.0.0/16 → local**.
5. Chọn **Save changes**, kiểm tra tuyến mới ở trạng thái **Active**.

| Destination | Target | Status mong đợi |
| --- | --- | --- |
| 10.0.0.0/16 | local | Active |
| 0.0.0.0/0 | ID thực tế của igw-secure-web | Active |

**0.0.0.0/0 ở đây là đích định tuyến**, không phải quy tắc cho phép mọi cổng vào EC2. Không nhầm với việc mở SSH cho toàn Internet. Không chọn NAT Gateway, không thêm tuyến IPv6 trong phạm vi bài thực hành IPv4 này.

![](/FCAJ_Worklog/images/005/rtb2.png)


## Bước 5: Liên kết Route Table với subnet

1. Chọn **Subnets → subnet-web-public → Route table**.
2. Chọn **Edit route table association**.
3. Chọn ID của **rtb-web-public** và lưu bằng **Save**.
4. Kiểm tra lại tab Route table của subnet: phải hiển thị bảng mới và tuyến Internet.
5. Có thể đối chiếu thêm tại **Route tables → rtb-web-public → Subnet associations**: subnet-web-public nằm trong **Explicit subnet associations**.

Chỉ thêm route vào một bảng chưa được subnet sử dụng sẽ không thay đổi đường đi của subnet. Không chỉ nhìn tên bảng; phải kiểm tra ID và liên kết.

![](/FCAJ_Worklog/images/005/rtb3.png)




## Bước 6: Kiểm tra và bàn giao

| Kiểm tra | Kết quả cần đạt |
| --- | --- |
| igw-secure-web | Attached vào đúng vpc-secure-web |
| rtb-web-public | Thuộc vpc-secure-web, không thay main route table |
| Tuyến local | 10.0.0.0/16 → local |
| Tuyến mặc định | 0.0.0.0/0 → ID igw-secure-web, Active |
| Liên kết subnet | subnet-web-public dùng rtb-web-public |
| ID cần ghi | Internet Gateway ID, Route Table ID, VPC ID, Subnet ID |

Lưu các ID thực tế vào bảng bàn giao và thông báo cho thành viên 2. Ở phần 5.4.1, EC2 sẽ được tạo trong subnet này và bật cấp Public IPv4 khi khởi tạo.

**Chưa kiểm thử website hoặc ping Internet tại bước này khi chưa có EC2.** Đây là kiểm tra cấu hình định tuyến. S3 Gateway VPC Endpoint sẽ được bổ sung vào Route Table ở 5.3.5.

## Lỗi thường gặp

| Hiện tượng | Kiểm tra và xử lý |
| --- | --- |
| Không thấy igw-secure-web khi chọn Target | Kiểm tra Region, VPC của Route Table và việc gateway đã Attached đúng VPC |
| Tuyến hiện Blackhole | Kiểm tra gateway đích có bị xóa hoặc tháo khỏi VPC; khôi phục đúng liên kết/target |
| Báo đã tồn tại tuyến 0.0.0.0/0 | Kiểm tra tuyến hiện có; nếu đúng thì giữ lại, nếu sai thì sửa target thay vì thêm trùng |
| Subnet vẫn dùng bảng mặc định | Thực hiện lại Bước 5 và kiểm tra ID bảng thực tế |
| Không chọn được rtb-web-public cho subnet | Kiểm tra bảng và subnet thuộc cùng VPC |
| AccessDenied | Ghi action và lỗi, nhờ quản trị kiểm tra quyền tạo route/gắn gateway/liên kết subnet và điều kiện policy |





Tiếp tục: [5.3.3. Cấu hình Security Group](../5.3.3-security-group/).

## Tài liệu tham khảo

- [Add internet access to a subnet](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-igw.html)
- [Manage subnet route tables](https://docs.aws.amazon.com/vpc/latest/userguide/WorkWithRouteTables.html)
