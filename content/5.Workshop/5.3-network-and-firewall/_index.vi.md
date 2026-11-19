---
title: "Thiết lập mạng và cấu hình Firewall"
date: 2026-10-29
weight: 3
chapter: false
pre: "<b>5.3. </b>"
---

## Tổng quan

Trong phần này, nhóm xây dựng môi trường mạng cho **website đăng nhập để tải lên, lưu trữ và tải xuống tài liệu**. Amazon VPC cung cấp mạng riêng trên AWS; Public Subnet là nơi triển khai EC2 ở phần 5.4. Internet Gateway và Route Table cung cấp đường kết nối Internet, còn Security Group và Network ACL kiểm soát lưu lượng.

**Thành viên 1** thực hiện chính và bàn giao thông tin mạng cho các thành viên còn lại. Cả nhóm sử dụng chung một VPC theo thông số đã thống nhất ở phần 5.2.2.

Trong workshop, “Firewall” chỉ việc kiểm soát lưu lượng bằng **Security Group và Network ACL**; không triển khai thêm dịch vụ AWS Network Firewall. Phần này là trang tổng quan và điều hướng; thao tác trên Console được trình bày ở các mục 5.3.1–5.3.5.

## Mục tiêu

- Tạo VPC và một Public Subnet cho máy chủ web.
- Gắn Internet Gateway vào VPC, tạo Route Table và liên kết với subnet.
- Chuẩn bị Security Group cho EC2, giới hạn SSH và cho phép truy cập web theo yêu cầu.
- Cấu hình Network ACL để minh họa lớp kiểm soát bổ sung ở cấp subnet.
- Tạo S3 Gateway VPC Endpoint để EC2 truy cập S3 trong cùng Region theo tuyến endpoint.
- Ghi lại ID tài nguyên và cấu hình để phục vụ triển khai, kiểm thử ở các phần tiếp theo.

## Điều kiện trước khi thực hiện

1. Hoàn thành phần 5.2, đăng nhập bằng IAM user phụ trách mạng đã được cấp quyền tạo và cấu hình tài nguyên.
2. Chọn **Asia Pacific (Singapore) — ap-southeast-1**.
3. Chuẩn bị IP công cộng của máy quản trị dùng cho SSH, biểu diễn dưới dạng **IP/32**.
4. Kiểm tra bảng bàn giao để tránh tạo trùng tài nguyên. Nếu đã có tài nguyên phù hợp, xác minh cấu hình và sử dụng lại.
5. Thống nhất với thành viên 3 tên bucket dự kiến. Bucket và quyền truy cập dữ liệu được triển khai ở phần 5.5; không cần tạo bucket chỉ để bắt đầu phần mạng.

Nếu gặp lỗi thiếu quyền, ghi lại action và thông báo lỗi để người quản trị điều chỉnh policy. Mở được trang VPC chưa có nghĩa là đã được cấp quyền tạo VPC.

## Thông số triển khai

| Thành phần | Tên hoặc giá trị |
| --- | --- |
| Region | ap-southeast-1 |
| VPC | vpc-secure-web — 10.0.0.0/16 |
| Public Subnet | subnet-web-public — 10.0.0.0/24 |
| Availability Zone | ap-southeast-1a nếu khả dụng; ghi lại AZ thực tế |
| DNS resolution và DNS hostnames | Bật |
| Internet Gateway | igw-secure-web |
| Route Table | rtb-web-public |
| Tuyến Internet | 0.0.0.0/0 → igw-secure-web |
| Security Group | sg-secure-web |
| Network ACL | nacl-web-public |
| S3 Gateway VPC Endpoint | s3-gateway-endpoint |
| Dịch vụ endpoint | com.amazonaws.ap-southeast-1.s3 |
| Tag dùng chung | Project=SecureWebWorkshop |

Tên tài nguyên không thay thế ID do AWS cấp. Khi cấu hình liên kết, luôn kiểm tra đúng VPC ID, Subnet ID và Route Table ID.

## Nguyên tắc kết nối và bảo mật

| Thành phần | Cách áp dụng trong workshop |
| --- | --- |
| Public Subnet | Có tuyến đến Internet Gateway. EC2 còn cần Public IPv4, quy tắc mạng phù hợp và dịch vụ đang chạy để nhận kết nối Internet qua IPv4. |
| Security Group | Gắn với giao diện mạng của EC2 khi khởi tạo ở phần 5.4; chỉ có quy tắc cho phép và tự theo dõi trạng thái kết nối. |
| Network ACL | Liên kết với subnet; hỗ trợ allow/deny, xét thứ tự số quy tắc và cần cho phép lưu lượng phản hồi vì không theo dõi trạng thái. |
| S3 Gateway VPC Endpoint | Liên kết với Route Table của Public Subnet; không tự cấp quyền đọc/ghi S3 và không thay thế IAM Role hoặc bucket policy. |

**Yêu cầu lưu lượng dự kiến:**

- **SSH — TCP 22:** chỉ từ IP quản trị /32. Nếu có nhiều người quản trị, dùng từng nguồn /32 riêng.
- **HTTPS — TCP 443:** cho phép người dùng Internet truy cập website; cấu hình HTTPS thực hiện ở phần 5.4.4.
- **HTTP — TCP 80:** thử trang Nginx và chuyển hướng sang HTTPS.
- **Lưu lượng đi ra:** đáp ứng cài đặt phần mềm, truy cập S3 và gửi nhật ký CloudWatch; quy tắc cụ thể được trình bày trong các mục cấu hình.
- **Lưu lượng phản hồi:** phải được tính đến trong Network ACL, bao gồm các cổng tạm thời cần thiết.

Không mở SSH cho 0.0.0.0/0. Không sao chép nguyên quy tắc Security Group sang Network ACL mà bỏ qua chiều phản hồi. Khi dùng Network ACL tùy chỉnh, chuẩn bị đủ quy tắc trước khi liên kết với subnet để tránh làm mất kết nối.

S3 là dịch vụ nằm ngoài subnet. Trong mô hình này, lưu lượng đến S3 cùng Region sử dụng gateway endpoint; lưu lượng Internet và kết nối đến CloudWatch vẫn sử dụng đường Internet Gateway theo cấu hình cho phép. Không tạo NAT Gateway hoặc interface endpoint trong phạm vi phần này.

## Trình tự thực hiện

| Mục | Công việc | Kết quả bàn giao |
| --- | --- | --- |
| 5.3.1 | Tạo VPC và Public Subnet | VPC ID, Subnet ID, CIDR và AZ |
| 5.3.2 | Tạo Internet Gateway và Route Table | Internet Gateway ID, Route Table ID, tuyến Internet và liên kết subnet |
| 5.3.3 | Cấu hình Security Group | Security Group ID và các quy tắc đã thống nhất |
| 5.3.4 | Cấu hình Network ACL | Network ACL ID, quy tắc hai chiều và liên kết subnet |
| 5.3.5 | Tạo S3 Gateway VPC Endpoint | VPC Endpoint ID và Route Table được liên kết |

Thực hiện theo thứ tự trên. EC2 được tạo ở phần 5.4; kiểm thử đọc/ghi S3 cần bucket và IAM Role ở phần 5.5. Chưa có EC2 hoặc bucket thì chỉ ghi nhận kết quả kiểm tra cấu hình mạng, không ghi rằng website hoặc truy cập tài liệu đã hoạt động.

## Kết quả cần đạt

- VPC và subnet có đúng CIDR, Region và thiết lập DNS.
- Internet Gateway đã gắn với VPC; Route Table có tuyến Internet và liên kết đúng subnet.
- Security Group đã sẵn sàng để thành viên 2 gắn vào EC2.
- Network ACL có quy tắc phù hợp và liên kết đúng subnet.
- S3 Gateway VPC Endpoint ở trạng thái Available và liên kết với đúng Route Table.
- Bảng bàn giao được cập nhật ID thực tế; thông tin đã được chia sẻ với thành viên 2, 3 và 4.

Ảnh thao tác và kết quả chi tiết được chèn ở từng mục con. Kiểm thử kết nối và allow/deny đầy đủ được thực hiện sau khi có các tài nguyên phụ thuộc, tại phần 5.7.

## Nội dung

1. [5.3.1. Tạo VPC và Public Subnet](./5.3.1-vpc-and-subnet/)
2. [5.3.2. Tạo Internet Gateway và Route Table](./5.3.2-gateway-and-route-table/)
3. [5.3.3. Cấu hình Security Group](./5.3.3-security-group/)
4. [5.3.4. Cấu hình Network ACL](./5.3.4-network-acl/)
5. [5.3.5. Tạo S3 Gateway VPC Endpoint](./5.3.5-s3-endpoint/)
- [5.3.6. Flow Logs và kiểm thử cổng 80/81](./5.3.6-flow-logs/)

## Tài liệu tham khảo

- [Kiểm soát mạng bằng Security Group và Network ACL](https://docs.aws.amazon.com/vpc/latest/userguide/infrastructure-security.html)
- [Thiết lập truy cập Internet cho subnet](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-igw.html)
- [S3 Gateway VPC Endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)
