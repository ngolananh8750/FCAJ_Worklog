---
title: "Tạo và kiểm thử cảnh báo CPU"
date: 2026-10-31
weight: 2
chapter: false
pre: "<b>5.6.2. </b>"
---


## Mục tiêu

Thành viên 4 tạo **workshop-web-01-cpu-high** và phối hợp với thành viên 2 tạo tải có giới hạn trên EC2 lab. Kiểm tra luồng **OK → ALARM → OK** bằng dữ liệu CPU thực tế, lưu đồ thị và Alarm history.

## Chuẩn bị

- Hoàn thành [5.6.1. Theo dõi chỉ số CPU của EC2](../5.6.1-cpu-metrics/); có dữ liệu CPU mới của workshop-web-01 tại ap-southeast-1.
- Thành viên 4 có quyền đọc metric, `cloudwatch:PutMetricAlarm`, `cloudwatch:DescribeAlarms`, `cloudwatch:DescribeAlarmHistory` trong phạm vi được giao. Không cần cấp quyền CloudWatch cho EC2 chỉ để tạo alarm trên Console.
- Thành viên 2 có quyền SSH và sudo trên Amazon Linux 2023.
- Thống nhất thời gian thử: CPU cao có thể làm website chậm. Không chạy khi nhóm đang trình diễn hoặc xử lý dữ liệu quan trọng.
- Giữ **Basic monitoring** và CPU credits ở **Standard**. Alarm có thể phát sinh phí theo tài khoản và mức sử dụng.

## Bước 1: Chọn metric cho cảnh báo

1. Mở **CloudWatch**, chọn **Singapore — ap-southeast-1**.
2. Chọn **Alarms → All alarms → Create alarm**.
3. Chọn **Select metric → EC2 → Per-Instance Metrics**.
4. Tìm Instance ID thực tế của **workshop-web-01**.
5. Chọn dòng **CPUUtilization** có đúng InstanceId, rồi chọn **Select metric**.
6. Đặt **Statistic = Average**, **Period = 5 minutes**.

Nếu alarm cùng tên đã tồn tại, kiểm tra Instance ID và người quản lý trước khi sửa; không ghi đè alarm của nhóm khác.

![](/FCAJ_Worklog/images/55/m1.png)

## Bước 2: Đặt điều kiện

| Trường | Giá trị |
| --- | --- |
| Threshold type | Static |
| Whenever CPUUtilization is | Greater |
| Threshold | 80 |
| Datapoints to alarm | 2 out of 2 |
| Missing data treatment | Treat missing data as missing |

1. Nhập điều kiện **CPU > 80%**.
2. Mở **Additional configuration**, đặt **2/2 datapoints**.
3. Chọn cách xử lý dữ liệu thiếu là **missing**; không chọn not breaching cho bài này.
4. Chọn **Next**.

Hai điểm CPU trung bình theo chu kỳ 5 phút phải vi phạm điều kiện để đáp ứng cấu hình 2/2 khi dữ liệu đầy đủ. Một đỉnh CPU vài giây không đủ. Trạng thái không đổi tức thì sau khi chạy lệnh; còn phụ thuộc thời điểm chu kỳ, dữ liệu gửi về và đánh giá alarm.

![](/FCAJ_Worklog/images/55/m2.png)

## Bước 3: Cấu hình hành động và tạo alarm

1. Ở **Configure actions**, bài lab mặc định kiểm tra trực tiếp trên Console. Chỉ bỏ qua notification khi tạo mới và chưa cấu hình SNS. Với alarm đã có, giữ action của nhóm.
2. Không thêm EC2 action Stop, Terminate, Reboot, Recover hoặc Auto Scaling action.
3. Chọn **Next**, nhập tên **workshop-web-01-cpu-high**.
4. Description: `CPU Average above 80 percent for 2 periods of 5 minutes on workshop web server`.
5. Chọn **Next**, kiểm tra lại Instance ID và tất cả thông số, rồi **Create alarm**.
6. Mở alarm, xem **Details** và **History**. Chờ trạng thái **OK** với dữ liệu nền dưới ngưỡng trước khi thử tải.

| Trạng thái | Ý nghĩa |
| --- | --- |
| OK | Dữ liệu được đánh giá không thỏa điều kiện cảnh báo |
| ALARM | Điều kiện cảnh báo được đáp ứng |
| INSUFFICIENT_DATA | Chưa có đủ dữ liệu để xác định trạng thái |

INSUFFICIENT_DATA lúc mới tạo không nhất thiết là lỗi. Nếu kéo dài, kiểm tra metric, Instance ID, Region và dữ liệu mới như phần 5.6.1.

![](/FCAJ_Worklog/images/55/m3.png)

### Tùy chọn: Nhận email qua SNS

Chỉ thực hiện nếu nhóm muốn có thông báo email và người thao tác có quyền SNS phù hợp:

1. Trong SNS tại cùng Region, tạo hoặc dùng lại topic **Standard** tên `workshop-alerts` của nhóm.
2. Mở topic → **Create subscription** → Protocol **Email**, nhập email nhận cảnh báo của thành viên, rồi tạo subscription.
3. Người nhận mở email AWS và chọn **Confirm subscription**. Kiểm tra subscription không còn Pending confirmation.
4. Sửa alarm, thêm notification cho trạng thái **In alarm** tới topic này; có thể thêm **OK** nếu muốn nhận thông báo phục hồi. Lưu thay đổi trước khi chạy tải.
5. Đảm bảo alarm actions được bật và topic policy cho phép CloudWatch gửi thông báo. Nếu có AccessDenied, nhờ quản trị bổ sung đúng quyền; không công khai topic.

Nếu không có quyền tạo topic/subscription, nhờ người được phân quyền chuẩn bị hoặc hoàn thành nhánh Console trước. Không coi trạng thái ALARM là bằng chứng email đã đến.


## Bước 4: Kiểm tra CPU credits trước khi tạo tải

1. Trong CloudWatch Metrics → EC2 → Per-Instance Metrics, chọn **CPUCreditBalance** của cùng Instance ID.
2. Xem điểm mới nhất với period 5 phút; có thể dùng Average để quan sát.
3. Kiểm tra credit mode của instance trong EC2 là **Standard**. Không đổi sang Unlimited chỉ để kích hoạt alarm.
4. Với bài thử hai vCPU tối đa 15 phút, nên có ít nhất khoảng **30 CPU credits** làm mức chuẩn bị bảo thủ; đây không phải điều kiện bắt buộc của AWS hoặc cam kết tải sẽ đạt ngưỡng.

Nếu credits quá thấp, hoãn bài thử để máy tích lũy credits khi nhàn rỗi. Không chạy tải lặp lại liên tục. Khi hết credits ở Standard, CPU bị giới hạn quanh baseline và có thể không vượt 80%.

## Bước 5: Tạo tải có thời gian tự dừng

Trên máy cá nhân, SSH như phần 5.4.1, thay PUBLIC_IPV4 bằng IP thực tế:

```bash
ssh -i "YOUR_KEY_PAIR.pem" ec2-user@PUBLIC_IPV4
```

Trên EC2, kiểm tra số CPU và thời gian:

```bash
nproc
date -u
```

Lệnh dùng số vCPU do `nproc` trả về.

Chạy **một lần**:

```bash
sudo systemd-run --unit=workshop-cpu-test \
  --property=RuntimeMaxSec=900 \
  --property=KillMode=control-group \
  /bin/bash -c 'for ((i=0; i<$(nproc); i++)); do yes > /dev/null & done; wait'
```

Các tiến trình yes tạo tải CPU, không ghi dữ liệu ra đĩa. systemd tự dừng dịch vụ sau tối đa 900 giây (15 phút); KillMode dừng cả tiến trình con. Lệnh trả về ngay, không phải đợi 15 phút tại terminal. Không tạo unit khác để chạy chồng bài thử.

Kiểm tra:

```bash
sudo systemctl status workshop-cpu-test --no-pager
top
```

Trong top, nhấn `q` để thoát màn hình theo dõi; **q không dừng tải**. Trên máy hai vCPU, mỗi tiến trình có thể gần 100% trong top, trong khi CloudWatch thể hiện mức tổng thể của instance theo cách tính khác.

Dừng tải ngay khi đã có bằng chứng ALARM hoặc website quá chậm:

```bash
sudo systemctl stop workshop-cpu-test
```

Nếu SSH ngắt, bộ giới hạn của systemd vẫn hoạt động. Nếu lệnh báo unit đã tồn tại, kiểm tra trạng thái trước; không chạy thêm bản sao. Với unit cũ đã dừng nhưng ở trạng thái failed, có thể dùng `sudo systemctl reset-failed workshop-cpu-test`, rồi chỉ thử lại khi đã xác định nguyên nhân. 


## Bước 6: Quan sát ALARM và phục hồi

1. Quay lại CloudWatch → Alarms → **workshop-web-01-cpu-high**.
2. Chọn khoảng thời gian **1 hour**, làm mới định kỳ. Không giảm period hoặc đổi ngưỡng chỉ để có ảnh ALARM.
3. Đối chiếu đồ thị: cần các điểm Average vượt 80% đáp ứng 2/2. Bài thử 15 phút tạo thời gian cho ít nhất hai chu kỳ đầy đủ khi tải duy trì được, nhưng không bảo đảm kết quả nếu thiếu credits hoặc có vấn đề dữ liệu.
4. Khi chuyển **ALARM**, xem **History** và lý do thay đổi trạng thái. Ghi thời gian, múi giờ, giá trị các datapoints và dừng tải.
5. Nếu dùng SNS, kiểm tra hộp thư và spam. Chỉ ghi nhận email thành công nếu nhận được thông báo khớp alarm và thời gian thử.


Xác nhận tải đã dừng:

```bash
sudo systemctl is-active workshop-cpu-test
```

Kết quả phải không còn active. Unit có thể ở trạng thái failed do chạm RuntimeMaxSec hoặc đã được systemd giải phóng; kiểm tra nhật ký nếu cần:

```bash
sudo journalctl -u workshop-cpu-test -n 30 --no-pager
```

Tiếp tục chờ dữ liệu CPU thấp được gửi và đánh giá. Khi alarm về **OK**, chụp History; không dừng EC2 để tạo thiếu dữ liệu thay cho kiểm thử phục hồi. Kiểm tra website vẫn đăng nhập và tải tài liệu bình thường.



## Lỗi thường gặp

| Hiện tượng | Xử lý |
| --- | --- |
| Alarm luôn OK | Kiểm tra CPU thực tế, đúng Instance ID, Average, 5 phút, >80 và 2/2; một điểm cao không đủ |
| CPU không lên cao dù yes chạy | Kiểm tra CPUCreditBalance, Standard mode và hai tiến trình tải; không chuyển Unlimited để ép kết quả |
| Alarm vẫn ALARM sau khi dừng tải | Chờ dữ liệu mới, xem History và CPU; kiểm tra có tải khác đang chạy |
| INSUFFICIENT_DATA kéo dài | Kiểm tra EC2 Running, metric, Region, time range và dữ liệu thiếu |
| Không nhận email | Kiểm tra subscription, topic, alarm action, quyền publish và spam; email không gửi liên tục chỉ vì vẫn ALARM |
| systemd unit failed sau 15 phút | Có thể do giới hạn thời gian chủ động; xem journal và xác nhận tải đã dừng |
| AccessDenied khi tạo alarm | Kiểm tra PutMetricAlarm trên đúng tên/ARN và quyền đọc; không cấp AdministratorAccess |

Nếu hết 15 phút chưa ALARM, dừng, lưu kết quả thực tế và xử lý nguyên nhân; không ghi nhận kiểm thử thành công. Không dùng SetAlarmState làm bằng chứng CPU đã vượt ngưỡng: thay trạng thái thủ công không kiểm chứng luồng metric thực tế.

## Kết quả và bàn giao

Ghi lại Instance ID, tên alarm, cấu hình, thời gian bắt đầu/dừng tải, thời điểm ALARM/OK và kết quả SNS nếu có. Giữ alarm để kiểm thử ở 5.7; dọn dẹp ở 5.8.

Tiếp tục: [5.6.3. Thu thập và kiểm tra nhật ký ứng dụng](../5.6.3-application-logs/).

## Tài liệu tham khảo

- [Create a CPU usage alarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/US_AlarmAtThresholdEC2.html)
- [Alarm evaluation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-evaluation.html)
- [CPU credits and baseline](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-credits-baseline-concepts.html)
- [SNS notifications for alarm changes](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Notify_Users_Alarm_Changes.html)
