---
title: "Bản đề xuất"
weight: 2
date: "` Sys.Date()`"
chapter: false
pre : "<b> 2 </b>"
---


## Kiến trúc Web Server bảo mật phân tầng và vận hành tối ưu chi phí trên AWS

---

### 1. Tóm tắt điều hành

Đề xuất này trình bày giải pháp kiến trúc và triển khai hệ thống **Web Server cơ bản nhưng đạt tiêu chuẩn bảo mật doanh nghiệp (Production-Ready)** trên hạ tầng đám mây Amazon Web Services (AWS). Thay vì tạo một máy chủ ảo EC2 đơn lẻ nằm trong dải mạng mặc định tiềm ẩn nhiều rủi ro bảo mật và dễ phát sinh chi phí ngoài kiểm soát, hệ thống áp dụng mô hình phân vùng mạng an toàn, cô lập môi trường tính toán và tự động hóa quy trình quản lý.

Toàn bộ hạ tầng được xây dựng dựa trên mạng ảo riêng **AWS VPC** với phân vùng Public/Private Subnet rõ ràng. Máy chủ **Amazon EC2** sử dụng hệ điều hành Linux (Ubuntu 22.04 LTS / Amazon Linux 2023) chạy **Nginx Web Server**, tích hợp mã hóa lưu lượng toàn bộ bằng chứng chỉ **SSL/TLS (HTTPS)** từ Let's Encrypt. Hệ thống áp dụng nguyên tắc phân quyền tối thiểu (Least Privilege) với **IAM Role**, bảo mật cổng truy cập mạng qua **Security Group** và tự động hóa triển khai mã nguồn qua **GitHub Actions CI/CD Pipeline**. Toàn bộ giải pháp được thiết kế tối ưu tuyệt đối để vận hành trong giới hạn **AWS Free Tier** (chi phí $\sim 0$ USD/tháng).

---

### 2. Tuyên bố vấn đề

#### Vấn đề hiện tại
* **Rủi ro mất an toàn thông tin & Lỗ hổng cổng công khai:** Khởi tạo EC2 mặc định thường mở tràn lan các cổng kết nối như SSH (Port 22), HTTP (Port 80) cho toàn mạng (`0.0.0.0/0`), dẫn đến nguy cơ cao bị tấn công dò quét mật khẩu (Brute-force) và tấn công khai thác lỗ hổng từ Internet.
* **Lưu lượng mạng không được mã hóa (Plaintext HTTP):** Việc chỉ chạy HTTP truyền thống khiến dữ liệu truyền tải giữa người dùng và máy chủ dễ bị chặn bắt (Eavesdropping/Man-in-the-Middle).
* **Phát sinh chi phí ngoài ý muốn (Uncontrolled Spend):** Người dùng Cloud mới rất dễ vi phạm giới hạn Free Tier do tạo nhầm các tài nguyên tốn phí (như NAT Gateway, Elastic IP không gán vào EC2 active, vượt dung lượng đĩa EBS).
* **Thao tác quản trị thủ công & Thiếu tự động hóa:** Quản trị viên phải SSH thủ công vào server để kéo code, cài đặt gói phụ thuộc, gây gián đoạn dịch vụ (Downtime) và tiềm ẩn lỗi do con người.

#### Giải pháp đề xuất
Hệ thống được thiết kế theo nguyên tắc cốt lõi: **Bảo mật phân tầng (Defense in Depth) và Tối ưu chi phí từ thiết kế (Cost Optimization by Design)**.

Hệ thống được chia làm 4 luồng xử lý độc lập:
1. **Flow C (GitOps Deployment Loop):** Tự động hóa CI/CD qua GitHub Actions, tự động test và deploy mã nguồn lên Nginx Web Server qua SSH/SSM mà không làm gián đoạn người dùng.
2. **Flow A (User Access & SSL/TLS Routing):** Định tuyến truy cập người dùng từ Internet qua CloudFront/WAF tới Nginx, tự động chuyển hướng HTTP $\rightarrow$ HTTPS và giải mã mã hóa tại hạ tầng biên/Web Server.
3. **Flow S (Security & Access Boundary):** Kiểm soát truy cập máy chủ tuyệt đối. Khóa cổng SSH (Port 22) hoặc chỉ giới hạn IP tĩnh quản trị, khuyến nghị dùng AWS Systems Manager (SSM) Session Manager để quản trị không cần mở cổng SSH.
4. **Flow M (Monitoring & Cost Control Loop):** Đẩy log hệ thống về CloudWatch và kích hoạt AWS Budgets gửi cảnh báo Email tự động khi chi phí chạm mốc $0.80 USD.

#### Lợi ích và Hoàn vốn đầu tư (ROI)
* **Chi phí vận hành tối ưu ($\sim 0$ USD/tháng):** Tận dụng tối đa các tài nguyên trong gói AWS Free Tier (750 giờ EC2, 30GB EBS, 100GB Data Out). Loại bỏ hoàn toàn các dịch vụ đắt đỏ như NAT Gateway hay Application Load Balancer thường trực.
* **Bảo mật tiêu chuẩn doanh nghiệp:** Mã hóa $100\%$ lưu lượng qua HTTPS, bảo mật cổng động, loại bỏ Access Key/Secret Key lưu cứng nhờ IAM Instance Profile.
* **Tự động hóa vận hành:** Giảm $90\%$ thời gian triển khai mã nguồn mới nhờ CI/CD Pipeline.

---

### 3. Kiến trúc giải pháp

#### Sơ đồ kiến trúc tổng thể

```text
                                +-----------------------------------------------------------------------------------+
                                |                                  AWS Cloud VPC                                    |
                                |                                                                                   |
[ Internet User ] ------------->|===> [ CloudFront / AWS WAF ]                                                      |
                                |            |                                                                      |
                                |            v                                                                      |
                                |   [ Internet Gateway ]                                                            |
                                |            |                                                                      |
                                |            v                                                                      |
                                |   +---------------------------------------------------------------------------+   |
                                |   | Public Subnet (10.0.1.0/24)                                               |   |
                                |   |                                                                           |   |
                                |   |   +-------------------------------------------------------------------+   |   |
                                |   |   | Security Group (Inbound: 80, 443, 22-MyIP)                         |   |   |
                                |   |   |                                                                   |   |   |
                                |   |   |   [ EC2 Instance: t2.micro / t3.micro ]                           |   |   |
                                |   |   |   +-----------------------------------------------------------+   |   |   |
                                |   |   |   | Nginx Web Server + Certbot (SSL/TLS)                      |   |   |   |
                                |   |   |   +-----------------------------------------------------------+   |   |   |
                                |   |   +-------------------------------------------------------------------+   |   |
                                |   +---------------------------------------------------------------------------+   |
                                |                                    |                                              |
                                |                                IAM Role                                           |
                                |                                    |                                              |
                                |            +-----------------------+-----------------------+                      |
                                |            v                                               v                      |
                                |   [ AWS CloudWatch Logs ]                         [ AWS Budgets Alarm ]           |
                                +-----------------------------------------------------------------------------------+
```

---

#### Chi tiết 4 luồng xử lý chính trong kiến trúc:

1. **Flow C — GitOps Deployment Loop (Tự động hóa triển khai)**
   * **C1 - C2:** Lập trình viên push mã nguồn trang Web lên GitHub Repository.
   * **C3:** GitHub Actions kích hoạt Workflow kiểm tra cú pháp (Linting/Build).
   * **C4:** Pipeline kết nối an toàn tới EC2 Instance để cập nhật thư mục tệp tin tĩnh (`/var/www/html`) và reload Nginx mà không làm gián đoạn trải nghiệm người dùng.

2. **Flow A — User Access & SSL/TLS Routing (Định tuyến & Mã hóa)**
   * **A1:** Người dùng gửi yêu cầu HTTP (Port 80) hoặc HTTPS (Port 443) tới tên miền trang web.
   * **A2:** Nginx nhận request, tự động chuyển hướng (301 Redirect) toàn bộ kết nối HTTP sang HTTPS.
   * **A3:** Chứng chỉ SSL/TLS (cấp bởi Certbot / Let's Encrypt) thực hiện bắt tay mã hóa mã nguồn truyền tải.
   * **A4:** Nginx trả về nội dung trang web đã được tối ưu hóa cho trình duyệt client.

3. **Flow S — Security & Identity Management (Quản lý an ninh)**
   * **S1:** Security Group đóng vai trò tường lửa stateful, chỉ chấp nhận truy cập HTTP/HTTPS từ `0.0.0.0/0` và giới hạn SSH (Port 22) từ chính địa chỉ `My IP` của nhà phát triển.
   * **S2:** EC2 được gắn **IAM Role (Instance Profile)** cấp quyền ghi log về CloudWatch, loại bỏ hoàn toàn việc lưu trữ Access Key cố định trong ứng dụng.
   * **S3 (Nâng cao):** Truy cập SSH trực tiếp có thể được thay thế bằng **AWS Systems Manager (SSM) Session Manager**, cho phép đóng hoàn toàn Port 22.

4. **Flow M — Monitoring & Budget Control (Giám sát & Quản lý chi phí)**
   * **M1:** CloudWatch Agent thu thập chỉ số CPU, RAM, Disk Utilization và Access Logs của Nginx.
   * **M2:** AWS Budgets liên tục theo dõi chi phí thực tế phát sinh trên tài khoản. Nếu chi phí dự báo vượt quá $0.80 USD, hệ thống tự động gửi Email cảnh báo lập tức.

---

#### Dịch vụ AWS sử dụng

* **AWS VPC & Networking:** Xây dựng mạng riêng ảo, chia Subnet, Route Table và Internet Gateway.
* **Amazon EC2:** Máy chủ ảo chạy Nginx Web Server (`t2.micro` hoặc `t3.micro`).
* **AWS Security Group:** Tường lửa ảo kiểm soát lưu lượng Inbound/Outbound.
* **AWS IAM:** Quản lý vai trò (Roles) và phân quyền an toàn cho EC2 Instance.
* **Amazon CloudWatch:** Giám sát hiệu năng và lưu trữ log tập trung.
* **AWS Budgets:** Tự động giám sát và cảnh báo ngưỡng chi phí.
* **Amazon CloudFront & AWS WAF (Tùy chọn bổ sung):** Mạng phân phối nội dung (CDN) và tường lửa ứng dụng web giúp tăng tốc độ tải và chống tấn công DDoS.

---

### 4. Triển khai kỹ thuật

#### Các giai đoạn triển khai

1. **Giai đoạn 1: Thiết kế hạ tầng mạng & Cấu hình An ninh (Tuần 1)**
   * Khởi tạo VPC riêng biệt (`10.0.0.0/16`), tạo Public Subnet (`10.0.1.0/24`) và gán Internet Gateway.
   * Cấu hình Route Table định tuyến lưu lượng `0.0.0.0/0` ra Internet Gateway.
   * Khởi tạo Security Group với quy tắc phân quyền tối thiểu (HTTP 80, HTTPS 443, SSH 22 chỉ cho `My IP`).

2. **Giai đoạn 2: Khởi tạo Compute & Quản lý truy cập IAM (Tuần 2)**
   * Launch máy chủ EC2 (`t2.micro` / `t3.micro`, Ubuntu 22.04 LTS) thuộc gói Free Tier.
   * Tạo SSH Key Pair (`.pem`), phân quyền tệp tin `chmod 400` trên máy cá nhân.
   * Tạo IAM Role với quyền `CloudWatchAgentServerPolicy` và gán vào EC2.

3. **Giai đoạn 3: Cài đặt Web Server & Cấu hình Mã hóa HTTPS (Tuần 3)**
   * Kết nối SSH vào EC2, cài đặt và tối ưu hóa Nginx:
     ```bash
     sudo apt update && sudo apt install nginx -y
     sudo systemctl enable --now nginx
     ```
   * Trỏ bản ghi A Record của Tên miền về Public IP của EC2.
   * Cài đặt Certbot và đăng ký chứng chỉ SSL/TLS miễn phí:
     ```bash
     sudo apt install certbot python3-certbot-nginx -y
     sudo certbot --nginx -d domain-cua-ban.com
     ```

4. **Giai đoạn 4: Tự động hóa CI/CD Deployment Pipeline (Tuần 4)**
   * Thiết lập GitHub Repository chứa mã nguồn Web.
   * Tạo GitHub Actions Workflow tự động deploy mã nguồn lên thư mục `/var/www/html` trên EC2 mỗi khi gộp code vào nhánh `main`.

5. **Giai đoạn 5: Tối ưu Bảo mật, Kiểm thử & Cảnh báo Chi phí (Tuần 5 - 6)**
   * Cấu hình Nginx Security Headers (X-Frame-Options, X-Content-Type-Options, CSP).
   * Tạo AWS Budget cảnh báo ngưỡng chi phí $1.00 USD.
   * Tiến hành đánh giá an toàn mạng (Port Scanning, SSL Lab Testing) và đóng gói Báo cáo Thực tập.

---

#### Yêu cầu kỹ thuật & Bảo mật

* **Mã hóa lưu lượng 100%:** Bắt buộc giao thức HTTPS (TLS 1.2/1.3) cho toàn bộ kết nối.
* **Bảo mật máy chủ (Server Hardening):** Tắt tính năng hiển thị phiên bản Nginx (`server_tokens off;`), chặn truy cập tệp tin ẩn (`.git`, `.env`).
* **Quản lý khóa an toàn:** Không bao giờ đưa file chứa Private Key (`.pem`) hay AWS Access Key lên GitHub.
* **Tự động gia hạn SSL:** Cấu hình Cron job chạy `certbot renew --dry-run` để đảm bảo chứng chỉ luôn hợp lệ.

---

### 5. Lộ trình & Mốc triển khai

```text
+-----------------------------------------------------------------------------------+
| Tuần 1 - 2: Khảo sát & Xây dựng Hạ tầng Mạng (VPC, Subnet, IGW, Security Group)    |
|   - Thiết lập VPC cô lập & Route Table                                            |
|   - Cấu hình Security Group tuân thủ nguyên tắc Least Privilege                    |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| Tuần 3 - 4: Triển khai EC2, Web Server Nginx & Mã hóa HTTPS                        |
|   - Launch EC2 Instance (Ubuntu 22.04 LTS), gán IAM Role                          |
|   - Cài đặt Nginx, gán Domain và cấp phát chứng chỉ SSL/TLS Let's Encrypt          |
+-----------------------------------------------------------------------------------+
                                         |
                                         v
+-----------------------------------------------------------------------------------+
| Tuần 5 - 6: Tự động hóa CI/CD, Giám sát Chi phí & Báo cáo Thực tập                |
|   - Tự động hóa Deploy qua GitHub Actions Pipeline                                |
|   - Cấu hình AWS Budgets & CloudWatch Alarms                                      |
|   - Kiểm thử bảo mật (SSL Labs Grade A+) và tổng kết báo cáo                      |
+-----------------------------------------------------------------------------------+
```

---

### 6. Ước tính ngân sách

Nhờ thiết kế loại bỏ các thành phần tốn phí không cần thiết (NAT Gateway, ALB) và hoạt động hoàn toàn trong phạm vi gói **AWS Free Tier**, chi phí vận hành ước tính như sau:

| Dịch vụ AWS / Tài nguyên | Cấu hình / Quy mô ước tính | Chi phí ước tính / Tháng (USD) |
| :--- | :--- | :---: |
| **Amazon EC2** | 1 x `t2.micro` / `t3.micro` (750 giờ/tháng) | **$0.00** (Free Tier) |
| **Amazon EBS Storage** | 30 GB GP3 Storage | **$0.00** (Free Tier) |
| **AWS Data Transfer** | < 100 GB Data Out ra Internet / tháng | **$0.00** (Free Tier) |
| **Amazon CloudWatch** | Basic Metrics & Log Collection (< 5GB) | **$0.00** (Free Tier) |
| **SSL/TLS Certificate** | Certbot / Let's Encrypt | **$0.00** (Mã nguồn mở) |
| **AWS Budgets** | 2 Cảnh báo ngân sách | **$0.00** (Miễn phí) |
| **Tổng chi phí ước tính** | **Hạ tầng Web Server Bảo mật trên AWS** | **~ $0.00 USD / Tháng** |

> **[!TIP] Điểm tối ưu chi phí vượt trội:**
> 1. Không sử dụng **NAT Gateway** (tiết kiệm $\sim \$32$/tháng).
> 2. Không sử dụng **Application Load Balancer** thường trực (tiết kiệm $\sim \$20$/tháng).
> 3. Không giữ **Elastic IP** tĩnh nếu EC2 bị Stop (tránh phí phạt giữ IP không gán).

---

### 7. Đánh giá rủi ro

#### Ma trận rủi ro & Chiến lược giảm thiểu

| Rủi ro tiềm ẩn | Mức độ ảnh hưởng | Xác suất | Chiến lược giảm thiểu |
| :--- | :---: | :---: | :--- |
| **Tấn công dò quét cổng SSH (Port 22)** | Cao | Cao | Giới hạn IP truy cập Port 22 chỉ cho `My IP`. Khuyến nghị dùng **AWS SSM Session Manager** để khóa hoàn toàn Port 22. |
| **Phát sinh chi phí bất ngờ trên AWS** | Trung bình | Trung bình | Cấu hình **AWS Budgets** gửi email cảnh báo ở mốc $0.80 USD. Không bật NAT Gateway hay các tài nguyên ngoài Free Tier. |
| **Chứng chỉ SSL hết hạn sau 90 ngày** | Trung bình | Thấp | Cấu hình Cron job chạy tự động kiểm tra và gia hạn chứng chỉ `certbot renew`. |
| **Lộ thông tin quản trị lên Git** | Rất cao | Thấp | Thêm file `.pem`, tệp môi trường `.env` vào `.gitignore`. Sử dụng GitHub Secrets lưu SSH Credentials cho CI/CD. |

---

### 8. Kết quả kỳ vọng

* **Hạ tầng chuẩn hóa & An toàn:** Xây dựng thành công hệ thống Web Server đáp ứng tiêu chuẩn bảo mật đám mây (Cloud Security Best Practices), đạt điểm **Grade A+** trên SSL Labs Test.
* **Tối ưu chi phí triệt để:** Khẳng định mô hình vận hành sản xuất thực tế với chi phí tối thiểu $\sim 0$ USD/tháng thuộc gói AWS Free Tier.
* **Khả năng tái sử dụng:** Đóng gói hoàn chỉnh Báo cáo Thực tập và Bộ tài liệu hướng dẫn (Architectural Blueprint) có thể nhân rộng để triển khai các ứng dụng Web khác trên hạ tầng AWS.