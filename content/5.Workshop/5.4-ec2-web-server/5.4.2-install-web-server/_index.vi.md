---
title: "Cài đặt máy chủ web"
date: 2026-10-30
weight: 2
chapter: false
pre: "<b>5.4.2. </b>"
---

## Mục tiêu

Thành viên 2 cài đặt **Nginx** trên EC2 **workshop-web-01** chạy **Amazon Linux 2023**, bật dịch vụ tự khởi động và kiểm tra khả năng trả trang web. Nginx sẽ nhận kết nối HTTPS và chuyển yêu cầu đến ứng dụng ở các phần tiếp theo.

Kiểm tra trang Nginx qua localhost, sau đó thử qua Public IPv4. Phần 5.4.3 hướng dẫn triển khai ứng dụng tài liệu.

## Chuẩn bị

- Hoàn thành [5.4.1. Khởi tạo EC2 và kết nối SSH](../5.4.1-launch-and-connect/).
- SSH vào đúng workshop-web-01 bằng ec2-user, có quyền sudo.
- EC2 có đường ra để tải gói: Route Table, DNS, Security Group outbound và NACL hai chiều đã cấu hình.
- S3 Endpoint policy chưa chặn bucket chứa gói hệ điều hành của AWS.

**Trừ bước mở đường hầm SSH được ghi riêng, các lệnh trong phần này chạy trong terminal SSH trên EC2.** Chạy lần lượt từng khối lệnh, kiểm tra kết quả trước khi chuyển bước.

## Bước 1: Kiểm tra hệ điều hành và cập nhật gói

```bash
whoami
cat /etc/os-release
sudo dnf upgrade -y
```

Xác nhận user ec2-user và Amazon Linux 2023. Lệnh upgrade cập nhật theo phiên bản repository hiện đang cấu hình. AL2023 dùng repository theo phiên bản; lệnh này không tự bảo đảm chuyển sang bản phát hành AL2023 mới nhất. Nếu dùng AMI cũ, kiểm tra hướng dẫn cập nhật của AWS và thống nhất phiên bản trước khi nâng cấp hệ điều hành.

Nếu quá trình cập nhật yêu cầu khởi động lại, chạy `sudo reboot`, chờ máy sẵn sàng rồi kết nối SSH lại. Reboot làm ngắt phiên SSH; không đồng nghĩa phải tạo EC2 mới.

![](/FCAJ_Worklog/images/54/inx1.png)


## Bước 2: Cài Nginx

```bash
sudo dnf install -y nginx
nginx -v
rpm -q nginx
```

Kết quả phải có phiên bản Nginx và tên gói đã cài. Ghi phiên bản thực tế vào nhật ký triển khai; không cần giống số phiên bản trong ảnh tham khảo. Không dùng amazon-linux-extras hoặc apt trên hệ điều hành này.

Bài này chỉ cài máy chủ Nginx. Runtime và thư viện ứng dụng sẽ được cài theo mã nguồn ở 5.4.3, tránh cài nhiều stack không dùng đến.

![](/FCAJ_Worklog/images/54/inx2.png)


## Bước 3: Kiểm tra cấu hình và chạy dịch vụ

```bash
sudo nginx -t
```

Chỉ tiếp tục khi có thông báo `syntax is ok` và `test is successful`. Sau đó chạy:

```bash
sudo systemctl enable --now nginx
systemctl is-active nginx
systemctl is-enabled nginx
sudo systemctl status nginx --no-pager
```

Kết quả mong đợi: **active** và **enabled**. Active cho biết dịch vụ đang chạy; enabled cho biết dịch vụ được thiết lập tự chạy khi khởi động máy.

Nếu nginx -t báo lỗi, sửa đúng file/dòng được chỉ ra trước khi khởi động hoặc reload. Không bỏ qua lỗi để tiếp tục.

![](/FCAJ_Worklog/images/54/inx3.png)


## Bước 4: Kiểm tra HTTP ngay trên EC2

```bash
curl -I http://127.0.0.1/
curl -sS http://127.0.0.1/ | head -n 15
sudo ss -lntp
```

Với cấu hình mặc định của gói, mong đợi **HTTP 200**, nội dung HTML và Nginx lắng nghe cổng 80. Nếu kết quả khác, xem cấu hình thực tế bằng `sudo nginx -T` và log ở bước 6; không sửa Firewall khi yêu cầu localhost chưa hoạt động.

HTTP ở đây chỉ dùng để kiểm tra máy chủ cục bộ. Chưa có TLS nên truy cập https://PUBLIC_IPV4 chưa thành công là bình thường. Việc curl localhost thành công cũng chưa chứng minh người dùng Internet truy cập được website.

![](/FCAJ_Worklog/images/54/inx4.png)


## Bước 5: Xem trang mặc định qua đường hầm SSH

Trên **máy cá nhân**, mở terminal/PowerShell thứ hai tại thư mục chứa YOUR_KEY_PAIR.pem. Thay PUBLIC_IPV4 bằng địa chỉ hiện tại của EC2, chạy:

```bash
ssh -i "YOUR_KEY_PAIR.pem" -N -L 127.0.0.1:8080:127.0.0.1:80 ec2-user@PUBLIC_IPV4
```

Giữ cửa sổ này mở. Lệnh `-N` không mở shell nên terminal không hiện dấu nhắc mới là bình thường khi kết nối thành công. Mở trình duyệt trên **chính máy cá nhân đó** tại:

```text
http://127.0.0.1:8080
```

Trang mặc định Nginx phải xuất hiện. Đường hầm dùng kết nối SSH đã mã hóa trên cổng 22; không cần thêm inbound 80 hoặc 8080 vào Security Group/NACL. Địa chỉ 127.0.0.1:8080 chỉ phục vụ kiểm tra trên máy của bạn, không phải URL website công khai.

Nếu cổng 8080 đang bận, đổi duy nhất `127.0.0.1:8080` phía đầu lệnh thành `127.0.0.1:8081`, rồi mở trình duyệt với cổng 8081. Đóng đường hầm bằng **Ctrl+C** sau khi chụp ảnh. Không nhập thông tin tài khoản ứng dụng ở bước này.




## Bước 6: Xác định cấu hình và nhật ký

| Vị trí thông thường của gói | Vai trò |
| --- | --- |
| /etc/nginx/nginx.conf | Cấu hình chính |
| /etc/nginx/conf.d/ | Cấu hình bổ sung nếu được include từ cấu hình chính |
| /usr/share/nginx/html/ | Nội dung trang mặc định |
| /var/log/nginx/access.log | Nhật ký yêu cầu HTTP |
| /var/log/nginx/error.log | Nhật ký lỗi Nginx |

Kiểm tra log sau các yêu cầu thử:

```bash
sudo tail -n 20 /var/log/nginx/access.log
sudo tail -n 20 /var/log/nginx/error.log
sudo journalctl -u nginx -n 30 --no-pager
```

Error log có thể trống nếu chưa phát sinh lỗi. Cấu hình đã chỉnh trước đó có thể dùng đường dẫn khác; `sudo nginx -T` cho biết cấu hình hiệu lực. Không công khai toàn bộ cấu hình/log nếu có thông tin nhạy cảm.

Sau khi sửa cấu hình ở các phần tiếp theo, kiểm tra rồi reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Không lưu mật khẩu, private key, file .env hoặc tài liệu S3 trong thư mục web mặc định. Ở 5.4.3 sẽ thiết lập ứng dụng và dịch vụ riêng; ở 5.4.4 sẽ cấu hình HTTPS và kết nối Nginx đến backend. Không cần thay thế toàn bộ nginx.conf trong bước cài đặt này.

## Lỗi thường gặp

| Hiện tượng | Kiểm tra/xử lý |
| --- | --- |
| dnf tải gói thất bại | Kiểm tra DNS, route, outbound 80/443, inbound ephemeral và S3 Endpoint policy; không tắt kiểm tra chữ ký gói |
| nginx: command not found | Xem lệnh cài có thành công không; xác nhận đúng EC2 và hệ điều hành |
| Address already in use | Dùng sudo ss -lntp xác định dịch vụ đang chiếm cổng; không dừng một dịch vụ chưa xác định |
| nginx -t báo lỗi | Sửa file/dòng được báo; kiểm tra dấu chấm phẩy và cặp ngoặc nếu đã chỉnh cấu hình |
| curl localhost bị từ chối | Kiểm tra trạng thái Nginx, cổng listen và journalctl |
| Tunnel không mở được | Kiểm tra SSH, Public IPv4, private key và cổng 8080 cục bộ; giữ terminal đường hầm mở |
| HTTP timeout | Kiểm tra SG/NACL cổng 80 và IP hiện tại. Kiểm tra localhost để xác nhận Nginx. |

## Kết quả hoàn thành

Nginx đã được cài, cấu hình hợp lệ, dịch vụ active/enabled và trang mặc định được kiểm tra bằng curl cùng trình duyệt qua SSH tunnel. 

Tiếp tục: [5.4.3. Triển khai đăng nhập và chức năng tải lên, tải xuống tài liệu](../5.4.3-document-web-app/).

## Tài liệu tham khảo

- [AL2023 package management](https://docs.aws.amazon.com/linux/al2023/ug/package-management.html)
- [Manage AL2023 updates](https://docs.aws.amazon.com/linux/al2023/ug/managing-repos-os-updates.html)
- [Nginx beginner guide](https://nginx.org/en/docs/beginners_guide.html)
- [Nginx command-line parameters](https://nginx.org/en/docs/switches.html)

