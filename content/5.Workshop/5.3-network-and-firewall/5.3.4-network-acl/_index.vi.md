---
title: "Cấu hình Network ACL"
date: 2026-10-30
weight: 4
chapter: false
pre: "<b>5.3.4. </b>"
---



## Mục tiêu

Tạo **nacl-web-public** cho **subnet-web-public**, bổ sung lớp lọc lưu lượng ở cấp subnet cho website đăng nhập, tải lên và tải xuống tài liệu. Thành viên 1 thực hiện phần này.

Network ACL (NACL) không có trạng thái: phải cho phép cả yêu cầu và phản hồi. Quy tắc được xét từ số nhỏ đến lớn, dừng tại quy tắc khớp đầu tiên. NACL hỗ trợ ALLOW và DENY; quy tắc `*` cuối bảng từ chối lưu lượng chưa khớp.

## Chuẩn bị

- Hoàn thành [5.3.3. Cấu hình Security Group](../5.3.3-security-group/).
- Region: **Singapore — ap-southeast-1**.
- VPC: **vpc-secure-web — 10.0.0.0/16**; subnet: **subnet-web-public — 10.0.0.0/24**.
- Có quyền tạo NACL, quản lý quy tắc, thay đổi liên kết subnet và gắn tag.
- Ghi lại NACL ID hiện tại của subnet tại **VPC → Subnets → subnet-web-public → Network ACL** để có thể khôi phục.
- Chuẩn bị Public IPv4/32 của quản trị viên, giống nguồn SSH trong sg-secure-web.

**Chỉ liên kết NACL mới sau khi đã cấu hình đủ hai chiều.** NACL tùy chỉnh mới tạo chặn toàn bộ lưu lượng; gắn ngay có thể làm mất kết nối của tài nguyên trong subnet.

## Bước 1: Tạo Network ACL

1. Mở **VPC → Network ACLs → Create network ACL**.
2. Nhập **Name: nacl-web-public**, chọn VPC ID của **vpc-secure-web**.
3. Thêm tags: **Project=SecureWebWorkshop**.
4. Chọn **Create network ACL**, ghi lại ID dạng `acl-...`.
5. Chưa liên kết subnet. Nếu NACL đã tồn tại, đối chiếu ID/VPC và kiểm tra trước khi dùng lại.

![](/FCAJ_Worklog/images/005/nacl1.png)


## Bước 2: Cấu hình inbound rules

Chọn **nacl-web-public → Inbound rules → Edit inbound rules → Add new rule**, điền từng dòng rồi **Save changes**.

| Rule number | Type | Protocol | Port range | Source | Allow/Deny |
| --- | --- | --- | --- | --- | --- |
| 100 | SSH | TCP | 22 | ADMIN_PUBLIC_IP/32 | ALLOW |
| 120 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 110| HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 130 | Custom TCP | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| * | All traffic | All | All | 0.0.0.0/0 | DENY |

Thay **ADMIN_PUBLIC_IP/32** bằng địa chỉ Internet thực tế của máy SSH kèm `/32`. Nếu có quản trị viên khác, thêm quy tắc SSH riêng số 101, 102… với nguồn /32 tương ứng. Không mở cổng 22 cho toàn Internet.

Rule 110 mở cổng 80 cho trang Nginx ban đầu và chuyển hướng HTTPS ở 5.4.4.

Rule 130 phục vụ phản hồi về các cổng tạm thời khi EC2 chủ động kết nối ra ngoài, ví dụ gọi S3/CloudWatch. Dải 1024-65535 được chọn để phù hợp nhiều hệ điều hành trong bài lab. NACL không nhận biết đây có phải phản hồi hay không; quy tắc cũng cho qua gói TCP mới đến dải này. **sg-secure-web vẫn phải chặn kết nối inbound mới tới các cổng backend.**

Giữ nguyên rule `*`; không thêm All traffic ALLOW trước các quy tắc trên. Không cần cấu hình IPv6 trong workshop này.

![](/FCAJ_Worklog/images/005/nacl2.png)


## Bước 3: Cấu hình outbound rules

Chọn **Outbound rules → Edit outbound rules → Add new rule**, nhập bảng dưới rồi **Save changes**.

| Rule number | Type | Protocol | Port range | Destination | Allow/Deny |
| --- | --- | --- | --- | --- | --- |
| 100 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 110 | Custom TCP | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| * | All traffic | All | All | 0.0.0.0/0 | DENY |  

Outbound 80/443 phục vụ tải gói cài đặt và gọi dịch vụ qua HTTP/HTTPS; outbound 80 không làm website nhận kết nối HTTP. Rule 110 cho phép máy chủ gửi phản hồi SSH/HTTPS về cổng tạm thời của máy khách. Không dùng outbound 22 để thay cho phản hồi SSH.



![](/FCAJ_Worklog/images/005/nacl3.png)


## Bước 4: Liên kết với Public Subnet

1. Đối chiếu lại cả hai bảng và ID NACL cũ đã ghi.
2. Mở **nacl-web-public → Subnet associations → Edit subnet associations** (nút có thể hiển thị **Edit**).
3. Chọn đúng **subnet-web-public**, đối chiếu Subnet ID và CIDR 10.0.0.0/24.
4. Lưu bằng **Save**. Chỉ thay đổi liên kết subnet của workshop.
5. Mở **Subnets → subnet-web-public → Network ACL**, xác nhận ID của nacl-web-public.

Một subnet chỉ dùng một NACL tại một thời điểm; liên kết mới thay thế liên kết cũ. Không sửa NACL mặc định dùng chung cho các subnet khác.

![](/FCAJ_Worklog/images/005/nacl4.png)


## Bước 5: Kiểm tra và bàn giao

- Xác nhận đúng Region, VPC, subnet và NACL ID.
- SSH chỉ cho phép các nguồn quản trị /32; HTTPS cho phép người dùng Internet.
- Có TCP 1024-65535 ở cả hai chiều; không có rule ALLOW toàn bộ hoặc DENY số nhỏ ngoài dự kiến.
- Cập nhật bảng bàn giao NACL ID mới, ID cũ để khôi phục và danh sách nguồn SSH.
- Khi đổi IP quản trị, cập nhật cả NACL và Security Group.

Đây là kiểm tra cấu hình; chưa kiểm thử kết nối khi chưa có EC2. Sau khi triển khai ở phần 5.4, kiểm thử allow/deny ở 5.7.1. Để minh họa DENY sau này, có thể thêm tạm inbound rule **90, HTTPS/TCP 443, nguồn IP máy kiểm thử/32, DENY**, mở kết nối HTTPS mới và ghi kết quả; sau đó xóa rule 90 và xác nhận truy cập phục hồi. Không thêm quy tắc thử nghiệm này trong bước cấu hình hiện tại.

## Lỗi thường gặp và khôi phục

| Hiện tượng | Kiểm tra/xử lý |
| --- | --- |
| Mất kết nối sau khi liên kết | Kiểm tra quy tắc hai chiều; nếu cần khôi phục nhanh, vào Subnets → Network ACL → Edit network ACL association, chọn NACL ID cũ và lưu |
| SSH/HTTPS timeout dù SG đúng | Kiểm tra rule DENY số nhỏ, nguồn /32 và cổng tạm thời chiều phản hồi |
| EC2 không tải gói/gọi S3 hoặc CloudWatch | Kiểm tra outbound 80/443, inbound ephemeral, route và DNS; AccessDenied của S3 cần kiểm tra IAM/policy |
| Không thấy subnet khi liên kết | Kiểm tra VPC và Region của cả hai tài nguyên |
| Không tạo/sửa được rule | Kiểm tra số rule trùng và quyền IAM; gửi action bị từ chối cho quản trị viên |



Tiếp tục: [5.3.5. Tạo S3 Gateway VPC Endpoint](../5.3.5-s3-endpoint/).

## Tài liệu tham khảo

- [Create a network ACL](https://docs.aws.amazon.com/vpc/latest/userguide/create-network-acl.html)
- [Custom network ACLs and ephemeral ports](https://docs.aws.amazon.com/vpc/latest/userguide/custom-network-acl.html)
- [Control subnet traffic with network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
