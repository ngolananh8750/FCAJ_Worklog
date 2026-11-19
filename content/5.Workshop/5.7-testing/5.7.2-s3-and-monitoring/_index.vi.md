---
title: "Kiểm tra quyền truy cập S3 và kết quả giám sát"
date: 2026-10-31
weight: 2
chapter: false
pre: "<b>5.7.2. </b>"
---

## Mục tiêu

Hoàn thành **S3-01–S3-04** và **MON-01–MON-03** của phần 5.7. Thành viên 3 kiểm tra bảo mật dữ liệu; thành viên 4 kiểm tra giám sát; phối hợp với thành viên 1 và 2 khi đối chiếu mạng hoặc chạy lệnh trên EC2.

Phần này dùng cấu hình đã triển khai, không cấp thêm quyền rộng để làm kết quả kiểm thử thành công. Ghi nhận Pass/Fail dựa trên bằng chứng thật.

## Chuẩn bị

- Hoàn thành 5.7.1; đã xóa quy tắc NACL tạm thời và website hoạt động trở lại.
- Ghi Instance ID, bucket thực tế, Region **ap-southeast-1**, role **WorkshopEC2AppRole**, endpoint **s3-gateway-endpoint** và route table **rtb-web-public**.
- Script `ec2-s3-check.py` của phần 5.5.3 đã được đặt tại `/opt/document-portal/ec2-s3-check.py` trên EC2. Nếu thiếu, chuyển lại theo hướng dẫn 5.5.3; không dùng script khác không rõ nội dung.
- Chuẩn bị object mẫu không chứa dữ liệu cá nhân và quyền Console đọc cấu hình S3, IAM, VPC, CloudWatch theo phần việc. Quyền người dùng Console khác với quyền instance profile của EC2.
- Alarm **workshop-web-01-cpu-high**, log group **/workshop/webserver**, stream **INSTANCE_ID/application** đã được cấu hình.

## Bước 1: Kiểm tra danh tính và phạm vi quyền

1. Trong **EC2 → Instances → workshop-web-01 → Security**, xác nhận role đang gắn là **WorkshopEC2AppRole**.
2. Trong IAM, xem các policy gắn vào role. Quyền S3 dự kiến: ListBucket có điều kiện prefix `portal/`, GetObject/PutObject trên `portal/*` của đúng bucket.
3. Đối chiếu cả inline policies, managed policies và bucket policy; một policy giới hạn không tự vô hiệu hóa quyền rộng được cấp ở policy khác.
4. Giữ policy ghi CloudWatch Logs của 5.6.3. Không thay role hoặc tạo access key để chạy thử.


## Bước 2: Kiểm tra đọc/ghi và truy cập bị từ chối — S3-01, S3-02

Trên EC2, chạy script bằng user dịch vụ trong môi trường sạch:

```bash
sudo -u portal env -i \
  PATH=/usr/bin:/bin HOME=/nonexistent \
  AWS_SHARED_CREDENTIALS_FILE=/dev/null AWS_CONFIG_FILE=/dev/null \
  /opt/document-portal/.venv/bin/python /opt/document-portal/ec2-s3-check.py
```
Script đọc tên bucket từ cấu hình, yêu cầu credentials đến từ instance profile và kiểm tra Caller ARN có role đúng. Nó không in mật khẩu/hash hoặc access key.

| Thao tác của script | Kết quả cần có |
| --- | --- |
| List prefix portal/ | LIST portal/: PASS |
| Tạo TXT mới có key ngẫu nhiên portal/ec2-check-...txt | PutObject thành công |
| Đọc lại object và kiểm tra bytes, SSE-S3 | PUT/GET/SSE-S3: PASS và SHA256 |
| List prefix outside/ | PASS (AccessDenied) |
| Xóa chính object dùng một lần vừa tạo | PASS (AccessDenied) |

**Lưu ý ca xóa:** Script chỉ thử xóa object mới do chính lần chạy tạo ra. Nếu quyền quá rộng, thao tác xóa có thể thành công và script sẽ báo FAIL. Không thay key bằng tài liệu thật. Nếu ca list outside/ đã thất bại theo nghĩa được cho phép ngoài dự kiến, script dừng trước ca xóa; đánh dấu ca xóa chưa thực hiện.

1. Ghi Caller ARN, bucket, key thử và thời gian chạy.
2. Chỉ ghi S3-01 Pass nếu list/read/write và kiểm tra nội dung thành công.
3. Chỉ ghi S3-02 Pass nếu **cả hai** thao tác ngoài phạm vi đều trả AccessDenied.
4. Timeout, lỗi DNS, thiếu credentials hoặc lỗi STS không phải AccessDenied. STS đi qua kết nối HTTPS Internet hiện có, không qua S3 Gateway Endpoint.
5. Nếu có FAIL, dừng để kiểm tra toàn bộ quyền; không sửa script để đổi kết quả mong đợi. Lưu object thử còn lại để quản trị dọn ở 5.8.

![](/FCAJ_Worklog/images/kiemthu/s302.png)

## Bước 3: Kiểm tra truy cập ẩn danh — S3-03

1. Trong **S3 → bucket → Objects → portal/**, chọn một object mẫu đang tồn tại. Xác nhận người quản trị dữ liệu có thể tải nó bằng quyền hợp lệ.
2. Trong thông tin object, sao chép **Object URL thông thường**, không dùng Share with a presigned URL hoặc URL sinh ra từ nút Open/Download có chữ ký.
3. Kiểm tra URL không có tham số chữ ký như `X-Amz-Signature`, `X-Amz-Credential`, `X-Amz-Security-Token`. Dùng HTTPS và đúng Region.
4. Mở URL trong cửa sổ riêng tư không đăng nhập. Kỳ vọng **403 AccessDenied**, không có nội dung tài liệu.
5. Nếu được tải file, kiểm tra đúng URL không ký, bucket policy và quyền công khai. Nếu là DNS lỗi, timeout, redirect sai Region hoặc key không tồn tại, chưa đủ bằng chứng ca này đạt.

Presigned URL cấp quyền tạm thời; mở được nó không có nghĩa bucket công khai. Truy cập website sau đăng nhập và truy cập S3 ẩn danh là hai luồng khác nhau.

**📸 Ảnh :** Object mẫu và truy cập URL không ký bị AccessDenied. 

![](/FCAJ_Worklog/images/kiemthu/s303.png)

## Bước 4: Kiểm tra bảo mật lưu trữ — S3-04

1. Mở bucket → **Permissions**: xác nhận cả bốn mục **Block Public Access** bật và **Object Ownership = Bucket owner enforced**, ACLs disabled.
   ![](/FCAJ_Worklog/images/kiemthu/41.png)
   ![](/FCAJ_Worklog/images/kiemthu/42.png)
   ![](/FCAJ_Worklog/images/kiemthu/43.png)
2. Xem bucket policy HTTPS-only đã cấu hình ở 5.5.3; không xóa policy để vượt lỗi.
3. Mở **Properties → Default encryption**: xác nhận **SSE-S3**.
4. Mở object được website tải lên ở 5.7.1 → **Properties → Server-side encryption**: kiểm tra SSE-S3 trên chính object. Mã hóa mặc định của bucket không thay thế kiểm tra object thực tế.
5. Ghi bucket, key và kết quả. Đây là kiểm tra cấu hình HTTPS-only, không tuyên bố đã kiểm thử yêu cầu HTTP có chữ ký.


## Bước 5: Đối chiếu S3 Gateway Endpoint — S3-04

1. Trong EC2, ghi subnet/VPC của workshop-web-01.
2. Mở **VPC → Subnets → subnet thực tế → Route table** để xác định bảng định tuyến đang có hiệu lực, không chọn chỉ dựa vào Name tag.
3. Mở **Endpoints → s3-gateway-endpoint**, đối chiếu VPC, loại **Gateway**, service **com.amazonaws.ap-southeast-1.s3**, trạng thái **Available**.
4. Trong Route tables của endpoint, xác nhận liên kết với bảng đang có hiệu lực của subnet EC2.
5. Trong Routes của bảng đó, xác nhận `pl-...` của S3 Singapore → `vpce-...` của endpoint. Giữ tuyến local và tuyến Internet `0.0.0.0/0 → igw-...`.
6. Đối chiếu bucket cùng Region và ứng dụng dùng S3 endpoint thông thường của Region, không endpoint tùy chỉnh/proxy ngoài thiết kế.

Kết quả ở đây là **kiểm tra cấu hình định tuyến kết hợp thử S3 từ EC2**, không phải bản ghi đường đi của từng request. DNS S3 có thể trả IP công cộng dù route đi qua Gateway Endpoint. Không dùng tải thành công từ laptop hoặc traceroute làm bằng chứng duy nhất. Không tháo route/endpoint hay thêm Deny theo SourceVpce chỉ để thử ở mục này.

![](/FCAJ_Worklog/images/kiemthu/s304.png)


## Bước 6: Kiểm tra CPU và cảnh báo — MON-01, MON-02

1. Mở **CloudWatch → Metrics → All metrics → EC2 → Per-Instance Metrics**.
2. Chọn CPUUtilization theo đúng InstanceId; đặt **Average**, **5 minutes**, khoảng thời gian có dữ liệu mới. Lưu thời gian/múi giờ.
3. Mở **Alarms → workshop-web-01-cpu-high**. Đối chiếu Instance ID, ngưỡng **>80%**, period **300 giây**, **2/2**, missing data **missing**.
4. Mở **History**, chọn khoảng thời gian bài thử ở 5.6.2. Kiểm tra ALARM có lý do dựa trên datapoints vượt ngưỡng và sau đó phục hồi OK khi tải đã dừng.
5. Có thể dùng bằng chứng 5.6.2 nếu tài nguyên/cấu hình không thay đổi; ghi rõ thời gian gốc. Nếu chưa có bằng chứng phù hợp hoặc cấu hình đã đổi, thực hiện lại đúng bài thử có giới hạn ở 5.6.2, kiểm tra credits trước; không chạy tải vô hạn.
6. Nếu dùng SNS, xác nhận email khớp alarm/thời gian. Nếu không triển khai SNS, ghi Không áp dụng cho email.

OK hiện tại không chứng minh đã từng kiểm thử ALARM. SetAlarmState thủ công không chứng minh CPU vượt ngưỡng. INSUFFICIENT_DATA không phải trạng thái hệ thống khỏe. Kết thúc phải xác nhận tải đã dừng và website hoạt động.

**📸 Ảnh :** CPU đúng máy, cấu hình alarm và lịch sử ALARM → OK. 
![](/FCAJ_Worklog/images/55/ec2m2.png)
![](/FCAJ_Worklog/images/kiemthu/mon02.png)
## Bước 7: Kiểm tra log mới — MON-03

1. Ghi thời điểm bắt đầu. Trên website HTTPS, đăng nhập sai một lần, đăng nhập đúng, upload/download tài liệu mẫu và logout. Không tạo nhiều lần sai vì giới hạn tốc độ.
2. Trên EC2, đối chiếu log ứng dụng:

```bash
date -u
sudo tail -n 50 /var/log/document-portal/application.log
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

3. Mở **CloudWatch → Logs → Log groups → /workshop/webserver → INSTANCE_ID/application**, chọn khoảng thời gian chứa bài thử, Refresh.
4. Tìm `login_failed`, `login_success`, `upload_success`, `download_success`, `logout`. So khớp thông điệp và thời điểm với file EC2; xóa filter khi đổi loại sự kiện.
5. Phân biệt timestamp trong message với thời gian Agent đọc/đẩy log; không yêu cầu trùng tuyệt đối. Kiểm tra múi giờ UTC/UTC+7.
6. Xác nhận retention 7 ngày và không có mật khẩu, cookie/token, access key hay nội dung tài liệu trong log.

Agent running hoặc log cũ có sẵn chưa đủ: phải thấy **sự kiện mới khớp thao tác lần này**. Nếu chưa thấy, kiểm tra file local, cấu hình Agent, quyền ghi log và kết nối HTTPS đến CloudWatch Logs theo 5.6.3; không mở bucket công khai để xử lý lỗi log.

**📸 Ảnh 08:** Sự kiện mới trong đúng stream, cùng bằng chứng local để đối chiếu. 
![](/FCAJ_Worklog/images/kiemthu/mon3.png)

## Tổng hợp và xử lý lỗi

| Mã | Tiêu chí | Trạng thái thực tế |
| --- | --- | --- |
| S3-01 | Đúng role, list/read/write portal/ và nội dung khớp | Pass |
| S3-02 | List ngoài prefix và xóa object thử bị AccessDenied | Pass |
| S3-03 | URL object không ký từ chối truy cập ẩn danh | Pass |
| S3-04 | Bucket/object bảo mật và route endpoint đúng | Pass |
| MON-01 | CPU mới của đúng Instance ID | Pass |
| MON-02 | Bằng chứng ALARM và phục hồi OK phù hợp | Pass |
| MON-03 | Log mới khớp thao tác, retention 7 ngày |Pass |

Nếu read/write bị AccessDenied, kiểm tra role, bucket/prefix và các policy áp dụng; nếu thao tác ngoài phạm vi lại được phép, tìm quyền rộng ở tất cả policy liên quan. Với timeout, kiểm tra mạng trước. Với log trống, phân biệt ứng dụng chưa ghi với Agent chưa gửi.

Điền kết quả theo mẫu 5.7, gồm người thử, thời gian, tài nguyên, bằng chứng và nguyên nhân/cách sửa nếu Fail. Lưu **8 vị trí ảnh**; không ghi Pass cho ca chưa chạy. Không công khai credentials, presigned URL hoặc thông tin bí mật trong báo cáo.

## Hoàn tất

Xác nhận website vẫn hoạt động, tải CPU đã dừng, cấu hình bảo mật không bị mở rộng. Ghi danh sách object thử do script/website tạo và giữ bằng chứng trước khi dọn dẹp. Không cấp DeleteObject cho role ứng dụng chỉ để xóa file thử; quản trị dữ liệu xử lý trong phần 5.8.

Tiếp tục: [5.8. Dọn dẹp tài nguyên](../../5.8-cleanup/).



