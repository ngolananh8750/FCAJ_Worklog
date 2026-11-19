---
title: "Kiểm thử hệ thống"
date: 2026-10-31
weight: 7
chapter: false
pre: "<b>5.7. </b>"
---

## Tổng quan

Sau khi triển khai VPC, EC2, S3 và CloudWatch, nhóm kiểm thử toàn bộ website đăng nhập, tải lên và tải xuống tài liệu. Mục tiêu là xác nhận chức năng hoạt động, truy cập ngoài phạm vi bị chặn và dữ liệu giám sát phản ánh đúng thao tác thực tế.

Phần này tổng hợp kế hoạch kiểm thử và tiêu chí đánh giá. Các bước thao tác, lệnh kiểm tra và vị trí chụp ảnh chi tiết được trình bày trong **5.7.1** và **5.7.2**. Bảng dưới là kết quả mong đợi, chưa phải kết quả đã thực hiện trên AWS.

## Mục tiêu

- Kiểm tra đăng nhập, đăng xuất, phiên truy cập và luồng tải lên/tải xuống qua HTTPS.
- Kiểm tra giới hạn truy cập mạng của Security Group và Network ACL.
- Kiểm tra quyền EC2 với prefix `portal/`, tính riêng tư và mã hóa dữ liệu trên S3.
- Đối chiếu CPU, trạng thái cảnh báo và log ứng dụng trong CloudWatch.
- Lưu bằng chứng có thời gian, tài nguyên và kết quả rõ ràng trước khi dọn dẹp.

## Chuẩn bị

1. Hoàn thành các phần 5.3–5.6, xác nhận **workshop-web-01** đang Running tại **ap-southeast-1**.
2. Ghi lại Instance ID, Public IPv4 hiện tại, VPC/subnet ID, Security Group ID, Network ACL ID, bucket thực tế và IAM Role **WorkshopEC2AppRole**.
3. Chuẩn bị URL HTTPS, tài khoản website thử nghiệm và hai tài liệu TXT/PDF nhỏ không chứa dữ liệu cá nhân. Giữ bản gốc để so sánh SHA-256 sau khi tải xuống.
4. Chuẩn bị máy quản trị có IP /32 được phép SSH và một nguồn mạng thử khác có Public IPv4 không nằm trong danh sách cho phép. Nếu chưa có nguồn thứ hai, ghi ca kiểm thử đó là **Chưa kiểm thử**.
5. Kiểm tra alarm **workshop-web-01-cpu-high**, log group **/workshop/webserver**, retention 7 ngày và Agent đang chạy. Dừng mọi tải CPU còn lại từ phần trước.
6. Thống nhất cửa sổ kiểm thử, người thay đổi cấu hình và cách hoàn tác. Lưu cấu hình gốc trước bất kỳ thay đổi tạm thời nào.

HTTPS của lab dùng chứng chỉ tự ký theo 5.4.4. Xác minh chứng chỉ/fingerprint theo hướng dẫn đó; không coi cảnh báo tin cậy của trình duyệt là lỗi firewall và không coi chứng chỉ tự ký là chứng chỉ được CA công cộng tin cậy.

## Phân công kiểm thử

| Thành viên | Phần phụ trách | Bằng chứng bàn giao |
| --- | --- | --- |
| 1 | SG, NACL, route và truy cập mạng | Quy tắc thực tế, nguồn thử và kết quả kết nối |
| 2 | HTTPS, đăng nhập, phiên và upload/download | Kết quả chức năng và đối chiếu file |
| 3 | Bucket riêng tư, SSE-S3 và IAM Role | Kết quả allow/deny, cấu hình bucket/object |
| 4 | CPU, alarm và log | Đồ thị, Alarm history, sự kiện log khớp thao tác |

## Ma trận kiểm thử dự kiến

| Mã | Ca kiểm thử | Kết quả mong đợi | Mục |
| --- | --- | --- | --- |
| WEB-01 | Chưa đăng nhập, mở trang tài liệu hoặc URL tải tài liệu | Yêu cầu đăng nhập, không trả nội dung tài liệu | 5.7.1 |
| WEB-02 | Nhập sai rồi nhập đúng thông tin đăng nhập | Sai bị từ chối; đúng truy cập được | 5.7.1 |
| WEB-03 | Upload TXT/PDF hợp lệ, không rỗng, tối đa 5 MiB | Tài liệu được lưu và xuất hiện trong danh sách | 5.7.1 |
| WEB-04 | Upload file không hợp lệ hoặc vượt giới hạn | Bị từ chối, không tạo object mới từ lần thử đó | 5.7.1 |
| WEB-05 | Download file đã upload | Đọc được, SHA-256 khớp bản gốc | 5.7.1 |
| WEB-06 | Đăng xuất rồi yêu cầu lại tài liệu | Máy chủ yêu cầu đăng nhập lại | 5.7.1 |
| NET-01 | SSH từ nguồn được phép và nguồn không được phép | Nguồn hợp lệ kết nối được; nguồn ngoài phạm vi bị chặn | 5.7.1 |
| NET-02 | Truy cập HTTPS và thử truy cập backend 8000 từ Internet | HTTPS hoạt động; backend không truy cập trực tiếp được | 5.7.1 |
| NET-03 | Kiểm thử NACL có thay đổi kiểm soát, rồi hoàn tác | Kết nối mới bị chặn/được khôi phục đúng quy tắc thử | 5.7.1 |
| S3-01 | EC2 dùng đúng instance profile để list/read/write portal/ | Các thao tác trong phạm vi thành công | 5.7.2 |
| S3-02 | List outside/ và thử xóa object thử nghiệm riêng | AccessDenied theo quyền đã cấu hình | 5.7.2 |
| S3-03 | Mở URL object thông thường không ký, không xác thực | Không đọc được tài liệu riêng tư | 5.7.2 |
| S3-04 | Kiểm tra bucket/object và đường đi S3 | Block Public Access bật, SSE-S3, route endpoint đúng | 5.7.2 |
| MON-01 | Xem CPUUtilization theo InstanceId | Có dữ liệu mới của đúng EC2, Average/5 phút | 5.7.2 |
| MON-02 | Kiểm tra bằng chứng tải CPU có giới hạn | Có ALARM và phục hồi OK phù hợp đồ thị/lịch sử | 5.7.2 |
| MON-03 | Tạo thao tác mới trên website | Log mới xuất hiện trong đúng stream, khớp thao tác | 5.7.2 |

Nếu dùng SNS, bổ sung ca xác nhận email khớp thời điểm alarm. Nếu không triển khai SNS, ghi **Không áp dụng** cho email, không ghi Pass.

## Nguyên tắc đọc kết quả

- Chỉ ghi **Pass** khi kết quả thực tế khớp yêu cầu và có bằng chứng. Ca mong đợi bị từ chối vẫn Pass nếu bị từ chối đúng nguyên nhân.
- `AccessDenied` là kết quả kiểm tra quyền; timeout, DNS lỗi hoặc thiếu credentials không thay thế được bằng chứng này.
- Kết nối thất bại không tự chứng minh firewall đúng. Đối chiếu địa chỉ nguồn, route, SG/NACL và dịch vụ đang lắng nghe; cổng không có dịch vụ cũng có thể không kết nối được.
- Kiểm thử firewall dùng kết nối mới; không dựa vào một phiên SSH đang mở. Đánh giá SG và NACL riêng, chỉ thay đổi một yếu tố trong mỗi lần thử.
- Kiểm thử cổng 80 chuyển hướng đến URL HTTPS và lưu kết quả Flow Logs theo 5.3.6.
- Không mở rộng SSH ra Internet, công khai bucket hoặc cấp AdministratorAccess để làm ca kiểm thử thành công.
- Ca thử xóa S3 chỉ dùng object dùng một lần do bài thử tạo; quyền sai có thể khiến thao tác xóa thành công. Không dùng tài liệu thật.
- Tải được file S3 chưa đủ chứng minh lưu lượng đi qua Gateway Endpoint. Cần đối chiếu endpoint, route table và subnet association; không ghi nhận bằng chứng đường đi mạnh hơn những gì đã kiểm tra.

## Trình tự thực hiện

1. Kiểm tra chức năng website và ghi nhận trạng thái ban đầu.
2. Kiểm tra mạng, mỗi thay đổi tạm thời phải được hoàn tác và kiểm tra lại HTTPS/SSH.
3. Kiểm tra S3 bằng đúng role EC2, sau đó đối chiếu bảo mật bucket/object.
4. Kiểm tra CloudWatch. Có thể dùng bằng chứng 5.6.2 nếu đúng instance, cấu hình và không có thay đổi liên quan; ghi rõ thời gian. Chỉ chạy lại tải khi cần xác nhận thay đổi hoặc xử lý kết quả chưa đạt.
5. Tạo sự kiện ứng dụng mới để kiểm tra Agent vẫn gửi log.
6. Tổng hợp kết quả, sửa lỗi, kiểm thử lại các ca bị ảnh hưởng và lưu bằng chứng trước phần 5.8.

## Mẫu bảng ghi kết quả

Mỗi ca dùng một dòng; sao chép dòng mẫu khi cần.

| Mã ca | Người thực hiện | Thời gian/múi giờ | Tài nguyên/nguồn thử | Kết quả thực tế | Trạng thái | Ảnh/log tham chiếu |
| --- | --- | --- | --- | --- | --- | --- |
| WEB-01 | Điền tên | Điền thời gian | URL/Instance ID | Điền sau khi thử | Chưa kiểm thử | Điền đường dẫn |
| NET-01 | Điền tên | Điền thời gian | IP nguồn/SG ID | Điền sau khi thử | Chưa kiểm thử | Điền đường dẫn |
| S3-01 | Điền tên | Điền thời gian | Role/bucket/prefix | Điền sau khi thử | Chưa kiểm thử | Điền đường dẫn |
| MON-03 | Điền tên | Điền thời gian | Log group/stream | Điền sau khi thử | Chưa kiểm thử | Điền đường dẫn |

Dùng các trạng thái **Pass**, **Fail**, **Chưa kiểm thử**, **Không áp dụng**. Với Fail, ghi nguyên nhân, cách sửa và lần thử lại; không xóa bằng chứng lỗi để thay bằng kết luận chung.

## Ảnh minh chứng

Ảnh chi tiết đặt ở từng mục con; trang này chỉ cần bảng tổng hợp khi hoàn tất:

| Nhóm | Bằng chứng cần lưu |
| --- | --- |
| Website | Đăng nhập, upload/download, đối chiếu SHA-256 và truy cập sau logout |
| Firewall | Quy tắc liên quan, nguồn kết nối, kết quả trước/sau và cấu hình đã hoàn tác |
| S3 | Role thực tế, kết quả allow/deny, Block Public Access và mã hóa object |
| CloudWatch | Alarm history/đồ thị, group/stream và sự kiện mới |

**📸 Ảnh tổng hợp — tùy chọn:** Bảng kết quả sau khi hoàn thành các mục con. Không điền Pass trước khi thử.

<!-- DÁN ẢNH TỔNG HỢP KẾT QUẢ KIỂM THỬ TẠI ĐÂY NẾU CẦN. -->

Không công khai mật khẩu, private key, access key, cookie hoặc token. Giữ tên tài nguyên và thời gian cần thiết để người xem đối chiếu được kết quả.

## Tiêu chí hoàn thành

Các ca bắt buộc đều Pass và có bằng chứng; ca chưa chạy phải được nêu rõ. Website hoạt động trở lại, tải CPU đã dừng, quy tắc tạm thời được hoàn tác và cấu hình bảo mật được giữ nguyên. Không tuyên bố hệ thống hoàn toàn an toàn chỉ dựa vào bộ kiểm thử lab này.

## Nội dung

- [5.7.1. Kiểm tra đăng nhập, tải lên, tải xuống và Firewall](./5.7.1-website-and-firewall/)
- [5.7.2. Kiểm tra quyền truy cập S3 và kết quả giám sát](./5.7.2-s3-and-monitoring/)

Sau khi hoàn tất: [5.8. Dọn dẹp tài nguyên](../5.8-cleanup/).
