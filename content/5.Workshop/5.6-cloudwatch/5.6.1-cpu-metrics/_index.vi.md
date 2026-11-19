---
title: "Theo dõi chỉ số CPU của EC2"
date: 2026-10-31
weight: 1
chapter: false
pre: "<b>5.6.1. </b>"
---

## Mục tiêu

Thành viên 4 sử dụng Amazon CloudWatch để xem **CPUUtilization** của **workshop-web-01**, cấu hình đồ thị thống nhất và ghi nhận dữ liệu nền trước khi kiểm thử cảnh báo ở phần 5.6.2.

Phần này chỉ xem chỉ số có sẵn của EC2, chưa cài CloudWatch Agent và chưa tạo tải CPU.

## Chuẩn bị

- Hoàn thành các bước chuẩn bị ở [5.6. Giám sát và cảnh báo với CloudWatch](../).
- EC2 workshop-web-01 đang chạy tại **Singapore — ap-southeast-1**.
- Tài khoản thao tác có quyền xem EC2 và đọc CloudWatch metrics, bao gồm các thao tác cần thiết như `ec2:DescribeInstances`, `cloudwatch:ListMetrics`, `cloudwatch:GetMetricData`, `cloudwatch:GetMetricStatistics`.
- Dùng **Basic monitoring** theo cấu hình workshop: CPU có dữ liệu theo chu kỳ 5 phút. Không bật Detailed monitoring chỉ để làm bài này.

EC2 tự gửi chỉ số CPU tiêu chuẩn đến CloudWatch. Không cần thêm quyền PutMetricData cho IAM Role của EC2, không cần mở cổng inbound và không cần cài Agent để xem chỉ số này. Quyền ghi log của Agent sẽ được xử lý riêng ở 5.6.3.

## Bước 1: Xác định đúng EC2

1. Đăng nhập AWS Management Console bằng tài khoản được phân công giám sát.
2. Chọn Region **Asia Pacific (Singapore) — ap-southeast-1**.
3. Tìm và mở dịch vụ **EC2**.
4. Trong menu bên trái, chọn **Instances**.
5. Tìm **workshop-web-01**, chọn dòng của máy chủ.
6. Kiểm tra **Instance state = Running** và ghi lại **Instance ID** dạng `i-...`.
7. Nếu có nhiều máy cùng Name tag, đối chiếu VPC, subnet và ID do thành viên 2 bàn giao; không chọn chỉ dựa vào tên.

![](/FCAJ_Worklog/images/55/ec2m1.png)


## Bước 2: Xem nhanh CPU trên EC2

1. Giữ nguyên instance đã chọn, mở tab **Monitoring** ở vùng thông tin bên dưới.
2. Tìm biểu đồ **CPU utilization** hoặc **CPUUtilization**.
3. Chọn khoảng thời gian **1 hour**; nếu máy đã chạy lâu và cần xem rộng hơn, chọn **3 hours**.
4. Làm mới biểu đồ và quan sát dữ liệu. Máy vừa khởi tạo cần thời gian để dữ liệu xuất hiện; không coi biểu đồ trống là CPU bằng 0.
5. Giữ Basic monitoring. Nếu Console mời bật Detailed monitoring, không bật trong bài này. Nếu trước đó đã bật, phối hợp với người quản lý EC2 trước khi thay đổi cấu hình.

![](/FCAJ_Worklog/images/55/ec2m2.png)


## Bước 3: Chọn metric trong CloudWatch

1. Mở dịch vụ **CloudWatch** trong một tab mới, giữ Region **ap-southeast-1**.
2. Trong menu trái, mở **Metrics → All metrics**.
3. Ở khu vực Browse, chọn namespace **EC2 (AWS/EC2)**.
4. Chọn nhóm dimension **Per-Instance Metrics**.
5. Dán **Instance ID thực tế** từ bước 1 vào ô tìm kiếm và nhấn Enter nếu cần.
6. Tìm dòng có **Metric name = CPUUtilization** và **InstanceId** đúng với máy chủ.
7. Tích ô ở đầu dòng để đưa metric lên đồ thị. Bỏ chọn các metric khác nếu đang được chọn.

Tên nhãn Console có thể thay đổi nhẹ. Luôn kiểm tra đủ ba thông tin: **AWS/EC2**, **CPUUtilization** và **InstanceId**, tránh chọn đồ thị gộp theo InstanceType hoặc ImageId.

## Bước 4: Cấu hình đồ thị

1. Chọn tab **Graphed metrics**.
2. Trên dòng CPUUtilization, đặt **Statistic = Average**.
3. Đặt **Period = 5 minutes (300 seconds)**. Nếu có tùy chọn Auto, chọn rõ 5 phút.
4. Chọn kiểu hiển thị **Line** nếu đang dùng kiểu khác.
5. Đặt khoảng thời gian **1 hour** hoặc **3 hours** ở phía trên đồ thị.
6. Kiểm tra múi giờ hiển thị. Có thể dùng UTC hoặc múi giờ máy cá nhân, nhưng phải ghi rõ khi lưu kết quả; giờ Việt Nam là UTC+7.
7. Chọn Refresh để cập nhật. Nếu bật tự làm mới mỗi phút, lưu ý điều đó không biến dữ liệu Basic monitoring thành dữ liệu mỗi phút.

| Thông số | Giá trị cần kiểm tra |
| --- | --- |
| Namespace | AWS/EC2 |
| Metric | CPUUtilization |
| Dimension | InstanceId = ID thực tế của workshop-web-01 |
| Statistic | Average |
| Period | 5 minutes |
| Unit | Percent (%) |
| Time range | 1 hour hoặc 3 hours, có dữ liệu mới |

![](/FCAJ_Worklog/images/55/ec2m3.png)



## Bước 5: Đọc dữ liệu và ghi nhận mốc nền

1. Di chuột lên một điểm dữ liệu đã xuất hiện trên đồ thị.
2. Đọc thời gian và giá trị CPU theo phần trăm.
3. Ghi ít nhất **ba điểm dữ liệu liên tiếp** nếu đã có; nếu chưa đủ, tiếp tục quan sát sau các lần cập nhật. Không điền số liệu giả.
4. Trong lúc quan sát, có thể đăng nhập website và tải lên/tải xuống một tài liệu mẫu nhỏ như thao tác sử dụng bình thường. Ghi lại thời gian thao tác nếu thực hiện.
5. Không yêu cầu CPU tăng rõ sau một lần tải file: website nhỏ và giá trị trung bình 5 phút có thể làm thay đổi khó nhận thấy. Tải thử có kiểm soát sẽ thực hiện ở 5.6.2.

| Thời điểm và múi giờ | CPU Average (%) | Hoạt động trên hệ thống |
| --- | --- | --- |
| Điền dữ liệu thực tế | Điền dữ liệu thực tế | Ví dụ: chưa thao tác trên website |
| Điền dữ liệu thực tế | Điền dữ liệu thực tế | Ghi hoạt động thực tế |
| Điền dữ liệu thực tế | Điền dữ liệu thực tế | Ghi hoạt động thực tế |

**Average** là giá trị trung bình trong khoảng đánh giá, không phải mức CPU tức thời của một tiến trình. Với Basic monitoring, đổi Period xuống 1 phút không tạo thêm mẫu CPU 1 phút. Công cụ trong hệ điều hành như top có thể hiển thị khác do cách đo và khoảng lấy mẫu.

CPU thấp không chứng minh website, kết nối S3 hay đăng nhập đang hoạt động đúng. Cần kết hợp kiểm thử chức năng và log ở các phần tiếp theo.



## Bước 6: Kiểm tra và bàn giao

- Xác nhận đồ thị lấy đúng Instance ID, Average và chu kỳ 5 phút.
- Lưu bốn ảnh thực tế cùng bảng ghi nhận CPU.
- Ghi lại Region, Instance ID, khoảng thời gian quan sát và múi giờ.
- Giữ thông tin metric để tạo **workshop-web-01-cpu-high** ở bước tiếp theo. Không cần tạo dashboard riêng để hoàn thành mục này.

## Lỗi thường gặp

| Hiện tượng | Cách kiểm tra và xử lý |
| --- | --- |
| Không thấy EC2 hoặc metric | Kiểm tra tài khoản, Region, Instance ID và bộ lọc đang dùng |
| Đồ thị trống khi máy vừa chạy | Chờ dữ liệu được gửi, làm mới, mở rộng time range; không tự kết luận CPU = 0 |
| Chỉ thấy dữ liệu cũ | Kiểm tra instance còn Running và time range có bao gồm hiện tại |
| Có nhiều dòng CPUUtilization | Chọn đúng dimension InstanceId; bỏ các dòng của máy khác |
| AccessDenied | Ghi action bị từ chối, nhờ quản trị bổ sung quyền đọc phù hợp; không cấp quyền quản trị toàn bộ |
| Dữ liệu không trùng giờ thao tác | Kiểm tra UTC và UTC+7; đối chiếu cùng múi giờ |
| Refresh mỗi phút nhưng giá trị ít thay đổi | Basic monitoring vẫn gửi CPU theo chu kỳ 5 phút |
| CPU thấp sau upload/download | Có thể bình thường với file nhỏ; chưa phải kiểm thử tải hoặc alarm |

## Kết quả hoàn thành

Có đồ thị CPUUtilization của đúng workshop-web-01 và dữ liệu nền thực tế.

Tiếp tục: [5.6.2. Tạo và kiểm thử cảnh báo CPU](../5.6.2-cpu-alarm/).

## Tài liệu tham khảo

- [View available metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/viewing_metrics_with_cloudwatch.html)
- [EC2 metrics and dimensions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/viewing_metrics_with_cloudwatch.html)
- [Basic and detailed monitoring](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/manage-detailed-monitoring.html)
