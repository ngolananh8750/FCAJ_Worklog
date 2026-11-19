---
title: "Dọn dẹp tài nguyên"
date: 2026-11-02
weight: 8
chapter: false
pre: "<b>5.8. </b>"
---

## Mục tiêu

Dọn các tài nguyên chỉ phục vụ workshop sau khi hoàn tất kiểm thử, tránh để tài nguyên tiếp tục phát sinh chi phí. Giữ lại báo cáo, mã nguồn và bằng chứng thực hành ngoài môi trường sẽ xóa.

**Phần này là hướng dẫn để nhóm thực hiện trên AWS, không phải xác nhận tài nguyên đã được xóa.** Terminate EC2, Empty bucket và Delete log group có thể làm mất dữ liệu không khôi phục được. Chỉ thao tác trên tài nguyên lab đã đối chiếu đúng ID và không còn dùng chung.

## Chuẩn bị và phân công

1. Hoàn thành [5.7. Kiểm thử hệ thống](../5.7-testing/), lưu kết quả và ảnh minh chứng về máy hoặc nơi lưu trữ khác ngoài bucket/EC2 của lab.
2. Tải về tài liệu cần giữ, mã nguồn và log cần cho báo cáo. Không đưa private key, mật khẩu/hash, cookie hoặc access key vào báo cáo công khai.
3. Chọn đúng AWS account và **Singapore — ap-southeast-1**. IAM là dịch vụ toàn cục; khi xử lý IAM vẫn phải đối chiếu đúng account và người sử dụng tài nguyên.
4. Nhóm thống nhất kết thúc lab; ghi danh sách ID thực tế và người thực hiện. Nếu tài nguyên dùng chung, để lại và ghi lý do, không xóa theo Name tag đơn thuần.
5. Tài khoản thực hiện cần quyền xóa tương ứng. Khi gặp AccessDenied, nhờ quản trị được phân quyền xử lý; không cấp quyền quản trị toàn bộ hoặc tăng quyền xóa cho role ứng dụng.

| Phụ trách | Tài nguyên |
| --- | --- |
| Thành viên 2 | EC2, EBS, key pair, địa chỉ IP cấp riêng nếu có |
| Thành viên 3 | Object và bucket S3 |
| Thành viên 4 | Alarm, log group, SNS nếu có |
| Thành viên 1 | Endpoint, subnet, route table, NACL, SG, IGW, VPC |
| Quản trị IAM | Role, instance profile và policy chỉ dùng cho lab |

Thứ tự khuyến nghị: lưu bằng chứng → ngừng ghi dữ liệu → xóa alarm → terminate EC2 và kiểm tra phần còn lại → xóa S3/logs/SNS → dọn mạng → dọn IAM → kiểm tra tài nguyên và chi phí.

## Bước 1: Ngừng ứng dụng và gửi log

Sau khi lưu bằng chứng cuối cùng, thành viên 2 SSH vào EC2 và chạy:

```bash
sudo systemctl stop document-portal
sudo systemctl stop nginx
sudo systemctl stop amazon-cloudwatch-agent
```

Nếu tải CPU thử vẫn đang active, dừng riêng dịch vụ đó:

```bash
sudo systemctl stop workshop-cpu-test
```

Nếu unit thử không tồn tại hoặc đã dừng, ghi nhận rồi tiếp tục. Website ngừng phục vụ từ bước này. Không cần gỡ Python, Nginx hay từng file nếu sẽ terminate máy ngay sau đó. Nếu không thể SSH nhưng đã lưu đủ dữ liệu, có thể terminate đúng instance ở bước 3 để ngừng nguồn ghi; không xóa bucket/log group khi nguồn ghi còn hoạt động.

## Bước 2: Xóa CloudWatch alarm

1. Mở **CloudWatch → Alarms → All alarms**.
2. Tìm **workshop-web-01-cpu-high**, kiểm tra Instance ID trong metric và lưu History nếu chưa lưu.
3. Chọn đúng alarm → **Actions → Delete**, xác nhận theo Console.
4. Làm mới và kiểm tra alarm không còn trong danh sách.
5. Nếu nhóm tự tạo thêm alarm chỉ dùng cho lab, kiểm tra và xử lý từng alarm đó; không xóa alarm dùng chung.

Xóa alarm không xóa EC2 hoặc các datapoint CPU lịch sử. Không cần tìm cách xóa thủ công metric EC2 chỉ để hoàn tất cleanup.



## Bước 3: Terminate EC2 và kiểm tra EBS/IP/key pair

1. Mở **EC2 → Instances**, chọn **workshop-web-01** bằng Instance ID đã lưu.
2. Trước khi xóa, mở tab **Storage**, ghi Volume ID và kiểm tra **Delete on termination** cho từng EBS volume. Bài lab đặt root volume được xóa khi terminate, nhưng phải kiểm tra cấu hình thực tế.
3. Chọn **Instance state → Terminate (delete) instance**. Đọc tên/ID, xác nhận và chờ trạng thái **Terminated**.
4. Nếu bật termination protection, chỉ tắt bảo vệ của đúng instance lab đã xác nhận cần xóa rồi thực hiện lại.
5. Mở **Elastic Block Store → Volumes**, tìm các Volume ID đã ghi. Volume không tự xóa chỉ được xóa sau khi xác nhận không cần dữ liệu và không dùng nơi khác; chọn volume **Available → Actions → Delete volume**.
6. Nếu đã tự tạo snapshot/AMI ngoài hướng dẫn chính, kiểm kê riêng: chỉ deregister AMI lab không dùng nữa và xóa snapshot không còn cần cho AMI/backup khác. Không xóa snapshot chỉ vì không có tag.
7. Public IPv4 tự cấp của instance được giải phóng khi terminate. Bài lab không yêu cầu Elastic IP; nếu nhóm có tự cấp, vào **Elastic IP addresses**, đối chiếu Allocation ID, disassociate nếu còn gắn, rồi **Release Elastic IP addresses** khi không còn dùng. Không nhầm địa chỉ tự cấp với Elastic IP.
8. Trong **Network & Security → Key Pairs**, chỉ xóa **workshop-web-key** nếu không dùng cho instance khác. Xóa bản private key local chỉ khi không còn cần; xóa key pair trên Console không tự thu hồi key đã cài trên máy khác.

**Stop không thay thế Terminate** trong cleanup cuối cùng. EBS/snapshot hoặc tài nguyên giữ lại vẫn có thể phát sinh phí. Instance Terminated có thể còn xuất hiện một thời gian trên Console.



## Bước 4: Làm rỗng và xóa bucket S3

- Ở bucket đã bật Versioning, chọn **Empty** với quyền xóa phiên bản. Kiểm tra **Show versions** để xác nhận mọi version và delete marker đều đã dọn. Sau đó xóa bucket. Lifecycle 30 ngày không thay thế việc Empty.

Thành viên 3 hoặc người có quyền xóa dữ liệu thực hiện. Không cấp DeleteObject cho WorkshopEC2AppRole chỉ để cleanup.

1. Mở **S3 → General purpose buckets**, chọn đúng tên bucket thực tế của workshop; đối chiếu account, Region và dữ liệu đã sao lưu.
2. Đảm bảo EC2/ứng dụng không còn ghi vào bucket.
3. Chọn bucket → **Empty**. Đọc phạm vi xóa, nhập chính xác chuỗi xác nhận Console yêu cầu và thực hiện.
4. Chờ tác vụ hoàn thành, kiểm tra kết quả từng lỗi nếu có. Object mẫu, file upload và `portal/ec2-check-...txt` đều nằm trong phạm vi bucket lab này.
5. Nếu từng bật/suspend Versioning, phải xóa cả object versions và delete markers. Console Empty hỗ trợ làm rỗng các phiên bản; chỉ xóa object hiện tại thông thường có thể chưa làm bucket rỗng.
6. Nếu đã dùng multipart upload ngoài bài lab, kiểm tra các upload chưa hoàn tất và abort chúng bằng công cụ S3 được phân quyền trước khi kết thúc cleanup. Không giả định chúng là object hiện trong danh sách.
7. Quay lại danh sách bucket → chọn bucket → **Delete**, nhập tên bucket theo yêu cầu và xác nhận.
8. Làm mới, kiểm tra bucket không còn trong danh sách.

Nếu gặp Object Lock/retention/legal hold, không tự bỏ bảo vệ; đây là cấu hình ngoài lab, cần người quản lý dữ liệu xử lý. Với AccessDenied, kiểm tra quyền/bucket policy/SCP; giữ Block Public Access, không công khai bucket để xóa. Bucket đã xóa không còn giữ tên cho bạn; gỡ các cấu hình hoặc liên kết ứng dụng còn tham chiếu tên đó nếu chúng được dùng ở nơi khác.


## Bước 5: Xóa log group và SNS tùy chọn

- Xóa cấu hình `flow-web-public` ở subnet **trước** khi xóa log group `/workshop/vpc/flow-logs`. Đợi ghi log kết thúc và lưu bản cần giữ.

- Xóa dashboard `workshop-web-01`. Sau khi xóa alarm, xóa subscription và topic `workshop-alerts` nếu chỉ phục vụ lab.



1. Xác nhận Agent đã dừng hoặc EC2 đã Terminated để tránh tiếp tục gửi log/tạo lại tài nguyên.
2. Vào **CloudWatch → Logs → Log groups**, chọn đúng **/workshop/webserver**.
3. Sau khi lưu log cần giữ, chọn **Actions → Delete log group(s)**, xác nhận và kiểm tra không còn group. Xóa group xóa các stream và sự kiện trong đó; retention 7 ngày không thay thế bước xóa group cuối lab.
4. Nếu từng tạo dashboard hoặc log group bổ sung chỉ dùng cho workshop, kiểm kê và xóa đúng các tài nguyên đó. Không mặc định có VPC Flow Logs nếu nhóm chưa triển khai.
5. Nếu dùng SNS: mở **SNS → Topics → workshop-alerts**, kiểm tra ARN/subscriptions và xác nhận topic không phục vụ hệ thống khác. Xóa subscription email lab đã xác nhận nếu giao diện cho phép, rồi xóa topic. Kiểm tra lại danh sách; subscription Pending confirmation có cách xử lý khác trên Console, không tạo lại topic chỉ để xử lý mục hiển thị còn lưu.
6. Nếu không triển khai SNS, ghi **Không áp dụng**.


## Bước 6: Xóa S3 Gateway Endpoint

1. Mở **VPC → Endpoints**, chọn **s3-gateway-endpoint** theo Endpoint ID và VPC ID đã lưu.
2. Chọn **Actions → Delete VPC endpoints**, xác nhận.
3. Chờ xóa hoàn tất. Kiểm tra route `pl-... → vpce-...` do endpoint quản lý đã được gỡ khỏi route table liên quan.
4. Không xóa AWS-managed prefix list của S3 và không sửa thủ công route endpoint để thay bước xóa endpoint.



## Bước 7: Dọn mạng và xóa VPC

Chỉ thực hiện sau khi EC2 và endpoint không còn dùng mạng. Đối chiếu **VPC ID**, không chọn default VPC.

1. **Network interfaces:** kiểm tra interface trong vpc-secure-web. Interface chính của EC2 thường tự được xóa sau terminate; chờ hoàn tất. Nếu còn interface của dịch vụ khác, xác định và xử lý tài nguyên sở hữu trước, không force-delete interface không rõ nguồn gốc.
2. **Subnets:** chọn **subnet-web-public**, kiểm tra không còn tài nguyên sử dụng, chọn **Actions → Delete subnet** và xác nhận.
3. **Route tables:** chọn **rtb-web-public**. Khi không còn subnet association và nó không phải main route table, chọn Delete route table. Nếu nó là main, để VPC deletion xử lý main table thay vì xóa nhầm bảng khác.
4. **Network ACLs:** xóa **nacl-web-public** khi không còn subnet association. Không xóa default NACL thủ công.
5. **Security groups:** xóa **sg-secure-web** khi không gắn interface và không bị nhóm khác tham chiếu. Nếu có tham chiếu, kiểm tra nhóm sở hữu; chỉ gỡ tham chiếu thuộc lab. Không xóa default SG thủ công.
6. **Internet gateways:** chọn **igw-secure-web → Actions → Detach from VPC**, xác nhận đúng vpc-secure-web, sau đó **Delete internet gateway**.
7. **Your VPCs:** chọn **vpc-secure-web → Actions → Delete VPC**. Đọc danh sách phụ thuộc Console hiển thị; nếu có tài nguyên không thuộc phạm vi, dừng để kiểm tra. Xác nhận xóa đúng VPC.
8. Làm mới và tìm theo VPC ID để xác nhận không còn VPC.

Nếu gặp DependencyViolation, đọc tài nguyên đang chặn, kiểm tra interface, subnet association hoặc dịch vụ sở hữu. Không giải quyết bằng cách xóa hàng loạt tài nguyên trong account. Default SG/NACL/main route table thuộc VPC sẽ được xử lý khi xóa VPC; default VPC của account không thuộc phạm vi.



## Bước 8: Dọn IAM dành riêng cho lab

- Quản trị IAM gỡ role giao Flow Logs sau khi xác nhận không còn cấu hình nào tham chiếu. Chỉ thu hồi các IAM user và role thành viên nếu chúng được tạo riêng cho lab và nhóm không còn sử dụng.

Người quản trị IAM thực hiện sau khi EC2 đã terminate. Quyền IAM không có Region nhưng cần đúng account.

1. Kiểm tra **WorkshopEC2AppRole** và instance profile có đang dùng bởi EC2, launch template hoặc hệ thống khác không. Nếu còn dùng, giữ lại và ghi lý do.
2. Ghi tên instance profile thực tế: có thể là **WorkshopEC2AppRole** hoặc trùng tên role nếu được tạo bằng Console. Không đoán tên khi xóa.
3. Nếu chỉ dành cho lab, tháo các policy và xóa role bằng **IAM → Roles → role → Delete**. Kiểm tra instance profile liên quan: Console có thể xóa profile cùng tên; profile có tên khác cần được tháo role và xóa riêng.
4. Khi cần xử lý profile bằng CLI, người quản trị dùng môi trường CLI đã xác thực của mình, không cài access key lên EC2. Lệnh dưới chỉ là mẫu; thay tên thực tế, chỉ chạy nếu role còn nằm trong profile:

```bash
aws iam remove-role-from-instance-profile \
  --instance-profile-name ACTUAL_PROFILE_NAME \
  --role-name WorkshopEC2AppRole
aws iam delete-instance-profile \
  --instance-profile-name ACTUAL_PROFILE_NAME
```

5. Trong **IAM → Policies**, xóa customer-managed policy **WS-S3-Portal-ReadWrite** và **WS-CloudWatch-PortalLogs** nếu không còn được gắn với identity khác. Không xóa AWS-managed policy.
6. Tài khoản IAM của bốn thành viên không mặc định bị xóa theo bước này. Nếu là tài khoản dùng chung/học tập tiếp, giữ lại và chỉ gỡ quyền lab không cần thiết. Chỉ quản trị vòng đời tài khoản riêng nếu nhóm đã xác nhận tài khoản được tạo duy nhất cho workshop.



## Bước 9: Kiểm tra tài nguyên còn sót và chi phí

1. Kiểm tra lại trong Region triển khai: EC2, EBS volumes/snapshots, Elastic IP nếu có, S3, CloudWatch alarms/log groups, SNS và VPC endpoints.
2. Nếu từng triển khai nhầm sang Region khác, kiểm tra các Region đó. IAM kiểm tra ở cấp account.
3. Nếu nhóm đã tự tạo NAT Gateway, load balancer, interface endpoint hoặc tài nguyên bổ sung ngoài hướng dẫn, đưa chúng vào danh sách kiểm kê riêng; chỉ xóa sau khi xác nhận chủ sở hữu và phụ thuộc.
4. Người có quyền Billing mở **Billing and Cost Management → Bills** hoặc Cost Explorer để đối chiếu chi phí dịch vụ. Dữ liệu chi phí có độ trễ, không dùng số tiền chưa thay đổi ngay làm bằng chứng xóa thất bại hoặc tài nguyên vẫn chạy.
5. Không tuyên bố hóa đơn bằng 0: chi phí đã phát sinh trước cleanup vẫn phải thanh toán, và tài nguyên giữ lại có thể tiếp tục tính phí.





## Tài liệu tham khảo

- [Terminate EC2 instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/terminating-instances.html)
- [Empty an S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/empty-bucket.html)
- [Delete a VPC](https://docs.aws.amazon.com/vpc/latest/userguide/delete-vpc.html)
