---
title: "Quản lý vòng đời và phục hồi phiên bản tài liệu"
weight: 4
chapter: false
pre: "<b>5.5.4. </b>"
---

## Tạo hai phiên bản của cùng một object

1. Dùng `s3` chuyển sang `WorkshopStorageRole`. Mở bucket lab và kiểm tra **Properties → Bucket Versioning → Enabled**.
2. Upload hai tệp TXT mẫu có cùng tên `recovery-test.txt`, nội dung lần lượt là `version 1` và `version 2`, vào đúng prefix `portal/`. Thực hiện hai lượt upload.
3. Bật **Show versions** và lọc theo `portal/recovery-test.txt`. Ghi hai Version ID, tải bản cũ và đối chiếu nội dung.



## Phục hồi bản cũ

Tải phiên bản 1 theo Version ID rồi upload lại đúng key. S3 tạo một phiên bản mới chứa nội dung `version 1`. Tải tệp hiện hành qua website, kiểm tra nội dung và chụp kết quả. Thao tác bằng role lưu trữ. Ứng dụng chỉ đọc và ghi object hiện hành trong `portal/` theo 5.5.3.



## Cấu hình Lifecycle cho lab

Trong **Management → Create lifecycle rule**, dùng tên `portal-expire-noncurrent`, lọc prefix `portal/`, chọn **Permanently delete noncurrent versions of objects**, nhập **30 ngày** tính từ lúc phiên bản trở thành noncurrent. Giữ current version và chụp phần Review. Quy tắc áp dụng cho cả phiên bản cũ đã đủ điều kiện và chạy bất đồng bộ. Thực hiện demo phục hồi bằng tệp mẫu trước khi thử với dữ liệu cần giữ.



[Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/versioning-workflows.html) · [Lifecycle](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html)
