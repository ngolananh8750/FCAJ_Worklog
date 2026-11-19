---
title: "Tạo bucket riêng tư và kiểm tra mã hóa"
date: 2026-10-31
weight: 1
chapter: false
pre: "<b>5.5.1. </b>"
---

## Mục tiêu

Thành viên 3 tạo bucket S3 riêng tư tại **Singapore — ap-southeast-1**, bật Block Public Access và xác nhận mã hóa mặc định **SSE-S3**. Bucket này sẽ chứa tài liệu của ứng dụng trong prefix portal/.

Phần này kiểm tra cấu hình bucket. Mã hóa của object thực tế sẽ được kiểm tra sau khi tải tài liệu mẫu ở 5.5.2; không coi cấu hình mặc định là bằng chứng cho một object chưa tồn tại.

## Chuẩn bị

- Có tài khoản được cấp quyền tạo bucket và cấu hình bảo mật theo 5.2.1; không mặc định policy chỉ đọc/ghi object đủ quyền tạo bucket.
- Thống nhất tên thực tế theo 5.2.2: **secureweb-docs-683699-280958772766**.

- Nếu bucket đã tồn tại trong tài khoản, kiểm tra Region, chủ sở hữu và cấu hình rồi dùng lại đúng tài nguyên của nhóm. Không tạo thêm bucket chỉ vì không thấy do thiếu quyền.

Tên bucket không chứa mật khẩu hoặc bí mật. Với không gian tên dùng chung, tên phải khả dụng trên toàn bộ AWS partition tương ứng, không chỉ trong tài khoản. Nếu Console có lựa chọn namespace, bài lab này sử dụng **General purpose với shared global namespace** để giữ mẫu tên đã thống nhất; không chuyển sang tên tự sinh khác mà chưa cập nhật ứng dụng.

Nếu dùng lại bucket, kiểm tra Versioning ở **Properties → Bucket Versioning** và bật nếu cần. Kiểm tra các tùy chọn bảo mật ở bước 4.

## Bước 1: Nhập thông tin bucket

1. Đăng nhập AWS Management Console bằng tài khoản phụ trách S3.
2. Mở **Amazon S3 → Buckets → Create bucket**.
3. Ở **AWS Region**, chọn **Asia Pacific (Singapore) — ap-southeast-1**. S3 Console có thể hiển thị nhiều Region; phải kiểm tra Region trong biểu mẫu tạo bucket.
4. Chọn **Bucket type: General purpose** nếu biểu mẫu có trường này; không chọn Directory bucket/S3 Express One Zone.
5. Nhập tên bucket thực tế đã thống nhất.
6. Không sao chép cấu hình từ bucket không thuộc workshop.

![](/FCAJ_Worklog/images/55/b1.png)


## Bước 2: Giữ bucket riêng tư

1. Trong **Object Ownership**, giữ **ACLs disabled (recommended)**, tương ứng **Bucket owner enforced**.
2. Trong **Block Public Access settings for this bucket**, giữ **Block all public access** được chọn.
3. Kiểm tra cả bốn tùy chọn chặn public đều bật. Không bỏ chọn để làm website hoạt động.

Website ở EC2 đọc/ghi S3 bằng IAM Role; người dùng tải tài liệu qua backend nên bucket không cần public. ACL bị tắt giúp quản lý quyền bằng policy thay vì cấp quyền riêng qua ACL object.

Block Public Access không thay thế IAM và không tự loại bỏ mọi quyền truy cập có xác thực hoặc quyền liên tài khoản. Các policy vẫn phải giới hạn đúng phạm vi ở 5.5.3.

![](/FCAJ_Worklog/images/55/b2.png)



## Bước 3: Chọn mã hóa và tùy chọn lab

| Thuộc tính | Giá trị |
| --- | --- |
| Bucket Versioning | Enable |
| Tags | Project=SecureWebWorkshop |
| Default encryption | Server-side encryption with Amazon S3 managed keys (SSE-S3) |
| Object Lock | Không bật |

Chọn **SSE-S3**, không chọn SSE-KMS hoặc DSSE-KMS. Không tạo khóa KMS riêng. Nếu giao diện không có trường S3 Bucket Key khi dùng SSE-S3, đó là bình thường; trường này liên quan SSE-KMS.

Bật Versioning để thực hành phục hồi ở 5.5.4. Khi bật lần đầu, đợi 15 phút trước khi ghi hoặc xóa object.

![](/FCAJ_Worklog/images/55/b3.png)



## Bước 4: Tạo bucket và kiểm tra lại

1. Kiểm tra Region, tên, Block Public Access và SSE-S3.
2. Chọn **Create bucket**.
3. Mở bucket vừa tạo; ghi tên chính xác và ARN bucket từ **Properties**.
4. Mở **Permissions**: xác nhận Block Public Access vẫn bật và Object Ownership là Bucket owner enforced.
5. Kiểm tra **Bucket policy**: ở bucket mới, chưa cần thêm policy cấp quyền. Không dán policy Allow với Principal * để ứng dụng truy cập được.
6. Mở **Properties → Default encryption**, xác nhận SSE-S3. Kiểm tra **Static website hosting** ở trạng thái Disabled.

Nếu dùng lại bucket có policy, đọc toàn bộ policy với người quản trị trước khi sửa; không xóa policy hiện có. Policy chỉ cho phép HTTPS và quyền EC2 sẽ được xử lý cùng phần phân quyền, không làm bucket công khai.

![](/FCAJ_Worklog/images/55/b4.png)


## Bước 5: Bàn giao và giới hạn kiểm tra

| Nội dung | Kết quả cần đạt |
| --- | --- |
| Tên và Region | Đúng tên thực tế, ap-southeast-1 |
| Chủ sở hữu | Đúng tài khoản AWS của nhóm |
| Public access | Bốn tùy chọn Block Public Access đều bật |
| Object Ownership | Bucket owner enforced / ACLs disabled |
| Default encryption | SSE-S3 |
| Hosting | Static website hosting Disabled |
| Bàn giao | Tên bucket, ARN dạng arn:aws:s3:::TEN_BUCKET và Region |

Thành viên 2 đối chiếu tên bucket với /etc/document-portal/config.json; không chụp hoặc đưa file cấu hình chứa password hash vào báo cáo. Chưa tạo policy rộng nếu ứng dụng hiện AccessDenied: quyền EC2 sẽ hoàn thiện ở 5.5.3.

Chưa thử URL object khi bucket đang trống. Một lỗi 403 trên URL bucket không đủ chứng minh toàn bộ chính sách an toàn; cần kiểm tra cấu hình và kiểm thử object cụ thể sau. SSE-S3 mã hóa khi lưu trữ; vẫn phải dùng HTTPS khi truyền dữ liệu. Thiết lập mã hóa mặc định không phải thao tác mã hóa lại toàn bộ object cũ trong bucket dùng lại.

## Lỗi thường gặp

| Hiện tượng | Cách xử lý |
| --- | --- |
| Tên bucket đã được sử dụng | Nếu không thuộc nhóm, đổi hậu tố; cập nhật tên thống nhất và cấu hình ứng dụng |
| Invalid bucket name | Dùng chữ thường, số và dấu gạch ngang theo mẫu; không dùng dấu cách, chữ hoa hoặc placeholder |
| AccessDenied khi tạo/lưu cấu hình | Ghi action bị từ chối, nhờ quản trị kiểm tra quyền tạo bucket, public access block, ownership, encryption và tag |
| Region sai sau khi tạo | Region của bucket không đổi trực tiếp được; thống nhất tạo bucket đúng Region và xử lý tài nguyên nhầm, không xóa dữ liệu tùy tiện |
| Không thấy Bucket Key | Với SSE-S3 không cần cấu hình tùy chọn này |
| Ứng dụng vẫn báo 503 | Bucket có thể đã đúng nhưng IAM Role chưa có quyền; thực hiện 5.5.3 |

## Kết quả hoàn thành

Đã tạo bucket riêng tư, kiểm tra ownership, public access và mã hóa mặc định.

Tiếp tục: [5.5.2. Chuẩn bị tài liệu mẫu và kiểm tra lưu trữ trên S3](../5.5.2-upload-documents/).
