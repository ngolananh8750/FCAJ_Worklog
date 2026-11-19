---
title: "Tạo S3 Gateway VPC Endpoint"
date: 2026-10-30
weight: 5
chapter: false
pre: "<b>5.3.5. </b>"
---



## Mục tiêu

Tạo **s3-gateway-endpoint** loại **Gateway** trong vpc-secure-web, liên kết với **rtb-web-public** để EC2 truy cập S3 trong cùng Region qua endpoint. Thành viên 1 thực hiện và bàn giao thông tin cho thành viên 2, 3.

Gateway Endpoint không có phí sử dụng bổ sung; các chi phí S3 và tài nguyên khác vẫn áp dụng. Không tạo NAT Gateway hoặc S3 Interface Endpoint cho bài lab này.

## Chuẩn bị

- Hoàn thành [5.3.4. Cấu hình Network ACL](../5.3.4-network-acl/).
- Region: **Singapore — ap-southeast-1**.
- Xác nhận subnet-web-public đang sử dụng rtb-web-public trong vpc-secure-web.
- DNS resolution và DNS hostnames của VPC đã bật từ 5.3.1.
- Có quyền tạo VPC Endpoint, liên kết Route Table, gắn tag và xem tài nguyên mạng.
- Bucket tài liệu sẽ được tạo ở **ap-southeast-1** trong phần 5.5.1; chưa cần có bucket để tạo endpoint.

| Thuộc tính | Giá trị |
| --- | --- |
| Name | s3-gateway-endpoint |
| Service | com.amazonaws.ap-southeast-1.s3 |
| Type | Gateway |
| VPC | vpc-secure-web |
| IP address type | IPv4 |
| Route table | rtb-web-public |
| Policy ở bước khởi tạo | Full access |
| Tags | Project=SecureWebWorkshop |

## Bước 1: Chọn dịch vụ S3 Gateway

1. Mở **VPC → Endpoints → Create endpoint**.
2. Nhập Name tag **s3-gateway-endpoint**.
3. Chọn nhóm dịch vụ **AWS services**.
4. Lọc **Type = Gateway**, tìm và chọn **com.amazonaws.ap-southeast-1.s3**.
5. Kiểm tra cột Type là **Gateway**. Không chọn Interface hoặc dịch vụ s3express.

![](/FCAJ_Worklog/images/005/ep1.png)


## Bước 2: Chọn VPC và Route Table

1. Chọn **VPC: vpc-secure-web**, đối chiếu VPC ID thực tế.
2. Nếu có trường **IP address type**, chọn **IPv4**; nếu có lựa chọn DNS record IP type, chọn IPv4 phù hợp với mạng của workshop.
3. Trong **Route tables**, đánh dấu **rtb-web-public** theo đúng Route Table ID.
4. Đối chiếu bảng này đang liên kết với subnet-web-public, không chọn nhầm main route table mặc định.

Gateway Endpoint dùng liên kết Route Table; không yêu cầu chọn Security Group riêng cho endpoint như Interface Endpoint. Nếu màn hình yêu cầu chọn subnet/AZ và Security Group cho endpoint, kiểm tra lại Type.

![](/FCAJ_Worklog/images/005/ep2.png)


## Bước 3: Chọn policy và tạo endpoint

1. Trong **Policy**, chọn **Full access** cho bước khởi tạo này.
2. Thêm các tag Project, Environment và Owner trong bảng chuẩn bị.
3. Kiểm tra service, Type, VPC và Route Table, rồi chọn **Create endpoint**.
4. Ghi Endpoint ID dạng `vpce-...` và chờ trạng thái **Available**; làm mới danh sách nếu cần.

Full access ở đây là chính sách của đường truy cập endpoint, **không cấp IAM permission và không làm bucket công khai**. Nhóm dùng lựa chọn này trong giai đoạn dựng hạ tầng để không vô tình chặn các bucket chứa gói cài đặt/agent. Quyền ứng dụng vẫn phải giới hạn bằng IAM Role và chính sách bucket ở phần 5.5.

Nếu siết endpoint policy về duy nhất bucket tài liệu sau này, cần kiểm tra các phụ thuộc cài đặt trước. Không dán policy chứa tên bucket hoặc ARN mẫu chưa thay thế. Cũng chưa thêm bucket policy bắt buộc aws:SourceVpce ở bước này, vì thao tác đó có thể chặn truy cập quản trị qua Console ngoài endpoint.

![](/FCAJ_Worklog/images/005/ep3.png)


![](/FCAJ_Worklog/images/005/ep4.png)


## Bước 4: Kiểm tra tuyến S3 được thêm tự động

Mở **VPC → Route tables → rtb-web-public → Routes**. Đối chiếu ba tuyến:

| Destination | Target | Ý nghĩa |
| --- | --- | --- |
| 10.0.0.0/16 | local | Lưu lượng nội bộ VPC |
| 0.0.0.0/0 | ID của igw-secure-web | Truy cập Internet và dịch vụ khác |
| pl-... của S3 Region hiện tại | ID của s3-gateway-endpoint | Truy cập S3 qua Gateway Endpoint |

AWS tự thêm tuyến prefix list khi liên kết endpoint với Route Table. Không thêm thủ công tuyến 0.0.0.0/0 đến endpoint và không xóa tuyến Internet Gateway. Với cấu hình này, đích S3 khớp prefix list đi qua endpoint; CloudWatch và các đích khác vẫn dùng đường Internet hiện có.

Nếu chưa thấy tuyến S3, mở endpoint → **Route tables**, kiểm tra liên kết; dùng **Actions → Manage route tables** để chọn đúng rtb-web-public và lưu. Không chỉnh sửa trực tiếp tuyến endpoint do AWS quản lý.

![](/FCAJ_Worklog/images/005/ep5.png)


## Bước 5: Kiểm tra firewall và bàn giao

- sg-secure-web giữ outbound cho phép lưu lượng cần thiết theo 5.3.3.
- nacl-web-public có outbound TCP 443 và inbound TCP 1024-65535 cho yêu cầu S3 và phản hồi theo 5.3.4.
- Giữ bucket cùng Region. Ứng dụng dùng S3 SDK/CLI với Region ap-southeast-1 và HTTPS; không cần nhập Endpoint ID làm URL.
- Ghi Endpoint ID, VPC ID, Route Table ID, prefix list ID thực tế và Region vào bảng bàn giao.

NACL không hỗ trợ prefix list làm nguồn/đích. Bộ quy tắc hiện tại cho phép các CIDR IPv4 cần thiết; nếu sau này giới hạn riêng dải S3, phải cập nhật CIDR thực tế và duy trì khi AWS thay đổi.

Chưa kiểm thử upload/download ở bước này vì EC2, bucket và IAM Role chưa hoàn tất. Thực hiện kiểm tra dữ liệu ở **5.5.3** và **5.7.2**. Trạng thái Available chỉ xác nhận endpoint sẵn sàng; không chứng minh ứng dụng đã có quyền S3. Một lần tải thành công từ laptop cũng không chứng minh lưu lượng EC2 đi qua endpoint.

## Lỗi thường gặp

| Hiện tượng | Cách xử lý |
| --- | --- |
| Chỉ thấy Type Interface | Lọc lại Gateway và service com.amazonaws.ap-southeast-1.s3 |
| Không thấy rtb-web-public | Kiểm tra VPC/Region và Route Table ID |
| Không có tuyến pl-... | Kiểm tra Route Table đã liên kết với endpoint và subnet dùng đúng bảng |
| Báo đã có endpoint/tuyến cùng dịch vụ | Kiểm tra endpoint hiện có và dùng lại nếu đúng; không tạo trùng |
| Sau này gọi S3 bị timeout | Kiểm tra DNS, route, Region, outbound 443 và inbound ephemeral của NACL |
| S3 trả AccessDenied | Kiểm tra IAM Role, bucket policy và endpoint policy; không xử lý bằng cách mở bucket public |
| AccessDenied khi tạo endpoint | Gửi action bị từ chối cho quản trị kiểm tra quyền tạo/liên kết và điều kiện policy |


Tiếp tục: [5.4. Triển khai Web Server trên EC2](../../5.4-ec2-web-server/).

## Tài liệu tham khảo

- [Gateway endpoints for Amazon S3](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)
- [Gateway endpoints: routing and security](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html)
