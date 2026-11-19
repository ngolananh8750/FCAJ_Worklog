---
title: "Giới thiệu"
date: 2026-10-26
weight: 1
chapter: false
pre: "<b>5.1. </b>"
---

## Giới thiệu

Workshop **Thiết kế và triển khai hệ thống Web Server bảo mật trên AWS sử dụng VPC, EC2, S3 và CloudWatch** hướng dẫn xây dựng một website lưu trữ tài liệu đơn giản, kết hợp kiểm soát truy cập mạng, bảo vệ dữ liệu và giám sát máy chủ.

**Bài toán thực tế:** Một nhóm thực tập cần website nhỏ để lưu trữ các tài liệu học tập mẫu. Người dùng đăng nhập, tải tài liệu lên, xem danh sách và tải tài liệu xuống khi cần. Tài liệu được lưu trong Amazon S3 và không cho phép truy cập công khai trực tiếp.

Để phù hợp với phạm vi thực hành, website chỉ gồm **trang đăng nhập** và **trang tài liệu**, sử dụng một tài khoản dùng chung được tạo sẵn. Nhóm không triển khai đăng ký tài khoản, quên mật khẩu, phân quyền nhiều người dùng hay cơ sở dữ liệu riêng. Tài khoản dùng chung chỉ phục vụ minh họa trong workshop; dữ liệu sử dụng là các tệp mẫu không chứa thông tin cá nhân.

Sau khi hoàn thành workshop, người thực hiện có thể:

- Thiết lập môi trường mạng VPC và kiểm soát lưu lượng bằng Security Group, Network ACL.
- Triển khai ứng dụng web trên EC2 với chức năng đăng nhập, tải lên và tải xuống tài liệu.
- Lưu tài liệu trong S3 bucket riêng tư, cấp quyền cho EC2 bằng IAM Role.
- Theo dõi chỉ số CPU, cấu hình cảnh báo và thu thập nhật ký ứng dụng bằng CloudWatch.
- Kiểm thử các trường hợp được phép và bị từ chối truy cập, sau đó dọn dẹp tài nguyên.

## Tổng quan

Hệ thống sử dụng bốn dịch vụ chính, mỗi dịch vụ đảm nhận một vai trò trong cùng bài toán:

| Dịch vụ | Vai trò trong hệ thống | Nội dung thực hành |
| --- | --- | --- |
| **Amazon VPC** | Tạo môi trường mạng cho máy chủ web. | Tạo VPC, Public Subnet, Internet Gateway và Route Table; cấu hình Security Group, Network ACL và S3 Gateway VPC Endpoint. |
| **Amazon EC2** | Chạy ứng dụng web và xử lý yêu cầu của người dùng. | Triển khai đăng nhập, quản lý phiên, xem danh sách tài liệu, tải lên và tải xuống; cấu hình HTTPS. |
| **Amazon S3** | Lưu trữ tài liệu của website. | Tạo bucket riêng tư, bật Block Public Access, kiểm tra mã hóa và cấp quyền đọc, ghi tối thiểu cho EC2. |
| **Amazon CloudWatch** | Theo dõi hoạt động máy chủ và hỗ trợ kiểm tra lỗi. | Xem chỉ số CPU, tạo cảnh báo và cấu hình thu thập nhật ký ứng dụng. |

**Phạm vi chức năng:**

- Đăng nhập bằng tài khoản được tạo sẵn và đăng xuất khi sử dụng xong.
- Chỉ cho phép người đã đăng nhập xem danh sách, tải lên và tải xuống tài liệu.
- Giới hạn tệp mẫu ở định dạng PDF hoặc TXT, dung lượng tối đa 5 MB mỗi tệp.
- Không cung cấp chức năng xóa tài liệu trên website trong phạm vi bài thực hành.

**Trọng tâm bảo mật:**

- Security Group kiểm soát lưu lượng ở cấp tài nguyên và có trạng thái; Network ACL kiểm soát lưu lượng ở cấp subnet và không có trạng thái. Khi cấu hình Network ACL cần cho phép cả lưu lượng phản hồi phù hợp.
- Chỉ cho phép SSH từ địa chỉ IP quản trị được chỉ định. Truy cập ứng dụng sử dụng HTTPS; HTTP chỉ phục vụ chuyển hướng sang HTTPS nếu được cấu hình.
- Ứng dụng lưu mật khẩu dưới dạng băm, kiểm tra phiên đăng nhập trước từng thao tác với tài liệu và không ghi mật khẩu hoặc mã phiên vào nhật ký.
- EC2 sử dụng IAM Role để truy cập S3, không lưu Access Key cố định trong mã nguồn. Quyền được giới hạn ở thao tác liệt kê, đọc và ghi trong phạm vi tài liệu cần dùng.
- Bucket S3 được giữ riêng tư, bật Block Public Access và mã hóa dữ liệu lưu trữ. Ứng dụng kiểm tra loại tệp, dung lượng và xử lý tên tệp an toàn trước khi lưu.

**Phân công:** Thành viên 1 phụ trách VPC và Firewall; thành viên 2 phụ trách EC2 và ứng dụng web; thành viên 3 phụ trách S3 và bảo mật dữ liệu; thành viên 4 phụ trách CloudWatch và giám sát. Cả bốn thành viên sử dụng chung kiến trúc và phối hợp kiểm thử trên cùng hệ thống.

Để giữ quy mô nhỏ, kiến trúc chỉ sử dụng một EC2 trong một Public Subnet, không triển khai NAT Gateway, Load Balancer hoặc cơ sở dữ liệu riêng. Nhóm giới hạn dung lượng tài liệu, thời gian lưu nhật ký và dọn dẹp tài nguyên sau bài thực hành. Đây là mô hình học tập, chưa có khả năng dự phòng máy chủ hoặc tính sẵn sàng cao.

## Kiến trúc tổng quan

Hệ thống được triển khai trong một AWS Region. EC2 nằm trong Public Subnet thuộc VPC; S3 và CloudWatch là các dịch vụ được EC2 truy cập, không nằm bên trong Public Subnet.

![Sơ đồ kiến trúc](/FCAJ_Worklog/images/5/so_do_kien_truc_aws.drawio.png?featherlight=false&width=90pc)

**Các thành phần và kết nối:**

| Thành phần | Vị trí và kết nối |
| --- | --- |
| **Người dùng** | Truy cập ứng dụng web trên EC2 qua HTTPS. |
| **Internet Gateway và Route Table** | Public Subnet có tuyến mặc định đến Internet Gateway; EC2 sử dụng địa chỉ IPv4 công cộng để giao tiếp với Internet theo các quy tắc mạng được cho phép. |
| **Network ACL và Security Group** | Network ACL gắn với subnet; Security Group gắn với giao diện mạng của EC2, kiểm soát lưu lượng vào và ra. |
| **EC2 và IAM Role** | EC2 chạy ứng dụng, xác thực người dùng và thực hiện thao tác với S3 bằng quyền của IAM Role. |
| **S3 Gateway VPC Endpoint** | Liên kết với Route Table của subnet để lưu lượng EC2 đến S3 trong cùng Region đi qua endpoint. Endpoint không thay thế quyền IAM hoặc chính sách bucket. |
| **S3 bucket riêng tư** | Lưu các tài liệu mẫu; người dùng thao tác thông qua ứng dụng trên EC2. |
| **CloudWatch** | Nhận chỉ số giám sát EC2; nhận nhật ký ứng dụng khi đã cấu hình agent và quyền gửi log phù hợp. Kết nối gửi log sử dụng HTTPS đến endpoint CloudWatch qua Internet Gateway trong mô hình này. |

**Luồng hoạt động của hệ thống:**

1. Người dùng mở website qua HTTPS và nhập thông tin đăng nhập.
2. Ứng dụng trên EC2 kiểm tra thông tin; nếu hợp lệ, tạo phiên đăng nhập và hiển thị trang tài liệu.
3. Khi người dùng xem danh sách, ứng dụng kiểm tra phiên và sử dụng IAM Role để liệt kê các tài liệu được phép trong S3.
4. Khi tải lên, ứng dụng kiểm tra phiên, loại tệp, dung lượng và tên tệp, sau đó ghi tài liệu vào S3 qua S3 Gateway VPC Endpoint.
5. Khi tải xuống, ứng dụng kiểm tra phiên, đọc tài liệu từ S3 và truyền tệp về trình duyệt qua HTTPS. Website không yêu cầu mở công khai bucket.
6. CloudWatch phục vụ theo dõi CPU, kiểm tra cảnh báo và xem nhật ký ứng dụng đã được cấu hình thu thập.
7. Khi đăng xuất, phiên được vô hiệu hóa; các yêu cầu tiếp theo đến chức năng tài liệu phải đăng nhập lại.

**Kết quả cần chứng minh:** Người đăng nhập hợp lệ tải lên và tải xuống được tài liệu; người chưa đăng nhập bị từ chối truy cập chức năng tài liệu; truy cập công khai trực tiếp đến tệp S3 bị từ chối; CloudWatch hiển thị dữ liệu giám sát theo cấu hình.

