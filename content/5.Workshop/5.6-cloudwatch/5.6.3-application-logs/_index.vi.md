---
title: "Thu thập và kiểm tra nhật ký ứng dụng"
date: 2026-10-31
weight: 3
chapter: false
pre: "<b>5.6.3. </b>"
---

## Mục tiêu

Thành viên 4 phối hợp với thành viên 2 đưa log của website vào **CloudWatch Logs**, kiểm tra sự kiện đăng nhập, tải lên, tải xuống và đặt retention **7 ngày**. Phần này tiếp tục dùng EC2 **workshop-web-01**, Region **ap-southeast-1**, IAM Role **WorkshopEC2AppRole**.

Luồng thực hiện: dịch vụ document-portal ghi file `/var/log/document-portal/application.log`; CloudWatch Agent đọc file và gửi đến log group `/workshop/webserver`, stream `{instance_id}/application`.

## Chuẩn bị

- Hoàn thành 5.4.3–5.5.3; website HTTPS và thao tác S3 đang hoạt động. Đã dừng tải CPU ở 5.6.2.
- Dùng Amazon Linux 2023, dịch vụ `document-portal.service`, user `portal` như hướng dẫn trước; thành viên 2 có SSH và sudo.
- Người thao tác Console cần quyền tạo log group, đặt retention và đọc log. Người quản lý IAM cần quyền tạo/gắn policy vào role EC2. Quyền chỉ đọc của thành viên 4 không tự cấp quyền triển khai.
- EC2 có DNS, outbound TCP 443 và route Internet qua IGW/Public IPv4. S3 Gateway Endpoint không chuyển tiếp lưu lượng CloudWatch Logs. Không mở thêm cổng inbound.
- Cấu hình dưới dành cho máy lab chưa có cấu hình Agent khác. Nếu đã có, sao lưu file và hợp nhất phần thu thập log; không dùng fetch-config để vô tình thay cấu hình đang phục vụ hệ thống khác.

## Bước 1: Tạo log group và đặt retention

1. Mở **CloudWatch**, chọn **Singapore — ap-southeast-1**.
2. Chọn **Logs → Log groups → Create log group**.
3. Nhập `/workshop/webserver`, chọn log class **Standard**. Dùng mã hóa mặc định của CloudWatch Logs cho bài lab, không chọn customer managed KMS key.
4. Đặt **Retention = 1 week (7 days)** nếu màn hình tạo có trường này. Nếu không, tạo group rồi mở **Edit retention setting** để chọn 7 ngày và lưu.
5. Nếu group đã tồn tại, dùng lại sau khi kiểm tra chủ sở hữu và cấu hình. Giảm retention có thể làm hết hạn log cũ; chỉ áp dụng cho log lab đã thống nhất.
6. Kiểm tra Region, tên group và retention sau khi lưu.



## Bước 2: Cấp quyền ghi log cho EC2

1. Người được phân quyền mở **IAM → Policies → Create policy → JSON**.
2. Dán policy dưới, thay **ACCOUNT_ID** ở cả hai ARN bằng AWS Account ID 12 chữ số của nhóm.
3. Kiểm tra policy, đặt tên **WS-CloudWatch-PortalLogs**, rồi tạo.
4. Mở **IAM → Roles → WorkshopEC2AppRole → Add permissions → Attach policies**, gắn policy vừa tạo.
5. Trong EC2 → workshop-web-01 → Security, xác nhận máy đang gắn role này. Giữ quyền S3 của 5.5.3 và trust policy cho EC2.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "WriteWorkshopStreams",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:ap-southeast-1:ACCOUNT_ID:log-group:/workshop/webserver:log-stream:*"
    },
    {
      "Sid": "DescribeWorkshopStreams",
      "Effect": "Allow",
      "Action": "logs:DescribeLogStreams",
      "Resource": "arn:aws:logs:ap-southeast-1:ACCOUNT_ID:log-group:/workshop/webserver:*"
    }
  ]
}
```

Policy cho phép tạo stream/ghi sự kiện trong đúng group. Log group được người quản trị tạo trước; Agent không được cấp quyền tạo group, xóa log hay sửa retention. Không đặt `retention_in_days` trong cấu hình Agent bên dưới vì retention đã được quản lý trên Console. Không dùng access key hoặc chạy aws configure trên EC2.



## Bước 3: Cài công cụ và chuyển đầu ra ứng dụng sang file

SSH vào EC2 như phần 5.4.1. Chạy trên EC2:

```bash
sudo dnf install -y amazon-cloudwatch-agent logrotate
rpm -q amazon-cloudwatch-agent logrotate
```

Kiểm tra cấu hình dịch vụ và log hiện tại:

```bash
sudo systemctl cat document-portal
sudo journalctl -u document-portal -n 30 --no-pager
```

Dịch vụ mẫu hiện dùng StandardOutput/StandardError=journal. Tạo drop-in riêng bên dưới để chuyển cả hai sang file. Nếu file `20-file-logging.conf` đã tồn tại, sao lưu và đối chiếu nội dung trước khi thay. Không sửa file gốc của dịch vụ.

**Restart sẽ hủy phiên đăng nhập đang lưu trong bộ nhớ; thông báo cho nhóm và đăng nhập lại sau thao tác.**

```bash
sudo install -d -o root -g portal -m 0750 /var/log/document-portal
sudo touch /var/log/document-portal/application.log
sudo chown root:portal /var/log/document-portal/application.log
sudo chmod 0640 /var/log/document-portal/application.log
sudo install -d -m 0755 /etc/systemd/system/document-portal.service.d
sudo tee /etc/systemd/system/document-portal.service.d/20-file-logging.conf > /dev/null <<'CONF'
[Service]
StandardOutput=append:/var/log/document-portal/application.log
StandardError=append:/var/log/document-portal/application.log
CONF
sudo systemctl daemon-reload
sudo systemctl restart document-portal
sudo systemctl status document-portal --no-pager
sudo tail -n 20 /var/log/document-portal/application.log
```

Systemd mở file đầu ra và truyền file descriptor cho tiến trình. Giữ nguyên User=portal, ProtectSystem=strict và các tùy chọn bảo vệ của dịch vụ; không dùng chmod 777. Những thông báo ứng dụng mới sẽ nằm trong file, còn thông báo vòng đời dịch vụ vẫn xem được bằng journalctl. Không tự động sao chép log cũ sang CloudWatch.

Nếu dịch vụ không active, xem journalctl để xử lý trước khi tiếp tục. Muốn hoàn tác thay đổi này, chỉ xóa drop-in `20-file-logging.conf` vừa tạo, daemon-reload và restart dịch vụ; không xóa các drop-in khác.

## Bước 4: Cấu hình xoay vòng log trên EC2

Tạo file cấu hình sau; nếu đã có file cùng tên, sao lưu và đối chiếu trước:

```bash
sudo tee /etc/logrotate.d/document-portal > /dev/null <<'CONF'
/var/log/document-portal/application.log {
    daily
    rotate 7
    maxsize 10M
    missingok
    notifempty
    compress
    delaycompress
    copytruncate
    su root root
}
CONF
sudo logrotate -d /etc/logrotate.d/document-portal
sudo systemctl enable --now logrotate.timer
sudo systemctl status logrotate.timer --no-pager
```

`logrotate -d` chỉ kiểm tra, không xoay log thật. maxsize được kiểm tra khi logrotate chạy, không phải giới hạn đĩa tức thì. Cấu hình daily/rotate 7 giữ tối đa 7 bản xoay vòng; khác với retention 7 ngày trên CloudWatch.

Bài lab dùng copytruncate để dịch vụ tiếp tục ghi vào file đang mở mà không restart và làm mất phiên. Có một khoảng đua nhỏ có thể làm mất log khi copy/truncate; phù hợp bài thực hành ít tải, không phải giải pháp bảo đảm audit không mất sự kiện. Không ép xoay log khi đang thu ảnh kiểm thử; theo dõi dung lượng đĩa.


## Bước 5: Cấu hình và khởi động CloudWatch Agent

Tạo file cấu hình:

```bash
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/workshop-logs.json > /dev/null <<'JSON'
{
  "agent": {
    "region": "ap-southeast-1",
    "run_as_user": "root"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/document-portal/application.log",
            "log_group_name": "/workshop/webserver",
            "log_stream_name": "{instance_id}/application"
          }
        ]
      }
    },
    "force_flush_interval": 5
  }
}
JSON
sudo python3 -m json.tool /opt/aws/amazon-cloudwatch-agent/etc/workshop-logs.json > /dev/null
```

Chỉ thu thập file application.log, không dùng wildcard toàn bộ /var/log. Không có phần metrics hoặc traces nên không chủ động thu thêm custom metrics/traces. Agent chạy root trong lab để đọc file hạn chế quyền; IAM vẫn giới hạn vào group đã nêu. Không thêm credentials tĩnh vào file hoặc môi trường của Agent.

Khởi động và bật tự chạy khi boot:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl   -a fetch-config -m ec2   -c file:/opt/aws/amazon-cloudwatch-agent/etc/workshop-logs.json -s
sudo systemctl enable amazon-cloudwatch-agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
sudo systemctl status amazon-cloudwatch-agent --no-pager
```

Kết quả mong đợi: Agent báo **running** và dịch vụ **active**. Chỉ active chưa chứng minh log đã được gửi; kiểm tra đầu cuối ở bước tiếp theo. Nếu lỗi validation, dừng để sửa JSON/quyền trước khi tiếp tục.



## Bước 6: Tạo sự kiện ứng dụng thật

Sau khi Agent chạy, mở website qua HTTPS và thực hiện theo thứ tự:

| Thao tác | Sự kiện dự kiến trong ứng dụng mẫu |
| --- | --- |
| Đăng nhập sai mật khẩu một lần bằng form hợp lệ | login_failed |
| Đăng nhập đúng | login_success |
| Tải lên một TXT/PDF mẫu nhỏ | upload_success bytes=... |
| Tải xuống tài liệu vừa tải | download_success bytes=... |
| Đăng xuất | logout |

Không thử sai liên tục vì ứng dụng có giới hạn tốc độ đăng nhập. Dùng tài liệu không chứa thông tin cá nhân. Ghi thời điểm từng thao tác và múi giờ, rồi kiểm tra trên EC2:

```bash
date -u
sudo tail -n 30 /var/log/document-portal/application.log
```

Tìm các sự kiện mới khớp thao tác. Nếu file không có sự kiện, xử lý log ứng dụng trước; không ghi tay các dòng login_success vào file để làm bằng chứng. Không cần cố tạo lỗi S3 hoặc thay quyền bảo mật để sinh storage_error.

Ứng dụng không được ghi mật khẩu, password hash, cookie, token, access key hoặc nội dung tài liệu. Kiểm tra nội dung log trước khi chụp/công khai; không bật SDK debug chứa request/body.



## Bước 7: Đối chiếu trên CloudWatch Logs

1. Mở **CloudWatch → Logs → Log groups → /workshop/webserver**.
2. Mở stream tên **INSTANCE_ID/application**, thay INSTANCE_ID bằng ID thật của máy. Có thể dùng ô tìm kiếm theo Instance ID.
3. Chọn khoảng thời gian 30 phút gần nhất hoặc khoảng bao gồm bài thử, rồi Refresh.
4. Chờ log được gửi và hiển thị; không yêu cầu xuất hiện tức thời.
5. Tìm lần lượt `login_failed`, `login_success`, `upload_success`, `download_success`, `logout`; xóa filter trước khi tìm sự kiện tiếp theo.
6. Đối chiếu nội dung và thời gian với file trên EC2. Cấu hình không khai báo timestamp_format nên timestamp sự kiện CloudWatch có thể dựa trên thời điểm Agent đọc; timestamp trong message là thời gian ứng dụng ghi. Lưu rõ múi giờ, không yêu cầu hai mốc trùng tuyệt đối.
7. Kiểm tra lại retention vẫn 7 ngày. Thực hiện thêm một lần đăng nhập/đăng xuất và xác nhận log mới tiếp tục xuất hiện.



## Lỗi thường gặp

| Hiện tượng | Cách xử lý |
| --- | --- |
| Agent active nhưng không có stream | Kiểm tra group tạo trước, Region, instance profile, quyền CreateLogStream/PutLogEvents và log lỗi Agent |
| AccessDenied | Đối chiếu ACCOUNT_ID, ARN, role thật trên EC2, SCP/permissions boundary và quyền liên quan; không gắn AdministratorAccess |
| ResourceNotFoundException | Tạo đúng /workshop/webserver tại ap-southeast-1 trước khi gửi |
| File không có sự kiện mới | Kiểm tra drop-in, restart dịch vụ, thao tác website và lỗi ứng dụng |
| File có log nhưng Agent không gửi | Kiểm tra file_path, quyền đọc, config đã fetch, credentials và mạng |
| Timeout kết nối Logs | Kiểm tra DNS, outbound 443, route IGW/Public IPv4 và NACL chiều về; S3 endpoint không phục vụ Logs |
| Retention vẫn Never expire | Người có quyền PutRetentionPolicy đặt lại trên Console; cấu hình này không giao việc đó cho Agent |
| SELinux từ chối đọc | Xem audit log và cấu hình policy/context phù hợp; không tắt SELinux toàn cục |
| Agent báo không đọc được JSON | Sửa lỗi cú pháp, chạy json.tool rồi fetch-config lại |

Lệnh chẩn đoán:

```bash
sudo tail -n 50 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
sudo tail -n 50 /opt/aws/amazon-cloudwatch-agent/logs/configuration-validation.log
sudo journalctl -u document-portal -n 30 --no-pager
```

Không chụp toàn bộ file cấu hình ứng dụng chứa password hash hoặc credentials. CloudWatch Logs phát sinh phí theo dữ liệu thu thập/lưu trữ; chỉ giữ log cần thiết. Retention trên CloudWatch không dọn file local; logrotate chịu trách nhiệm phía EC2.

## Kết quả hoàn thành

Có log ứng dụng mới từ EC2 trên đúng group/stream, retention 7 ngày, quyền ghi giới hạn và cơ chế xoay log local.

Tiếp tục: [5.7. Kiểm thử hệ thống](../../5.7-testing/).

## Tài liệu tham khảo

- [CloudWatch Agent configuration](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)
- [CloudWatch Logs IAM policies](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/iam-identity-based-access-control-cwl.html)
- [CloudWatch Agent troubleshooting](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/troubleshooting-CloudWatch-Agent.html)
