---
title: "Ghi VPC Flow Logs và đối chiếu cổng 80, 81"
weight: 6
chapter: false
pre: "<b>5.3.6. </b>"
---

## Thiết lập ghi log ở cấp subnet

1. Thành viên 4 tạo log group `/workshop/vpc/flow-logs` ở Singapore nếu chưa có. Đặt retention 7 ngày cho lab.
2. Quản trị viên kiểm tra trust của `WorkshopFlowLogsDeliveryRole` cho `vpc-flow-logs.amazonaws.com`, quyền ghi CloudWatch Logs và quyền `iam:PassRole` đúng role cho `WorkshopNetworkRole`.
3. Thành viên 1 vào **VPC → Subnets → subnet-web-public → Flow logs**. Đối chiếu subnet `subnet-03d4ed311fc7d0e28`. Dùng lại `flow-web-public` nếu đã có. Khi tạo mới chọn **Filter = All**, **Maximum aggregation interval = 1 minute**, **Destination = Send to CloudWatch Logs**, đúng log group và delivery role. Chọn định dạng mặc định cho câu truy vấn bên dưới.
4. Xác nhận trạng thái Active. Sau khi tạo lưu lượng, chờ các bản ghi xuất hiện và kiểm tra lỗi delivery nếu chưa thấy.

<!-- 📸 Chèn ảnh subnet, Flow Log và log group tại đây. -->

## Đối chiếu lưu lượng

Sau khi Nginx chạy, dùng PowerShell trên máy cá nhân, thay PUBLIC_IPV4 bằng IP hiện tại:

```powershell
Test-NetConnection -ComputerName PUBLIC_IPV4 -Port 80
Test-NetConnection -ComputerName PUBLIC_IPV4 -Port 81
```

Trên Linux/macOS, có thể dùng `curl -I --connect-timeout 5 http://PUBLIC_IPV4/` và `curl -I --connect-timeout 5 http://PUBLIC_IPV4:81/`. Kết quả đã quan sát trong lab là **ACCEPT 80** và **REJECT 81**. Sau khi bật HTTPS, 80 trả chuyển hướng. Giữ 81 ngoài danh sách cổng được phép.

Trong **CloudWatch → Logs Insights**, chọn log group và khoảng thời gian thử. Với định dạng Flow Logs mặc định, chạy:

```text
fields @timestamp, @message
| parse @message "* * * * * * * * * * * * * *" as version, account, eni, src, dst, sport, dport, proto, packets, bytes, started, ended, action, status
| filter dport in ["80", "81"] and proto = "6"
| display @timestamp, eni, src, dst, dport, action, status
| sort @timestamp desc
```

Đối chiếu ENI, IP nguồn và thời gian kiểm thử. Flow Logs cho biết lưu lượng được chấp nhận hay từ chối. Muốn xác định quy tắc chặn, kiểm tra thêm SG và NACL. Một phút là khoảng tổng hợp tối đa, bản ghi có thể đến sau.

<!-- 📸 Chèn ảnh cặp bản ghi ACCEPT 80 và REJECT 81 tại đây. -->

[Tạo Flow Log gửi CloudWatch Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-cwl-create-flow-log.html)
