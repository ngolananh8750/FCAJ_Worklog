---
title: "Lưu trữ và bảo mật dữ liệu trên S3"
date: 2026-10-31
weight: 5
chapter: false
pre: "<b>5.5. </b>"
---

## Tổng quan

Trong phần này, nhóm sử dụng **Amazon S3** để lưu trữ tài liệu của website đã triển khai trên EC2. Người dùng đăng nhập vào ứng dụng để tải lên và tải xuống; bucket S3 giữ riêng tư, không cung cấp tài liệu công khai qua URL trực tiếp.

**Thành viên 3** phụ trách tạo bucket, chuẩn bị tài liệu mẫu và kiểm tra bảo mật dữ liệu. Thành viên 2 phối hợp cấu hình ứng dụng và kiểm thử quyền của EC2; thành viên 1 hỗ trợ S3 Gateway Endpoint và mạng. Việc tạo hoặc sửa IAM Role phải do người được cấp quyền IAM thực hiện, không mặc định tài khoản phụ trách S3 có quyền này.

Trang này giới thiệu phạm vi và thứ tự triển khai. Hướng dẫn thao tác chi tiết nằm trong ba mục con bên dưới.

## Mục tiêu

- Tạo bucket riêng tư tại cùng Region với EC2 và S3 Gateway Endpoint.
- Giữ Block Public Access bật, sử dụng mã hóa phía máy chủ SSE-S3.
- Tổ chức tài liệu trong prefix portal/ và chuẩn bị tệp thực hành phù hợp với ứng dụng.
- Cấp cho EC2 quyền liệt kê, đọc và ghi đúng phạm vi cần thiết bằng IAM Role.
- Kiểm tra upload/download qua website, đồng thời xác nhận truy cập không được phép bị từ chối.

## Chuẩn bị

| Thành phần | Thông tin cần đối chiếu |
| --- | --- |
| Region | Singapore — ap-southeast-1 |
| EC2 | workshop-web-01 |
| IAM Role của EC2 | WorkshopEC2AppRole |
| Instance profile | WorkshopEC2AppRole hoặc tên thực tế đã xác nhận ở 5.4.1 |
| S3 Gateway Endpoint | s3-gateway-endpoint, liên kết rtb-web-public |
| Bucket dự kiến | secureweb-docs-683699-280958772766 |
| Prefix tài liệu | portal/ |
| Ứng dụng | document-portal, chạy bằng Gunicorn tại 127.0.0.1:8000 |
| Kết nối người dùng | HTTPS qua Nginx theo 5.4.4 |

Dùng bucket đã ghi ở 5.2.2 và bật Versioning để thực hành phục hồi ở 5.5.4.

Hoàn thành [5.3.5. Tạo S3 Gateway VPC Endpoint](../5.3-network-and-firewall/5.3.5-s3-endpoint/) và phối hợp với [5.4. Triển khai Web Server trên EC2](../5.4-ec2-web-server/). Có thể tạo bucket trước khi ứng dụng hoàn tất, nhưng kiểm thử toàn bộ luồng cần cả HTTPS và IAM Role sẵn sàng.

## Cấu hình lưu trữ thống nhất

| Thuộc tính | Cấu hình cho workshop |
| --- | --- |
| Loại bucket | General purpose |
| Block Public Access | Bật toàn bộ tùy chọn chặn truy cập công khai |
| Object Ownership | Bucket owner enforced, ACLs disabled |
| Mã hóa mặc định | SSE-S3, khóa do Amazon S3 quản lý |
| Storage class của tệp mẫu | S3 Standard |
| Static website hosting | Không sử dụng |
| Dữ liệu thực hành | PDF hoặc TXT UTF-8, không chứa thông tin cá nhân/bí mật |
| Giới hạn ứng dụng | Tối đa 5 MiB mỗi tệp, tương ứng 5 × 1024 × 1024 byte |

SSE-S3 phù hợp phạm vi lab hiện tại; không bổ sung khóa KMS riêng trong phần này. Mã hóa dữ liệu lưu trữ không thay thế việc kiểm soát quyền truy cập. Prefix portal/ là phần đầu của object key, không phải thư mục hệ điều hành có quyền riêng.

Để ứng dụng mẫu ở 5.4.3 hiển thị đúng tài liệu đưa lên trực tiếp từ Console, dùng tên đơn giản như **huong-dan.txt** hoặc **tai-lieu.pdf**, phần mở rộng chữ thường và không tạo thư mục con trong portal/. Ứng dụng tự thêm chuỗi ngẫu nhiên vào tên khi upload để tránh ghi đè.

## Luồng truy cập và các lớp bảo mật

1. Người dùng gửi yêu cầu qua HTTPS đến ứng dụng trên EC2.
2. Ứng dụng kiểm tra phiên đăng nhập và tính hợp lệ của tệp/yêu cầu.
3. Boto3 sử dụng thông tin xác thực tạm thời từ IAM Role của EC2 để gọi S3.
4. Lưu lượng S3 phù hợp tuyến trong cùng Region đi qua Gateway Endpoint.
5. S3 đánh giá các chính sách liên quan; ứng dụng chỉ trả tài liệu cho người dùng khi yêu cầu được phép.

Không lưu access key dài hạn trong mã nguồn hoặc cấu hình website. Không tắt Block Public Access để xử lý lỗi AccessDenied. Endpoint tạo đường truy cập mạng, không tự cấp quyền đọc/ghi bucket; SSE-S3 bảo vệ dữ liệu lưu trữ, không cấp quyền cho người dùng website.

## Phạm vi quyền của ứng dụng

| Tác vụ | Quyền cần thiết | Phạm vi |
| --- | --- | --- |
| Liệt kê tài liệu | s3:ListBucket | ARN bucket, giới hạn điều kiện s3:prefix cho portal/ |
| Tải tài liệu xuống | s3:GetObject | ARN object dưới portal/* |
| Tải tài liệu lên | s3:PutObject | ARN object dưới portal/* |

Ứng dụng không cần s3:DeleteObject, quyền quản trị bucket hoặc quyền S3 toàn tài khoản. Chính sách chi tiết và kiểm thử giới hạn sẽ nằm ở 5.5.3. Quyền quản trị viên dùng để tạo bucket/dọn dẹp tách biệt với quyền chạy ứng dụng.

Bucket policy có thể bổ sung yêu cầu dùng HTTPS. Nếu thêm điều kiện bắt buộc qua một VPC Endpoint, phải đánh giá luồng quản trị và dịch vụ trước vì có thể chặn Console ngoài endpoint. Không tự thêm điều kiện này vào bucket đang hoạt động mà chưa kiểm tra đường truy cập và phương án khôi phục.

## Nội dung thực hiện

- [5.5.1. Tạo bucket riêng tư và kiểm tra mã hóa](./5.5.1-private-bucket/)
- [5.5.2. Chuẩn bị tài liệu mẫu và kiểm tra lưu trữ trên S3](./5.5.2-upload-documents/)
- [5.5.3. Cấp quyền IAM Role và kiểm tra EC2 đọc, ghi tài liệu trên S3](./5.5.3-ec2-access/)

Sau khi tạo bucket, đối chiếu tên trong /etc/document-portal/config.json với bucket thực tế mà không công khai nội dung file chứa password hash. Nếu sửa cấu hình, khởi động lại document-portal; các phiên đăng nhập trong RAM sẽ bị mất hiệu lực và người dùng đăng nhập lại.

## Kết quả cần đạt

| Hạng mục | Bằng chứng cần có |
| --- | --- |
| Bucket riêng tư | Block Public Access bật, không có cấp quyền public |
| Mã hóa | Cấu hình SSE-S3 và thông tin mã hóa của object mẫu |
| Dữ liệu | Tệp PDF/TXT nằm đúng portal/ |
| Quyền EC2 | Liệt kê, tải lên, tải xuống trong phạm vi cho phép |
| Giới hạn quyền | Thao tác ngoài phạm vi bị từ chối theo ca kiểm thử ở 5.5.3 |
| Website | Người dùng đăng nhập upload/download được; yêu cầu chưa đăng nhập bị chặn |



Bắt đầu: [5.5.1. Tạo bucket riêng tư và kiểm tra mã hóa](./5.5.1-private-bucket/).
- [5.5.4. Versioning, Lifecycle và phục hồi](./5.5.4-versioning-lifecycle-recovery/)
