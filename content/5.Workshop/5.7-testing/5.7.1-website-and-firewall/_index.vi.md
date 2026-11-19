---
title: "Kiểm tra đăng nhập, tải lên, tải xuống và Firewall"
date: 2026-10-31
weight: 1
chapter: false
pre: "<b>5.7.1. </b>"
---

## Mục tiêu

Kiểm thử các ca **WEB-01–WEB-06** và **NET-01–NET-03** trong phần 5.7. Thành viên 2 kiểm tra website; thành viên 1 kiểm tra mạng và hoàn tác mọi thay đổi tạm thời. Ghi kết quả thực tế, không đánh dấu Pass trước khi thao tác.

## Chuẩn bị

- Hoàn thành 5.3–5.6; website chạy trên workshop-web-01 tại ap-southeast-1, Nginx phục vụ HTTPS 443, backend chỉ lắng nghe 127.0.0.1:8000.
- Chuẩn bị URL `https://PUBLIC_IPV4`, tài khoản website, TXT/PDF mẫu không rỗng và không quá **5 MiB = 5.242.880 byte**. Thay PUBLIC_IPV4 bằng IP hiện tại.
- Xác minh chứng chỉ tự ký theo 5.4.4 trước khi đăng nhập; không dùng tùy chọn bỏ kiểm tra TLS để che lỗi chứng chỉ.
- Có một máy quản trị với Public IPv4 /32 được phép SSH và một nguồn mạng khác ngoài danh sách cho phép. Cùng Wi-Fi thường có chung IP Internet, không đủ làm nguồn thử bị chặn.
- Ghi Instance ID, tất cả Security Group gắn với network interface, NACL và subnet association. Lưu ảnh quy tắc gốc.
- Người sửa NACL phải có AWS Console đang đăng nhập và quyền hoàn tác. Không dựa riêng vào SSH để phục hồi cấu hình mạng. Thử trên tài nguyên lab của nhóm, không sửa subnet dùng chung ngoài phạm vi.

## Bước 1: Kiểm tra yêu cầu đăng nhập — WEB-01

1. Mở cửa sổ riêng tư/ẩn danh chưa đăng nhập.
2. Truy cập URL HTTPS của website. Kết quả mong đợi: trang đăng nhập, không hiển thị danh sách tài liệu.
   ![](/FCAJ_Worklog/images/kiemthu/chualogin.png)
3. Dùng URL download của một tài liệu mẫu đã có: sao chép liên kết tải xuống từ một phiên đăng nhập riêng, rồi dán vào cửa sổ chưa đăng nhập. Không sao chép cookie hay token.
   ![](/FCAJ_Worklog/images/kiemthu/kiemthu1.png)
4. Kết quả mong đợi: chuyển về đăng nhập, không trả nội dung file. 
    ![](/FCAJ_Worklog/images/kiemthu/chualogin.png)

## Bước 2: Kiểm tra đăng nhập — WEB-02

1. Nhập tài khoản đúng và mật khẩu sai **một lần**, gửi form bình thường.
   
2. Xác nhận bị từ chối và chưa xem được tài liệu.
   ![](/FCAJ_Worklog/images/kiemthu/dangnhapfail.png)
3. Nhập thông tin đúng, xác nhận vào được trang danh sách tài liệu.
    ![](/FCAJ_Worklog/images/kiemthu/dangnhapthanhcong.png)
4. Không chụp mật khẩu hoặc cookie. Nếu gặp giới hạn tốc độ, đợi rồi thử lại; không coi lỗi giới hạn tốc độ là bằng chứng sai mật khẩu.




## Bước 3: Kiểm tra upload hợp lệ — WEB-03

1. Chuẩn bị `kiem-thu.txt` UTF-8 và `kiem-thu.pdf` PDF thật, mỗi file nhỏ hơn giới hạn. Không tạo PDF bằng cách đổi đuôi TXT.
   ![](/FCAJ_Worklog/images/kiemthu/2filekiemthu.png)
2. Ghi dung lượng và SHA-256 bản gốc. Trên **PowerShell của máy cá nhân**, trong thư mục chứa file:

```powershell
Get-Item .\kiem-thu.txt, .\kiem-thu.pdf | Select-Object Name, Length
Get-FileHash .\kiem-thu.txt -Algorithm SHA256
Get-FileHash .\kiem-thu.pdf -Algorithm SHA256
```
    
![](/FCAJ_Worklog/images/kiemthu/buoc2.png)

3. Trên website đã đăng nhập, chọn từng file → Upload.
    ![](/FCAJ_Worklog/images/kiemthu/txt.png)
    ![](/FCAJ_Worklog/images/kiemthu/pdf.png)
   
4. Xác nhận upload thành công, file mới xuất hiện. Ứng dụng thêm tiền tố ngẫu nhiên vào tên; ghi lại **tên thực tế** và URL download.
    ![](/FCAJ_Worklog/images/kiemthu/upfilexong.png)
5. Nhờ thành viên 3 đối chiếu object mới dưới `portal/` trong đúng bucket nếu cần xác nhận lưu trữ đầu cuối.
   ![](/FCAJ_Worklog/images/kiemthu/s3check.png)

## Bước 4: Kiểm tra file bị từ chối — WEB-04

Trong thư mục thử nghiệm riêng trên máy cá nhân, dùng PowerShell tạo các file mới sau. Nếu trùng tên file đang có, đổi tên trước, không ghi đè dữ liệu thật.

```powershell
Set-Content -Path .\khong-hop-le.exe -Value 'Harmless test text' -Encoding ascii
Set-Content -Path .\gia-pdf.pdf -Value 'This is not a PDF' -Encoding ascii
[System.IO.File]::WriteAllText((Join-Path (Get-Location) 'rong.txt'), '')
[System.IO.File]::WriteAllText((Join-Path (Get-Location) 'qua-lon.txt'), ('A' * 5242881))
Get-Item .\khong-hop-le.exe, .\gia-pdf.pdf, .\rong.txt, .\qua-lon.txt | Select-Object Name, Length
```

File .exe ở đây chỉ chứa văn bản vô hại, không chạy file đó. `qua-lon.txt` có nội dung ASCII và lớn hơn 5 MiB một byte.

1. Upload lần lượt bốn file khi đang đăng nhập.
2. Kỳ vọng: từ chối phần mở rộng không hỗ trợ, PDF giả, file rỗng và file vượt giới hạn.
3. Nếu hộp chọn file chỉ hiển thị TXT/PDF, chọn All files để chọn file thử. Nếu trình duyệt chặn trước khi gửi, chỉ ghi nhận kiểm tra phía trình duyệt; chưa kết luận máy chủ đã kiểm tra.
4. Kiểm tra thông báo lỗi hoặc HTTP status trong Developer Tools → Network nếu cần. Không chụp request headers chứa cookie.
5. Đối chiếu danh sách website và object mới trên S3 theo thời gian thử để xác nhận không có object được tạo từ các lần bị từ chối.

Ứng dụng có giới hạn file 5 MiB và Nginx có giới hạn request 6 MiB theo phần trước. Lỗi 413 do request quá lớn có thể đến từ Nginx; không coi mọi trường hợp từ chối là cùng một lớp kiểm tra.
![qua-lon.txt](/FCAJ_Worklog/images/kiemthu/1.png)
![gia-pdf.pdf](/FCAJ_Worklog/images/kiemthu/2.png)
![khong-hop-le.exe](/FCAJ_Worklog/images/kiemthu/4.png)




## Bước 5: Tải xuống và đối chiếu — WEB-05

1. Tải TXT và PDF đã upload ở bước 3 bằng nút Download.
2. Lưu vào thư mục riêng để tránh ghi đè bản gốc. Mở file, kiểm tra nội dung đọc được.
3. Chạy Get-FileHash với **đường dẫn file tải về thực tế**, rồi so sánh với SHA-256 bản gốc. Ví dụ dưới dùng tên giữ chỗ, phải thay:

```powershell
Get-FileHash 'C:\DUONG_DAN_THUC_TE\FILE_TAI_VE.txt' -Algorithm SHA256
```

4. Pass khi hai cặp hash khớp và file mở được. Tên file có thể khác do tiền tố ngẫu nhiên; nội dung phải giữ nguyên.

![](/FCAJ_Worklog/images/kiemthu/test5.png)

## Bước 6: Đăng xuất và thử truy cập lại — WEB-06

1. Khi đang đăng nhập, lưu URL tải tài liệu, sau đó chọn **Log out**.
   ![](/FCAJ_Worklog/images/kiemthu/logout.png)
2. Dán lại URL download vào thanh địa chỉ của cùng trình duyệt và gửi yêu cầu mới.
    ![](/FCAJ_Worklog/images/kiemthu/kiemthu1.png)
3. Mở lại trang danh sách. Kỳ vọng cả hai yêu cầu đều cần đăng nhập lại.
   ![](/FCAJ_Worklog/images/kiemthu/chualogin.png)
4. Nút Back có thể hiển thị trang đã lưu trong lịch sử; phải Reload/gửi yêu cầu mới để kiểm tra phía máy chủ. File đã tải về máy trước đó không tự biến mất khi logout.
   ![](/FCAJ_Worklog/images/kiemthu/chualogin.png)




## Bước 7: SSH từ nguồn được phép và bị chặn — NET-01

Trên máy cá nhân, kiểm tra kết nối TCP bằng PowerShell, thay IP:

```powershell
Test-NetConnection -ComputerName PUBLIC_IPV4 -Port 22
```

1. Từ nguồn được phép /32, kỳ vọng `TcpTestSucceeded : True`; xác nhận đăng nhập SSH bằng key đã có theo 5.4.1.
2. Từ nguồn mạng thứ hai ngoài danh sách, chạy lại lệnh; kỳ vọng `False` hoặc timeout. Không cần chép private key sang máy khác chỉ để thử TCP.
3. Ghi Public IPv4 thực tế của cả hai nguồn, thời điểm thử và quy tắc SG/NACL tương ứng.
4. Nếu cả hai nguồn đều thất bại, kiểm tra EC2 Running, IP, route, NACL và sshd trước; chưa kết luận allow/deny đạt.

Lỗi SSH `Permission denied (publickey)` cho thấy đã đến dịch vụ SSH nhưng xác thực key thất bại, không phải bằng chứng firewall chặn TCP. Nếu thiếu nguồn thứ hai, ghi **Chưa kiểm thử**. Ca này kiểm tra kết quả của các lớp mạng kết hợp; không quy riêng lỗi cho SG nếu NACL cũng chặn nguồn đó.



## Bước 8: HTTPS và backend — NET-02

Trên máy cá nhân:

```powershell
Test-NetConnection -ComputerName PUBLIC_IPV4 -Port 443
Test-NetConnection -ComputerName PUBLIC_IPV4 -Port 8000
```

Kỳ vọng 443 True và website HTTPS hoạt động; 8000 False. Trên EC2, đối chiếu socket:

```bash
sudo ss -lntp
```

Xác nhận Gunicorn chỉ bind **127.0.0.1:8000**, Nginx nghe 443, tất cả SG gắn với máy không mở 8000 ra Internet. Không thay bind address hoặc mở cổng backend để tạo bằng chứng thử. Kết nối 8000 thất bại cùng kiểm tra socket/quy tắc xác nhận backend không công khai, không chứng minh riêng hiệu lực của SG.

Chạy `curl.exe -I http://PUBLIC_IPV4/` trên Windows hoặc `curl -I http://PUBLIC_IPV4/` trên Linux/macOS. Kỳ vọng 301 và Location trỏ đến HTTPS. Đối chiếu cổng 81 với 5.3.6.



## Bước 9: Kiểm thử NACL có kiểm soát — NET-03

Thử chặn **HTTPS từ duy nhất IP máy kiểm thử**, giữ nguyên SSH và outbound. Chỉ làm khi đã có Console và quyền hoàn tác.

1. Xác định Public IPv4 hiện tại của máy thử, gọi là **TEST_PUBLIC_IP/32**. Đây là chỗ giữ chỗ, phải thay bằng CIDR thật; tránh VPN/proxy làm nguồn trình duyệt và PowerShell khác nhau.
2. Trước thay đổi, chạy lại kiểm tra TCP 443 từ máy đó và ghi kết quả True.
3. Trong **VPC → Network ACLs**, chọn đúng **nacl-web-public** theo ID. Kiểm tra **Subnet associations** có subnet-web-public của EC2. Lưu inbound/outbound gốc.
4. Mở **Inbound rules → Edit inbound rules → Add new rule**. Nếu số 90 còn trống và nằm trước quy tắc cho phép 443, thêm:

| Trường | Giá trị thử |
| --- | --- |
| Rule number | 90 |
| Type / Protocol / Port | HTTPS / TCP / 443 |
| Source | TEST_PUBLIC_IP/32 thực tế |
| Allow/Deny | DENY |

5. Nếu 90 đã được dùng, không ghi đè: chọn số còn trống nhỏ hơn quy tắc allow 443 liên quan. Kiểm tra không có quy tắc allow số nhỏ hơn đã khớp trước deny thử. Nếu không chắc, dừng và nhờ người phụ trách mạng đối chiếu.
6. Lưu, mở **kết nối TCP mới** bằng Test-NetConnection tới 443; kỳ vọng False/timeout. Không dùng trang trình duyệt cache hoặc phiên cũ làm bằng chứng duy nhất.
7. **Hoàn tác ngay sau khi ghi nhận:** xóa duy nhất dòng deny thử vừa tạo, lưu lại. Không sửa dấu * mặc định, không xóa quy tắc gốc, không thay subnet association.
8. Chạy lại Test-NetConnection 443, kỳ vọng True; mở website và đăng nhập được. Xác nhận SSH vẫn hoạt động.

Nếu deny không có hiệu lực, kiểm tra nguồn IP, số thứ tự và NACL association; không mở rộng deny sang 0.0.0.0/0. Nếu thử không thành công, vẫn phải xóa dòng tạm thời trước khi xử lý tiếp.

**📸 Ảnh 09:** Rule deny và kết quả chặn 443. 

![](/FCAJ_Worklog/images/kiemthu/9.png)

**📸 Ảnh 10:** Rule tạm đã xóa, 443 phục hồi và website hoạt động. 

![](/FCAJ_Worklog/images/kiemthu/10.png)

## Tổng hợp kết quả

| Mã | Kết quả cần có | Trạng thái thực tế |
| --- | --- | --- |
| WEB-01 | Chưa đăng nhập không đọc được tài liệu | Pass |
| WEB-02 | Sai bị từ chối, đúng đăng nhập được | Pass|
| WEB-03 | TXT/PDF hợp lệ upload thành công | Pass |
| WEB-04 | File không hợp lệ bị từ chối, không tạo object | Pass |
| WEB-05 | Download và SHA-256 khớp | Pass |
| WEB-06 | Yêu cầu mới sau logout cần đăng nhập | Pass |
| NET-01 | SSH allow/deny theo nguồn | Pass |
| NET-02 | HTTPS hoạt động, backend không công khai | Pass|
| NET-03 | NACL chặn rồi phục hồi sau hoàn tác | Pass |




Tiếp tục: [5.7.2. Kiểm tra quyền truy cập S3 và kết quả giám sát](../5.7.2-s3-and-monitoring/).
