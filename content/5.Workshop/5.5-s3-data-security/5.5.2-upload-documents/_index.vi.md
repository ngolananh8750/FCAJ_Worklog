---
title: "Chuẩn bị tài liệu mẫu và kiểm tra lưu trữ trên S3"
date: 2026-10-31
weight: 2
chapter: false
pre: "<b>5.5.2. </b>"
---

## Mục tiêu

Thành viên 3 chuẩn bị tài liệu mẫu, tải vào prefix **portal/** của bucket riêng tư, kiểm tra mã hóa của object và tải lại để đối chiếu nội dung. Phần này thao tác qua **S3 Console bằng tài khoản quản trị dữ liệu**; quyền của EC2 và luồng website sẽ kiểm tra riêng ở 5.5.3.

## Chuẩn bị

- Hoàn thành [5.5.1. Tạo bucket riêng tư và kiểm tra mã hóa](../5.5.1-private-bucket/).
- Dùng đúng bucket của nhóm tại **ap-southeast-1**; Block Public Access vẫn bật, ACLs disabled và mã hóa mặc định SSE-S3.
- Tài khoản thao tác có quyền liệt kê bucket, tải lên, đọc object và xem cấu hình mã hóa trong phạm vi lab. Nếu thiếu quyền Console, ghi lỗi để quản trị bổ sung đúng action.
- Tạo thư mục `s3-sample-documents` trên máy cá nhân để lưu bản gốc và đối chiếu sau khi tải xuống.

## Bước 1: Tạo tài liệu mẫu

Chuẩn bị hai tệp nhỏ, không chứa thông tin cá nhân hoặc bí mật:

| Tệp | Cách tạo | Yêu cầu |
| --- | --- | --- |
| huong-dan.txt | Notepad/VS Code, lưu UTF-8 | Không rỗng, phần mở rộng .txt thật |
| tai-lieu.pdf | Word hoặc trình soạn thảo → Export/Save as PDF | PDF hợp lệ, mở đọc được |

Nội dung TXT có thể dùng:

```text
Tai lieu thuc hanh AWS Secure Web Workshop.
Chuc nang: dang nhap, tai len va tai xuong tai lieu.
Du lieu nay chi dung de kiem thu, khong chua thong tin ca nhan.
```

PDF có thể chứa một trang mô tả tên workshop, các dịch vụ VPC/EC2/S3/CloudWatch và mục tiêu kiểm thử. **Không đổi đuôi .txt thành .pdf** để tạo PDF.

Mỗi tệp tối đa **5 MiB = 5 × 1024 × 1024 byte** theo ứng dụng ở 5.4.3; nên dùng tệp dưới 100 KB cho bài thử. S3 Console có thể cho tải tệp lớn hơn, nhưng ứng dụng vẫn sẽ từ chối xử lý tệp vượt giới hạn. Dùng tên ASCII như bảng, phần mở rộng chữ thường, không đặt trong thư mục con của portal/ để tương thích mã mẫu.

Trên Windows, bật **View → Show → File name extensions** trong File Explorer để tránh tên huong-dan.txt.txt. Ghi kích thước từng tệp và giữ nguyên bản gốc.




## Bước 2: Tạo prefix portal/

1. Mở **Amazon S3 → Buckets → bucket của nhóm → Objects**.
2. Chọn **Create folder**, nhập **portal** và giữ cấu hình mã hóa mặc định phù hợp; chọn **Create folder**.
3. Mở **portal/** trước khi tải tệp. Nếu prefix đã có, mở trực tiếp, không tạo lồng portal/portal/.

S3 lưu object theo key; “folder” là cách Console biểu diễn prefix. Thao tác Create folder có thể tạo object đánh dấu kích thước 0 byte với key portal/. Đây không phải tài liệu mẫu và ứng dụng không hiển thị nó như một tệp tải xuống.

![](/FCAJ_Worklog/images/55/ud1.png)


## Bước 3: Tải tệp lên S3

1. Khi đang ở **portal/**, chọn **Upload → Add files**.
2. Chọn huong-dan.txt và tai-lieu.pdf từ máy cá nhân.
3. Kiểm tra Destination là bucket đúng với prefix **portal/**, không phải gốc bucket.
4. Trong Properties, giữ **Storage class: S3 Standard**.
5. Nếu có phần Server-side encryption, chọn dùng cấu hình bucket mặc định (SSE-S3); không chọn KMS hoặc khóa do khách hàng cung cấp.
6. Không thêm ACL public hoặc tùy chọn Make public. Kiểm tra danh sách rồi chọn **Upload**.
7. Chờ cả hai tệp báo thành công, chọn **Close** và trở lại danh sách Objects.

Dùng tên tệp mẫu mới khi cần giữ các bản demo riêng. Ghi cùng key trong bucket đã bật Versioning tạo một phiên bản mới.




## Bước 4: Kiểm tra object và mã hóa

1. Trong portal/, mở **huong-dan.txt → Properties**.
2. Ghi key **portal/huong-dan.txt**, kích thước, thời gian Last modified và Storage class.
3. Tìm **Server-side encryption settings**, xác nhận dùng **SSE-S3 / Amazon S3 managed keys**.
4. Lặp lại với tai-lieu.pdf, key phải là **portal/tai-lieu.pdf**.
5. Kiểm tra lại tab Permissions của bucket nếu cần: Block Public Access vẫn bật và ACLs disabled.

Mã hóa mặc định ở bucket và mã hóa thực tế của object là hai bằng chứng khác nhau. Nếu object hiển thị KMS, đối chiếu lựa chọn khi upload và lịch sử object; không cấp thêm quyền KMS rộng để bỏ qua sự khác biệt cấu hình.




## Bước 5: Tải lại và so sánh với bản gốc

1. Chọn từng object trong Console, dùng **Download** và lưu vào thư mục riêng, ví dụ `s3-download-check`.
2. Mở TXT và PDF đã tải để kiểm tra đọc được, nội dung giống bản gốc.
3. So sánh SHA-256 của bản gốc và bản tải về. Chạy lệnh trên **máy cá nhân**; sửa đường dẫn theo vị trí thực tế.

**Windows PowerShell:**

```powershell
Get-FileHash -Algorithm SHA256 -Path ".\s3-sample-documents\huong-dan.txt", ".\s3-download-check\huong-dan.txt"
Get-FileHash -Algorithm SHA256 -Path ".\s3-sample-documents\tai-lieu.pdf", ".\s3-download-check\tai-lieu.pdf"
```

**Linux:**

```bash
sha256sum s3-sample-documents/huong-dan.txt s3-download-check/huong-dan.txt
sha256sum s3-sample-documents/tai-lieu.pdf s3-download-check/tai-lieu.pdf
```

Trên macOS, thay `sha256sum` bằng `shasum -a 256`. Hai hash của cùng một tài liệu phải giống nhau. Không dùng ETag như SHA-256: ETag không phải lúc nào cũng là checksum nội dung theo cách bạn mong đợi.

Download từ Console sử dụng quyền của người đã đăng nhập và có thể dùng URL ký sẵn; thành công không có nghĩa object public. Không chia sẻ URL tải có chữ ký hoặc chụp tham số chữ ký vào báo cáo.




## Bước 6: Kiểm tra truy cập ẩn danh và bàn giao

Trong chi tiết object, sao chép **Object URL thông thường**, không dùng URL từ nút Open/Download hoặc presigned URL. Với tên tệp đơn giản, URL có dạng:

```text
https://TEN_BUCKET.s3.ap-southeast-1.amazonaws.com/portal/huong-dan.txt
```

Thay TEN_BUCKET bằng tên thực tế. Mở URL trong cửa sổ ẩn danh không mang thông tin xác thực AWS. Mong đợi **AccessDenied / HTTP 403**, không phải nội dung tài liệu. Nếu có thể đọc công khai, dừng và kiểm tra Block Public Access cùng policy; không coi đó là kết quả thành công.

Kết quả 403 chỉ là một ca kiểm thử truy cập không ký, không chứng minh mọi chính sách đều an toàn. Luôn đối chiếu object tồn tại và đường dẫn đúng, tránh nhầm 403 của key sai với bằng chứng bảo vệ đúng tệp.




Bàn giao tên bucket, Region, hai object key, kích thước và SHA-256. Giữ bản gốc để thành viên 2 so sánh khi kiểm thử tải qua website. Không sửa quyền public để ứng dụng thấy tài liệu; thực hiện IAM Role ở 5.5.3.

## Lỗi thường gặp

| Hiện tượng | Cách xử lý |
| --- | --- |
| Tệp nằm ở gốc bucket hoặc portal/portal/ | Kiểm tra Destination; tải lại đúng prefix sau khi xác nhận không ghi đè, chỉ xóa bản đặt nhầm khi được phép |
| AccessDenied khi upload/download | Kiểm tra IAM user, bucket policy, quyền prefix và explicit deny; không tắt Block Public Access |
| Website không hiển thị tệp | Kiểm tra tên ASCII, phần mở rộng .pdf/.txt chữ thường, key trực tiếp dưới portal/ và quyền EC2 |
| Hash không khớp | Kiểm tra đã tải đúng object/version, đúng thư mục và bản gốc không bị sửa; tải lại để đối chiếu |
| Console upload được tệp >5 MiB | Console không thực thi giới hạn của ứng dụng; dùng lại mẫu nhỏ phù hợp |

## Kết quả hoàn thành

Hai tài liệu mẫu đã lưu đúng portal/, có SSE-S3, tải lại khớp SHA-256 và không đọc được bằng yêu cầu ẩn danh.

Tiếp tục: [5.5.3. Cấp quyền IAM Role và kiểm tra EC2 đọc, ghi tài liệu trên S3](../5.5.3-ec2-access/).
