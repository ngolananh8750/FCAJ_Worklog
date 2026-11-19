---
title: "Cấp quyền IAM Role và kiểm tra EC2 đọc, ghi tài liệu trên S3"
date: 2026-10-31
weight: 3
chapter: false
pre: "<b>5.5.3. </b>"
---

## Mục tiêu

Cho ứng dụng trên **workshop-web-01** dùng **WorkshopEC2AppRole** để liệt kê, đọc và ghi tài liệu trong **portal/** của đúng bucket. Kiểm tra cả thao tác thành công và thao tác ngoài quyền bị từ chối, sau đó kiểm tra upload/download qua HTTPS.

Thành viên 3 phối hợp thành viên 2 và người có quyền IAM. Không mặc định IAM user phụ trách S3 được phép tạo/sửa role. Đây là hướng dẫn để thực hiện trên tài khoản nhóm; chưa có thao tác AWS nào được chạy thay bạn.

## Bước 1: Đối chiếu role và bucket

Hoàn thành 5.5.1–5.5.2, có bucket riêng tư và tài liệu mẫu. EC2 đã có ứng dụng từ 5.4.3, HTTPS từ 5.4.4. Region là **ap-southeast-1**.

1. Trong EC2 → Instances → workshop-web-01 → Security, ghi IAM Role đang gắn.
2. Mở IAM → Roles → **WorkshopEC2AppRole** → Trust relationships.
3. Xác nhận trust cho dịch vụ EC2 theo mẫu dưới. Nếu role còn phục vụ hệ thống khác, không thay thế trust tùy tiện.
4. Nếu chưa có role, người quản trị tạo tại IAM → Roles → Create role → AWS service → EC2, đặt tên WorkshopEC2AppRole; sau đó gắn policy ở bước 2.

Tên role khác với instance profile là hợp lệ. Profile đã chuẩn bị là WorkshopEC2AppRole; profile do Console tạo có thể mang tên role. Xác nhận profile chứa đúng role, không tạo trùng chỉ vì tên khác.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```



## Bước 2: Tạo policy tối thiểu và gắn role

Người có quyền IAM mở **IAM → Policies → Create policy → JSON**. Dán JSON dưới, đối chiếu các ARN với bucket của nhóm. Chọn Next, đặt tên **WS-S3-Portal-ReadWrite**, kiểm tra rồi Create policy.

Mở **Roles → WorkshopEC2AppRole → Add permissions → Attach policies**, tìm đúng policy vừa tạo và gắn vào role. Nếu policy cùng tên đã tồn tại, kiểm tra nội dung trước khi dùng lại.

ListBucket áp dụng lên ARN bucket và giới hạn prefix; GetObject/PutObject áp dụng lên ARN object portal/*. Không cần DeleteObject, ListAllMyBuckets hoặc quyền KMS cho SSE-S3.

**Policy Allow này không phải permissions boundary.** Kiểm tra mọi policy khác của role và bucket: một quyền S3 rộng từ nơi khác có thể mở rộng phạm vi. Người quản trị cần loại bỏ quyền S3 dư thừa của role workshop sau khi đối chiếu; giữ các quyền CloudWatch cần thiết. Không tháo policy dùng chung của hệ thống khác.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListPortalOnly",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::secureweb-docs-683699-280958772766",
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "portal/",
            "portal/*"
          ]
        }
      }
    },
    {
      "Sid": "ReadWritePortalObjects",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::secureweb-docs-683699-280958772766/portal/*"
    }
  ]
}
```



## Bước 3: Yêu cầu HTTPS ở bucket

Người được quyền sửa bucket policy mở **S3 → bucket → Permissions → Bucket policy → Edit**. Với bucket chưa có policy, dùng JSON dưới và thay secureweb-docs-683699-280958772766 ở cả hai ARN. Nếu đã có policy, ghép statement Deny vào danh sách Statement hiện có, không ghi đè các điều kiện khác.

Deny này từ chối yêu cầu không dùng TLS của principal không phải AWS service; không cấp quyền public. Giữ Block Public Access bật. Trong mô hình cùng tài khoản, IAM policy ở bước 2 thường đủ để cấp quyền, không cần thêm Allow Principal *.

Không thêm aws:SourceVpce bắt buộc trong bước này. Endpoint đã định tuyến S3; buộc mọi truy cập phải qua endpoint là thay đổi khác có thể chặn quản trị Console.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::secureweb-docs-683699-280958772766",
        "arn:aws:s3:::secureweb-docs-683699-280958772766/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false",
          "aws:PrincipalIsAWSService": "false"
        }
      }
    }
  ]
}
```



## Bước 4: Xác nhận profile trên EC2

Nếu role đúng đã gắn từ 5.4.1, giữ nguyên. Nếu thiếu/sai, người được quyền EC2 và iam:PassRole chọn **EC2 → instance → Actions → Security → Modify IAM role**, chọn profile chứa WorkshopEC2AppRole rồi lưu.

Kiểm tra lại tab Security. Chờ một khoảng ngắn để thay đổi IAM có hiệu lực; không tạo access key để xử lý độ trễ. Không chạy aws configure với thông tin IAM user trên EC2.

Boto3 của ứng dụng cần lấy credentials từ instance profile. Kiểm tra service không đặt AWS_ACCESS_KEY_ID/AWS_SECRET_ACCESS_KEY hoặc profile khác; không in giá trị bí mật vào terminal/báo cáo. IMDSv2 vẫn được bật theo 5.4.1.



## Bước 5: Kiểm thử bằng đúng tài khoản dịch vụ

Dùng file **ec2-s3-check.py** đi kèm bài này. Trên máy cá nhân, đặt file cạnh key pair và chuyển lên EC2:

```bash
scp -i "YOUR_KEY_PAIR.pem" ec2-s3-check.py ec2-user@PUBLIC_IPV4:~/
```

Thay PUBLIC_IPV4, sau đó trên EC2:

```bash
sudo install -m 644 ~/ec2-s3-check.py /opt/document-portal/ec2-s3-check.py
```

Chạy lệnh dưới với user portal trong môi trường sạch. Script chỉ đọc bucket từ config, không in password hash hay credentials. Nó yêu cầu credentials.method là iam-role và Caller ARN chứa đúng WorkshopEC2AppRole; nếu khác thì dừng.

Script tạo **một TXT thử nghiệm mới** có key ngẫu nhiên trong portal/, tải lại, so sánh SHA-256 và kiểm tra AES256. Sau đó thử liệt kê outside/ và xóa **chính object thử nghiệm mới tạo**, mong đợi AccessDenied. Không dùng tài liệu thật cho ca thử xóa: nếu quyền đang quá rộng, thao tác có thể thành công và script sẽ báo FAIL.

STS dùng đường HTTPS Internet hiện có, không qua S3 Gateway Endpoint. Timeout ở STS cần kiểm tra mạng trước khi kết luận IAM lỗi.

```bash
sudo -u portal env -i \
  PATH=/usr/bin:/bin HOME=/nonexistent \
  AWS_SHARED_CREDENTIALS_FILE=/dev/null AWS_CONFIG_FILE=/dev/null \
  /opt/document-portal/.venv/bin/python /opt/document-portal/ec2-s3-check.py
```



## Bước 6: Kiểm tra website và hoàn tất

1. Xác nhận bucket trong /etc/document-portal/config.json đúng tên thực tế. Nếu cần sửa, dùng sudoedit và giữ quyền root:portal, mode 640; không chụp nội dung file.
2. Nếu đã sửa cấu hình, chạy `sudo systemctl restart document-portal`; phiên trong RAM bị xóa, đăng nhập lại.
3. Mở HTTPS theo 5.4.4, đăng nhập tài khoản website.
4. Xác nhận thấy tài liệu mẫu huong-dan.txt/tai-lieu.pdf của 5.5.2.
5. Upload một TXT nhỏ khác; kiểm tra object mới nằm dưới portal/ và có SSE-S3.
6. Download qua website, so sánh SHA-256 với bản gốc theo 5.5.2.
7. Logout, mở lại đường dẫn download: phải chuyển về login.

Thành công từ Console không thay cho kiểm thử EC2. PUT/GET thành công từ EC2 cũng chưa tự chứng minh đường đi endpoint; đối chiếu tuyến pl-... → vpce-... trong rtb-web-public với 5.3.5. Không gọi kết quả đó là bằng chứng ghi nhận lưu lượng endpoint.



## Xử lý lỗi và bàn giao

- **Unexpectedly allowed:** dừng, kiểm tra toàn bộ policy cấp quyền. Không coi việc chỉ gắn policy tối thiểu là đã thu hẹp quyền có sẵn.
- **AccessDenied ở thao tác phải được phép:** đối chiếu tên bucket, ARN, prefix portal/, role thực tế, permissions boundary, SCP và explicit deny ở bucket/endpoint policy.
- **No credentials/không phải iam-role:** kiểm tra profile, IMDSv2 và credential override. Không in metadata credentials.
- **Timeout:** kiểm tra DNS, route, NACL và outbound 443. Môi trường sạch của script không kế thừa proxy tùy chỉnh.
- **503 website dù script PASS:** kiểm tra service đọc đúng config, dùng cùng role và không có credential override.

Chỉ coi ca từ chối PASS khi lỗi là **AccessDenied** của đúng thao tác; timeout/NoSuchBucket không phải bằng chứng giới hạn quyền. Script không kiểm tra mọi khả năng truy cập; đây là các ca đại diện.

Ghi tên/ARN policy và role, bucket, key thử nghiệm và kết quả vào bảng bàn giao. Object thử nghiệm được giữ lại nếu ca xóa bị từ chối đúng; quản trị có quyền xóa sẽ dọn nó ở 5.8. Không cấp DeleteObject cho ứng dụng chỉ để dọn thử nghiệm.



Tiếp tục: [5.6. Giám sát và cảnh báo với CloudWatch](../../5.6-cloudwatch/).
