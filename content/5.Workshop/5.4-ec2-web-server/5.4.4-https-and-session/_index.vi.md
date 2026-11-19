---
title: "Cấu hình HTTPS và kiểm tra phiên đăng nhập"
date: 2026-10-31
weight: 4
chapter: false
pre: "<b>5.4.4. </b>"
---

## Mục tiêu

Cấu hình Nginx kết thúc TLS trên cổng 443 và chuyển tiếp tới Gunicorn **127.0.0.1:8000** của phần 5.4.3. Sau đó kiểm tra cookie, đăng nhập, đăng xuất và thời hạn phiên.

Bài lab dùng **chứng chỉ tự ký cho Public IPv4**, không cần tên miền. Chứng chỉ mã hóa kết nối nhưng không được trình duyệt tin cậy mặc định. Chỉ dùng tài khoản/tài liệu thực hành; triển khai thực tế cần chứng chỉ được tin cậy và quy trình gia hạn phù hợp.

## Chuẩn bị

- Hoàn thành 5.4.3: document-portal active và /healthz trên localhost:8000 trả 200.
- Nginx và OpenSSL trên Amazon Linux 2023; quyền sudo qua SSH.
- sg-secure-web cho inbound TCP 443; NACL cho inbound 443 và outbound cổng tạm thời theo 5.3.4.
- Cổng 80 chuyển hướng sang HTTPS. Gunicorn chỉ lắng nghe ở 127.0.0.1:8000.
- Ghi Public IPv4 hiện tại. Bucket/IAM chưa sẵn sàng vẫn kiểm tra được TLS và trang login; danh sách tài liệu sau đăng nhập có thể báo 503 cho đến khi hoàn tất 5.5.

Các lệnh chạy trên EC2, trừ phần được ghi rõ máy cá nhân.

## Bước 1: Tạo chứng chỉ có IP SAN

Nhập Public IPv4 thực tế khi được hỏi. Nếu lệnh Python báo lỗi địa chỉ, dừng và nhập lại trước khi tạo chứng chỉ. Giữ cùng phiên SSH để biến PORTAL_IP còn tồn tại.


```bash
sudo dnf install -y openssl
read -r -p "Current EC2 public IPv4: " PORTAL_IP
python3 -c 'import ipaddress,sys; ipaddress.IPv4Address(sys.argv[1])' "$PORTAL_IP"

```


```bash
sudo install -d -m 700 /etc/nginx/tls
sudo openssl req -x509 -nodes -newkey rsa:2048 -sha256 -days 30 \
  -keyout /etc/nginx/tls/portal.key \
  -out /etc/nginx/tls/portal.crt \
  -subj "/CN=$PORTAL_IP" \
  -addext "subjectAltName=IP:$PORTAL_IP" \
  -addext "extendedKeyUsage=serverAuth"
sudo chmod 600 /etc/nginx/tls/portal.key
sudo chmod 644 /etc/nginx/tls/portal.crt
sudo openssl x509 -in /etc/nginx/tls/portal.crt -noout -dates -fingerprint -sha256 -ext subjectAltName
```

![](/FCAJ_Worklog/images/54/https1.png)

## Bước 2: Cấu hình HTTPS và reverse proxy

Cấu hình dưới đây dành riêng cho EC2 mới của workshop và thay thế trang mặc định từ 5.4.2. Nếu máy đang phục vụ website khác, không dùng nguyên khối thay thế; cần ghép cấu hình riêng. Lệnh sao lưu tạo file có thời gian để tránh ghi đè bản cũ; ghi lại đường dẫn đó.

Cấu hình không include conf.d để tránh server mặc định/cổng 80 từ cấu hình cũ. Những file bổ sung trong conf.d sẽ không được áp dụng. Chạy cả khối; chỉ reload khi nginx -t thành công.


```bash
sudo cp -a /etc/nginx/nginx.conf "/etc/nginx/nginx.conf.before-https.$(date +%Y%m%d%H%M%S)"
sudo tee /etc/nginx/nginx.conf > /dev/null <<'NGINX'
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;
include /usr/share/nginx/modules/*.conf;
events { worker_connections 1024; }
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    sendfile on;
    server_tokens off;
    # Do not record URL paths, query strings, cookies, or credentials.
    log_format portal '$time_iso8601 $request_method $status $body_bytes_sent';
    access_log /var/log/nginx/access.log portal;
    server {
        listen 80 default_server;
        server_name _;
        return 301 https://PUBLIC_IPV4$request_uri;
    }
    server {
        listen 443 ssl default_server;
        server_name _;
        ssl_certificate /etc/nginx/tls/portal.crt;
        ssl_certificate_key /etc/nginx/tls/portal.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_session_tickets off;
        client_max_body_size 6m;
        location / {
            proxy_pass http://127.0.0.1:8000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Forwarded-Proto https;
            proxy_connect_timeout 5s;
            proxy_read_timeout 90s;
        }
    }
}
NGINX
sudo sed -i "s/PUBLIC_IPV4/${PORTAL_IP:?Set PORTAL_IP first}/g" /etc/nginx/nginx.conf
sudo nginx -t && sudo systemctl reload nginx
sudo ss -lntp
```

![](/FCAJ_Worklog/images/54/https2.png)

## Bước 3: Xác minh TLS ngay trên EC2

Các lệnh dưới đây dùng chứng chỉ vừa tạo làm trust anchor, đồng thời kiểm tra IP SAN. --resolve kết nối localhost nhưng vẫn xác minh tên/IP của URL, tránh phụ thuộc việc EC2 tự gọi Public IPv4 của chính nó.

Mong đợi /healthz trả status ok; / trả 302 với Location /login. Không dùng curl -k làm bằng chứng xác minh chứng chỉ.


```bash
sudo curl --noproxy '*' --cacert /etc/nginx/tls/portal.crt \
  --resolve "$PORTAL_IP:443:127.0.0.1" "https://$PORTAL_IP/healthz"
sudo curl --noproxy '*' --cacert /etc/nginx/tls/portal.crt \
  --resolve "$PORTAL_IP:443:127.0.0.1" -I "https://$PORTAL_IP/"
```

![](/FCAJ_Worklog/images/54/https3.png)

## Bước 4: Kiểm tra từ máy cá nhân

Trên EC2, chép **chứng chỉ công khai**, không chép private key:


```bash
sudo install -m 644 /etc/nginx/tls/portal.crt /home/ec2-user/portal.crt
```

Trên **máy cá nhân**, cùng thư mục key pair, thay PUBLIC_IPV4 rồi chạy:

```bash
scp -i "YOUR_KEY_PAIR.pem" ec2-user@PUBLIC_IPV4:~/portal.crt .
```


```bash
curl --cacert portal.crt https://PUBLIC_IPV4/healthz
```

Windows PowerShell: dùng `curl.exe` thay cho `curl` nếu curl là alias. Chứng chỉ được chuyển qua SSH đã xác minh; không tải chứng chỉ từ một kết nối chưa xác thực rồi tự coi là tin cậy. Nếu lỗi, không thêm -k; kiểm tra IP, SAN và file chứng chỉ.

## Bước 5: Kiểm tra trình duyệt và cookie

Mở **https://PUBLIC_IPV4/login** trên máy cá nhân. Với chứng chỉ tự ký, trình duyệt sẽ cảnh báo không tin cậy. Mở chi tiết chứng chỉ, đối chiếu SHA-256 fingerprint, IP SAN và thời hạn với kết quả bước 1 trước khi cho phép ngoại lệ cho đúng trang lab. Nếu không khớp hoặc trình duyệt không cho tiếp tục, dừng và xử lý tin cậy chứng chỉ; không tắt kiểm tra bảo mật toàn trình duyệt.

Ngoại lệ không biến chứng chỉ tự ký thành chứng chỉ công khai tin cậy. Không yêu cầu ảnh “ổ khóa xanh”. Bằng chứng nên là chi tiết chứng chỉ đúng IP và kết quả curl --cacert thành công.

Mở Developer Tools (F12) → Application/Storage → Cookies. Cookie **portal_session** phải có Secure, HttpOnly, SameSite=Lax, Path=/ và thời hạn khoảng 30 phút. Cookie trước login là phiên chưa xác thực phục vụ CSRF; sau login đúng, ID phiên được thay mới. **Che toàn bộ giá trị cookie/token trong ảnh**, chỉ để lại tên và thuộc tính.

Đăng nhập bằng tài khoản website đã tạo, không dùng IAM user. Nếu sau login hiện 503 Storage unavailable, kiểm tra quyền S3 ở 5.5; không tắt Secure hoặc sửa TLS để xử lý lỗi dữ liệu.

![](/FCAJ_Worklog/images/54/https4.png)

## Bước 6: Kiểm tra vòng đời phiên

Thực hiện lần lượt các ca trong bảng. Dùng Developer Tools → Network để quan sát mã phản hồi và bật Preserve log nếu cần; không công khai request body chứa mật khẩu hoặc token.

| Ca kiểm thử | Kết quả cần đạt |
| --- | --- |
| Cửa sổ ẩn danh mở / hoặc /download/sample.txt | 302 về /login, không đọc S3 |
| Sai mật khẩu | 401, không tạo phiên đã xác thực |
| Đăng nhập đúng | POST /login trả 302; cookie đổi ID; danh sách cần S3 sẵn sàng |
| Chọn Log out | POST /logout thu hồi phiên, xóa cookie và về /login |
| Sau logout mở lại / | Chuyển về login; bấm Back có thể hiện ảnh trang cũ, phải reload để kiểm tra quyền thực tế |
| Chờ quá 30 phút từ khi tạo phiên đã xác thực | Yêu cầu trang bảo vệ mới bị chuyển về login; hoạt động không kéo dài hạn phiên |
| Restart ứng dụng | Mọi phiên hiện tại bị mất hiệu lực |

Để kiểm tra restart, đăng nhập rồi chạy trên EC2:

```bash
sudo systemctl restart document-portal
systemctl is-active document-portal
```

Sau đó refresh trình duyệt: phải đăng nhập lại. Không tăng số worker quá 1 để khắc phục phiên bị mất. Không chụp cookie cũ nhằm làm bằng chứng; bài test replay cookie có thể kiểm tra riêng trong môi trường kín.



## Lỗi thường gặp và khôi phục

| Lỗi | Xử lý |
| --- | --- |
| Timeout cổng 443 | Kiểm tra Public IPv4, IGW, SG 443, NACL và listener Nginx |
| 502 Bad Gateway | Kiểm tra document-portal active, curl localhost:8000/healthz và nginx error.log |
| Permission denied khi proxy và SELinux Enforcing | Kiểm tra getenforce và audit log; nếu đúng SELinux chặn kết nối reverse proxy, quản trị có thể bật httpd_can_network_connect bằng setsebool -P; không tắt SELinux toàn bộ |
| Chứng chỉ không khớp IP | Public IPv4 đã đổi; tạo lại chứng chỉ SAN đúng, reload và chuyển lại public certificate |
| nginx -t lỗi | Không reload; sửa lỗi hoặc khôi phục bản nginx.conf.before-https đã ghi, kiểm tra lại rồi reload |
| Login lặp lại | Kiểm tra HTTPS, cookie, 1 worker và service có restart liên tục không |
| 503 Storage unavailable | Kiểm tra S3/IAM ở 5.5, không phải lỗi chứng chỉ |
| 413 upload | Nginx cho request 6m; ứng dụng vẫn chỉ cho nội dung tệp tối đa 5 MiB |

Nếu khôi phục cấu hình cũ chỉ có HTTP, dừng sử dụng đăng nhập cho đến khi HTTPS được khôi phục. Không nhập tài khoản qua HTTP.

## Kết quả hoàn thành

Có HTTPS trên 443, backend chỉ ở localhost, cookie đúng thuộc tính và kiểm tra phiên đạt.  Kết quả yêu cầu S3 để sau 5.5 nếu chưa sẵn sàng. Chứng chỉ hết hạn sau 30 ngày; lập kế hoạch tạo lại cho lab. Sau Stop/Start, kiểm tra Public IPv4 trước khi dùng lại. Không bật HSTS trong bài chứng chỉ tự ký này.

Các lệnh là hướng dẫn để thực hiện trên EC2 của nhóm; chưa được chạy trên tài khoản AWS của bạn.

Tiếp tục: [5.5. Lưu trữ và bảo mật dữ liệu trên S3](../../5.5-s3-data-security/).

## Tài liệu tham khảo

- [Nginx HTTPS configuration](https://nginx.org/en/docs/http/configuring_https_servers.html)
- [OpenSSL req](https://docs.openssl.org/3.0/man1/openssl-req/)
