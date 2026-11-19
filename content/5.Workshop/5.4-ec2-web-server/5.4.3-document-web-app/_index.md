---
title: "Implement Login and Document Upload/Download"
date: 2026-10-30
weight: 3
chapter: false
pre: "<b>5.4.3. </b>"
---

## Objective

Deploy the sample document application on the Amazon Linux 2023 EC2 instance with Nginx installed in 5.4.2. Member 2 uses **Python 3.11 + Flask + Gunicorn**, with Boto3 and the IAM role for S3.

Download **document-portal.zip** supplied with this section. It contains app.py, configure.py, requirements.txt, and document-portal.service. This standalone workshop sample does not require an external repository.

The application provides login/logout, document listing, upload, and backend-proxied download. It uses one shared account, without registration, a database, or deletion. Files use randomized prefixes under portal/ to avoid overwrites; the bucket remains private.

**Dependencies:** EC2/SSH/Nginx work. Use the agreed actual bucket name; the bucket and permissions may be completed later in 5.5. Data operations require the bucket, IAM role, and policies to be ready. HTTPS is configured in 5.4.4.

## Step 1: Transfer the source to EC2

On **your local computer**, place the ZIP beside YOUR_KEY_PAIR.pem, replace PUBLIC_IPV4, and run the command below. Do not transfer the private key to EC2.

```bash
scp -i "YOUR_KEY_PAIR.pem" document-portal.zip ec2-user@PUBLIC_IPV4:~/
```

![](/FCAJ_Worklog/images/54/deploy01.png)
## Step 2: Install the runtime and service account

Run subsequent commands **on EC2 through SSH**. Use a fresh installation directory. If portal already exists, check `id portal` and skip useradd; do not delete accounts or overwrite an existing application.

```bash
sudo dnf install -y python3.11 python3.11-pip unzip
unzip -n ~/document-portal.zip -d ~/portal-source
sudo useradd --system --home-dir /opt/document-portal --shell /sbin/nologin portal
sudo install -d -m 755 /opt/document-portal
sudo cp ~/portal-source/document-portal/app.py /opt/document-portal/
sudo cp ~/portal-source/document-portal/requirements.txt /opt/document-portal/
sudo cp ~/portal-source/document-portal/configure.py /opt/document-portal/
sudo python3.11 -m venv /opt/document-portal/.venv
sudo /opt/document-portal/.venv/bin/pip install -r /opt/document-portal/requirements.txt
sudo /opt/document-portal/.venv/bin/pip check
sudo /opt/document-portal/.venv/bin/pip freeze | sudo tee /opt/document-portal/requirements.lock.txt > /dev/null
```

![](/FCAJ_Worklog/images/54/deploy02.png)

## Step 3: Configure the website account

Run the interactive setup. Enter the actual bucket name, website username, and a password of at least 12 characters twice. Password input is hidden and only a hash is stored; do not put passwords on command lines. Do not enter IAM passwords or AWS access keys.

The script refuses to overwrite existing configuration. For changes, the administrator should back up privately and edit deliberately; do not casually delete the configuration to retry.

```bash
sudo /opt/document-portal/.venv/bin/python /opt/document-portal/configure.py
sudo chown root:portal /etc/document-portal /etc/document-portal/config.json
sudo chmod 750 /etc/document-portal
sudo chmod 640 /etc/document-portal/config.json
sudo -u portal test -r /etc/document-portal/config.json && echo "Config readable by portal"
```

![](/FCAJ_Worklog/images/54/deploy03.png)

## Step 4: Start Gunicorn with systemd

The service runs as portal without sudo privileges; its code is outside the web directory. It binds only to **127.0.0.1:8000**. Keep **1 worker**, since sessions and login counters reside in process memory; multiple workers would cause inconsistent sessions.

Expected results: active service, port 8000 on loopback only, /healthz returns 200, and unauthenticated / redirects with 302 to /login. The healthcheck does not call S3.

```bash
sudo install -m 644 ~/portal-source/document-portal/document-portal.service /etc/systemd/system/document-portal.service
sudo systemctl daemon-reload
sudo systemctl enable --now document-portal
sudo systemctl status document-portal --no-pager
sudo ss -lntp
curl -i http://127.0.0.1:8000/healthz
curl -I http://127.0.0.1:8000/
```

![](/FCAJ_Worklog/images/54/deploy04.png)

## Step 5: Inspect logs and restart

Logs record events such as login_success, login_failed, upload_success, download_success, and storage_error, without passwords, cookies, or file contents. Hand over the service name to the CloudWatch member. Logs currently reside in the journal; collection into CloudWatch is configured in 5.6.3.

Restarting removes all in-memory sessions, requiring users to log in again. This is intentional lab behavior.

```bash
sudo journalctl -u document-portal -n 50 --no-pager
sudo systemctl restart document-portal
systemctl is-active document-portal
systemctl is-enabled document-portal
```

![](/FCAJ_Worklog/images/54/deploy05.png)
## Step 6: Test features after HTTPS and S3 are ready

Session cookies always use Secure, HttpOnly, and SameSite=Lax. **Do not disable Secure to log in over HTTP.** This section stops at service checks; finish 5.4.4 and open https://server-address/login. Nginx must forward to 127.0.0.1:8000 and set client_max_body_size 6m for multipart overhead. The application separately limits file contents to **5 MiB = 5 × 1024 × 1024 bytes** (the lab's 5 MB convention).

| Test | Expected result |
| --- | --- |
| Open / or /download/sample.txt without login | Redirect to /login without reading S3 |
| Incorrect password | Generic error; no authenticated session |
| Correct login | Document list when S3 is ready |
| Upload valid UTF-8 TXT or PDF | Stored under portal/ with SSE-S3 and listed |
| .exe, invalid PDF header, invalid UTF-8 TXT, empty file, or >5 MiB | Rejected by the server |
| Download | Contents match the uploaded file; served as an attachment |
| Logout and replay old cookie | Session revoked; login required |
| POST without CSRF token | HTTP 400 |



## S3 permissions and sample limitations

The role needs s3:ListBucket on the bucket restricted to portal/, and s3:GetObject/s3:PutObject on arn:aws:s3:::BUCKET_NAME/portal/*. DeleteObject and public bucket access are unnecessary. Boto3 obtains credentials from the instance profile; do not run aws configure with long-term access keys.

Sessions have a fixed 30-minute expiry. Memory holds at most 1,000 sessions. A global limit of 10 login attempts/minute can make the entire group wait if one person retries frequently. This is a single-server lab sample, not a production system. PDF header checks and UTF-8 TXT checks are **not malware scanning**; use practice documents only. The interface has no pagination for large collections.

## Troubleshooting

- 503 Storage unavailable: check bucket, Region, IAM role, endpoint policy, and networking; do not make the bucket public.
- Service failed: inspect journalctl, configuration read permissions, and installed libraries; do not screenshot configuration contents.
- Repeated login redirects: use HTTPS and check Secure cookies, expiry, and exactly one worker.
- Upload 413: check file size and Nginx client_max_body_size.
- 429: wait one minute; do not bypass limits by adding workers.

The sample was tested locally with mocked S3: login, CSRF, file limits, prefix/encryption, download, and cookie revocation. It has not been run in your AWS account; the real integration tests above remain necessary.

Continue to [5.4.4. Configure HTTPS and Verify Login Sessions](../5.4.4-https-and-session/).

## References

- [Flask security](https://flask.palletsprojects.com/en/stable/web-security/)
- [Boto3 credentials](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html)
