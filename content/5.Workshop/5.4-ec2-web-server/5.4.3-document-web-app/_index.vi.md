---
title: "Triển khai đăng nhập và chức năng tải lên, tải xuống tài liệu"
date: 2026-10-30
weight: 3
chapter: false
pre: "<b>5.4.3. </b>"
---

## Mục tiêu

Triển khai ứng dụng tài liệu mẫu trên EC2 Amazon Linux 2023 đã cài Nginx ở 5.4.2. Thành viên 2 dùng **Python 3.11 + Flask + Gunicorn**, tích hợp S3 bằng Boto3 và IAM Role.

Tải **document-portal.zip** được cung cấp cùng bài này. Gói gồm app.py, configure.py, requirements.txt và document-portal.service. Đây là mã mẫu độc lập cho workshop, không yêu cầu repository bên ngoài.

Ứng dụng có đăng nhập/đăng xuất, danh sách tài liệu, upload và download qua backend. Dùng một tài khoản chung; không có đăng ký, cơ sở dữ liệu hoặc chức năng xóa. Tệp lưu dưới portal/ với tiền tố ngẫu nhiên tránh ghi đè, bucket giữ riêng tư.

**Phụ thuộc:** EC2/SSH/Nginx đã hoạt động. Có tên bucket thực tế đã thống nhất; bucket và quyền có thể hoàn thiện sau ở 5.5. Chỉ kiểm thử dữ liệu thành công khi bucket, IAM Role và policy đã sẵn sàng. HTTPS sẽ cấu hình ở 5.4.4.

## Bước 1: Chuyển mã nguồn lên EC2

Trên **máy cá nhân**, đặt ZIP cạnh YOUR_KEY_PAIR.pem, thay PUBLIC_IPV4 rồi chạy lệnh sau. Không chuyển private key lên EC2.

```bash
scp -i "YOUR_KEY_PAIR.pem" document-portal.zip ec2-user@PUBLIC_IPV4:~/
```

![](/FCAJ_Worklog/images/54/deploy01.png)

## Bước 2: Cài môi trường và tài khoản dịch vụ

Các lệnh từ đây chạy **trên EC2 qua SSH**. Dùng thư mục cài đặt mới. Nếu user portal đã tồn tại, kiểm tra bằng `id portal` và bỏ qua useradd; không xóa tài khoản hoặc ghi đè ứng dụng đang dùng.

```bash
sudo dnf install -y python3.11 python3.11-pip unzip
unzip -n ~/document-portal.zip -d ~/portal-source
sudo useradd --system --home-dir /opt/document-portal --shell /sbin/nologin portal
sudo install -d -m 755 /opt/document-portal
sudo cp ~/portal-source/document-portal/app.py /opt/document-portal/
sudo cp ~/portal-source/document-portal/requirements.txt /opt/document-portal/
sudo cp ~/portal-source/document-portal/configure.py /opt/document-portal/
sudo python3.11 -m venv /opt/document-portal/.venv
sudo /opt/document-portal/.venv/bin/pip install -r /opt/document-portal/requirements.txt
sudo /opt/document-portal/.venv/bin/pip check
sudo /opt/document-portal/.venv/bin/pip freeze | sudo tee /opt/document-portal/requirements.lock.txt > /dev/null
```

![](/FCAJ_Worklog/images/54/deploy02.png)

## Bước 3: Tạo cấu hình và tài khoản website

Chạy trình cấu hình tương tác. Nhập tên bucket thực tế, tên đăng nhập website và mật khẩu ít nhất 12 ký tự hai lần. Mật khẩu được nhập ẩn và lưu dưới dạng hash, không đặt vào dòng lệnh. Không nhập IAM password hoặc AWS access key.

Script từ chối ghi đè cấu hình có sẵn. Khi cần đổi cấu hình, quản trị sao lưu riêng rồi chỉnh có kiểm soát; không xóa file để thử lại một cách tùy tiện.

```bash
sudo /opt/document-portal/.venv/bin/python /opt/document-portal/configure.py
sudo chown root:portal /etc/document-portal /etc/document-portal/config.json
sudo chmod 750 /etc/document-portal
sudo chmod 640 /etc/document-portal/config.json
sudo -u portal test -r /etc/document-portal/config.json && echo "Config readable by portal"
```

![](/FCAJ_Worklog/images/54/deploy03.png)

## Bước 4: Khởi động Gunicorn bằng systemd

Dịch vụ chạy bằng user portal không có quyền sudo, code nằm ngoài thư mục web. Chỉ bind **127.0.0.1:8000**. Giữ **1 worker** vì phiên và bộ đếm đăng nhập nằm trong RAM của tiến trình; tăng workers sẽ làm phiên không nhất quán.

Kết quả: service active, cổng 8000 chỉ ở loopback, /healthz trả 200 và trang / trả 302 đến /login khi chưa xác thực. Healthcheck không gọi S3.

```bash
sudo install -m 644 ~/portal-source/document-portal/document-portal.service /etc/systemd/system/document-portal.service
sudo systemctl daemon-reload
sudo systemctl enable --now document-portal
sudo systemctl status document-portal --no-pager
sudo ss -lntp
curl -i http://127.0.0.1:8000/healthz
curl -I http://127.0.0.1:8000/
```

![](/FCAJ_Worklog/images/54/deploy04.png)

## Bước 5: Kiểm tra log và khởi động lại

Log ghi sự kiện như login_success, login_failed, upload_success, download_success và storage_error; không ghi mật khẩu, cookie hoặc nội dung tệp. Bàn giao tên service cho thành viên CloudWatch. Log hiện nằm trong journal; việc thu thập sang CloudWatch sẽ được cấu hình ở 5.6.3.

Khởi động lại sẽ xóa mọi phiên trong RAM, người dùng cần đăng nhập lại. Đây là hành vi chủ ý của ứng dụng lab.

```bash
sudo journalctl -u document-portal -n 50 --no-pager
sudo systemctl restart document-portal
systemctl is-active document-portal
systemctl is-enabled document-portal
```

![](/FCAJ_Worklog/images/54/deploy05.png)

## Bước 6: Kiểm thử chức năng sau khi có HTTPS và S3

Cookie phiên luôn có Secure, HttpOnly và SameSite=Lax. **Không tắt Secure để đăng nhập qua HTTP.** Phần này dừng ở kiểm tra dịch vụ; hoàn thành 5.4.4 rồi mở https://địa-chỉ-máy-chủ/login. Nginx cần chuyển tiếp đến 127.0.0.1:8000 và đặt client_max_body_size 6m để đủ phần bao multipart. Ứng dụng giới hạn riêng nội dung tệp **5 MiB = 5 × 1024 × 1024 byte** (quy ước 5 MB của lab).

| Ca kiểm thử | Kết quả mong đợi |
| --- | --- |
| Mở / hoặc /download/sample.txt chưa đăng nhập | Chuyển đến /login, không đọc S3 |
| Sai mật khẩu | Thông báo chung, không tạo phiên đã xác thực |
| Đăng nhập đúng | Danh sách tài liệu khi S3 sẵn sàng |
| Upload TXT UTF-8 hoặc PDF hợp lệ | Lưu dưới portal/, SSE-S3, xuất hiện trong danh sách |
| Tệp .exe, PDF sai header, TXT sai UTF-8, tệp rỗng hoặc >5 MiB | Bị từ chối tại máy chủ |
| Download | Nội dung tải về đúng tệp đã gửi; trả dạng attachment |
| Logout rồi dùng lại cookie cũ | Phiên bị thu hồi, phải đăng nhập lại |
| POST thiếu token CSRF | HTTP 400 |





## Quyền S3 và giới hạn mẫu

Role cần s3:ListBucket trên bucket, giới hạn prefix portal/; s3:GetObject và s3:PutObject trên arn:aws:s3:::TEN_BUCKET/portal/*. Không cần DeleteObject hoặc bucket public. Boto3 lấy thông tin xác thực từ instance profile; không chạy aws configure với access key lâu dài.

Phiên hết hạn cố định sau 30 phút. Tối đa 1.000 phiên trong RAM; giới hạn 10 lần thử đăng nhập/phút dùng chung cho toàn ứng dụng, có thể khiến cả nhóm phải chờ khi một người thử nhiều lần. Đây là mẫu một máy cho lab, chưa phải hệ thống production. Kiểm tra PDF chỉ là header và kiểm tra TXT là UTF-8, **không phải quét mã độc**; chỉ dùng tài liệu thực hành. Không có phân trang giao diện cho kho lớn.

## Xử lý lỗi

- 503 Storage unavailable: kiểm tra bucket, Region, IAM Role, endpoint policy và mạng; không đổi bucket sang public.
- Service failed: xem journalctl, quyền đọc config và thư viện đã cài; không in config ra ảnh.
- Lặp về trang login: dùng HTTPS, kiểm tra cookie Secure, thời hạn và đúng 1 worker.
- 413 khi upload: kiểm tra giới hạn tệp và client_max_body_size của Nginx.
- 429: chờ một phút; không tắt giới hạn bằng cách mở nhiều worker.

Mã mẫu đã được kiểm tra cục bộ với S3 giả lập: đăng nhập, CSRF, giới hạn tệp, prefix/mã hóa, download và thu hồi cookie. Chưa được chạy trên tài khoản AWS của nhóm; kiểm thử thực tế theo bảng trên vẫn cần thực hiện.

Tiếp tục: [5.4.4. Cấu hình HTTPS và kiểm tra phiên đăng nhập](../5.4.4-https-and-session/).

## Tài liệu tham khảo

- [Flask security](https://flask.palletsprojects.com/en/stable/web-security/)
- [Boto3 credentials](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html)
