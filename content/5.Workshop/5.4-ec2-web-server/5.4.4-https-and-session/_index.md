---
title: "Configure HTTPS and Verify Login Sessions"
date: 2026-10-31
weight: 4
chapter: false
pre: "<b>5.4.4. </b>"
---

## Objective

Configure Nginx to terminate TLS on port 443 and proxy to Gunicorn at **127.0.0.1:8000** from 5.4.3. Then verify cookies, login, logout, and session expiry.

The lab uses a **self-signed certificate for the public IPv4 address**, without a domain. It encrypts connections but is not trusted by browsers by default. Use practice accounts/documents only; real deployments need trusted certificates and an appropriate renewal process.

## Prerequisites

- Complete 5.4.3: document-portal is active and localhost:8000/healthz returns 200.
- Nginx and OpenSSL on Amazon Linux 2023, with sudo over SSH.
- sg-secure-web permits inbound TCP 443; the NACL permits inbound 443 and outbound ephemeral ports as in 5.3.4.
- Port 80 redirects to HTTPS. Gunicorn listens on 127.0.0.1:8000.
- Record the current public IPv4. TLS and the login page can be checked before S3/IAM is ready; the document list may return 503 after login until section 5.5 is complete.

Run commands on EC2 unless explicitly marked for your local computer.

## Step 1: Create a certificate with an IP SAN

Enter the actual public IPv4 when prompted. If Python reports an invalid address, stop and enter it again before generating the certificate. Keep the same SSH shell so PORTAL_IP remains set.


```bash
sudo dnf install -y openssl
read -r -p "Current EC2 public IPv4: " PORTAL_IP
python3 -c 'import ipaddress,sys; ipaddress.IPv4Address(sys.argv[1])' "$PORTAL_IP"

```


```bash
sudo install -d -m 700 /etc/nginx/tls
sudo openssl req -x509 -nodes -newkey rsa:2048 -sha256 -days 30 \
  -keyout /etc/nginx/tls/portal.key \
  -out /etc/nginx/tls/portal.crt \
  -subj "/CN=$PORTAL_IP" \
  -addext "subjectAltName=IP:$PORTAL_IP" \
  -addext "extendedKeyUsage=serverAuth"
sudo chmod 600 /etc/nginx/tls/portal.key
sudo chmod 644 /etc/nginx/tls/portal.crt
sudo openssl x509 -in /etc/nginx/tls/portal.crt -noout -dates -fingerprint -sha256 -ext subjectAltName
```

![](/FCAJ_Worklog/images/54/https1.png)


## Step 2: Configure HTTPS and the reverse proxy

This configuration is for the workshop's fresh EC2 instance and replaces the default page from 5.4.2. If the server hosts other websites, do not replace the full configuration; merge a dedicated configuration instead. The backup command uses a timestamp to preserve previous backups; record its path.

This configuration does not include conf.d, avoiding old default servers/port 80 listeners. Additional files in conf.d will not apply. Run the complete block; reload only if nginx -t succeeds.


```bash
sudo cp -a /etc/nginx/nginx.conf "/etc/nginx/nginx.conf.before-https.$(date +%Y%m%d%H%M%S)"
sudo tee /etc/nginx/nginx.conf > /dev/null <<'NGINX'
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;
include /usr/share/nginx/modules/*.conf;
events { worker_connections 1024; }
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    sendfile on;
    server_tokens off;
    # Do not record URL paths, query strings, cookies, or credentials.
    log_format portal '$time_iso8601 $request_method $status $body_bytes_sent';
    access_log /var/log/nginx/access.log portal;
    server {
        listen 80 default_server;
        server_name _;
        return 301 https://PUBLIC_IPV4$request_uri;
    }
    server {
        listen 443 ssl default_server;
        server_name _;
        ssl_certificate /etc/nginx/tls/portal.crt;
        ssl_certificate_key /etc/nginx/tls/portal.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_session_tickets off;
        client_max_body_size 6m;
        location / {
            proxy_pass http://127.0.0.1:8000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Forwarded-Proto https;
            proxy_connect_timeout 5s;
            proxy_read_timeout 90s;
        }
    }
}
NGINX
sudo sed -i "s/PUBLIC_IPV4/${PORTAL_IP:?Set PORTAL_IP first}/g" /etc/nginx/nginx.conf
sudo nginx -t && sudo systemctl reload nginx
sudo ss -lntp
```

![](/FCAJ_Worklog/images/54/https2.png)

## Step 3: Verify TLS locally on EC2

These commands trust the generated certificate explicitly and verify its IP SAN. --resolve connects to localhost while still checking the URL's identity, avoiding dependence on EC2 connecting to its own public IPv4.

Expect status ok from /healthz and a 302 redirect with Location /login from /. Do not use curl -k as evidence of certificate verification.


```bash
sudo curl --noproxy '*' --cacert /etc/nginx/tls/portal.crt \
  --resolve "$PORTAL_IP:443:127.0.0.1" "https://$PORTAL_IP/healthz"
sudo curl --noproxy '*' --cacert /etc/nginx/tls/portal.crt \
  --resolve "$PORTAL_IP:443:127.0.0.1" -I "https://$PORTAL_IP/"
```

![](/FCAJ_Worklog/images/54/https3.png)

## Step 4: Test from your local computer

On EC2, copy the **public certificate**, never the private key:


```bash
sudo install -m 644 /etc/nginx/tls/portal.crt /home/ec2-user/portal.crt
```

On **your local computer**, in the key-pair directory, replace PUBLIC_IPV4 and run:

```bash
scp -i "YOUR_KEY_PAIR.pem" ec2-user@PUBLIC_IPV4:~/portal.crt .
```


```bash
curl --cacert portal.crt https://PUBLIC_IPV4/healthz
```

Windows PowerShell: use `curl.exe` if curl is an alias. Transfer the certificate through verified SSH; do not trust a certificate obtained through an unverified connection. On errors, check IP, SAN, and certificate file rather than adding -k.

## Step 5: Verify browser behavior and cookies

Open **https://PUBLIC_IPV4/login** on your computer. A self-signed certificate triggers a trust warning. View its details and compare the SHA-256 fingerprint, IP SAN, and validity dates against step 1 before granting an exception for this exact lab site. If they differ or the browser prevents proceeding, stop and resolve certificate trust; do not disable browser-wide security checks.

An exception does not make the certificate publicly trusted. A “green padlock” screenshot is not required. Evidence should show the correct certificate IP and successful curl --cacert verification.

Open Developer Tools (F12) → Application/Storage → Cookies. **portal_session** must have Secure, HttpOnly, SameSite=Lax, Path=/, and approximately 30-minute expiry. The pre-login cookie represents an unauthenticated session for CSRF; successful login rotates the session ID. **Hide entire cookie/token values in screenshots**, leaving only names and attributes visible.

Log in with the website account, not an IAM user. If login leads to 503 Storage unavailable, check S3 permissions in 5.5; do not disable Secure or change TLS to resolve data-access failures.

![](/FCAJ_Worklog/images/54/https4.png)

## Step 6: Test the session lifecycle

Perform the tests below in order. Use Developer Tools → Network to inspect response codes and enable Preserve log if necessary. Never publish request bodies containing passwords or tokens.

| Test | Expected result |
| --- | --- |
| Incognito window opens / or /download/sample.txt | 302 to /login without reading S3 |
| Incorrect password | 401, without an authenticated session |
| Correct login | POST /login returns 302 and rotates the cookie ID; listing needs S3 |
| Select Log out | POST /logout revokes the session, deletes the cookie, and redirects to login |
| Reopen / after logout | Redirect to login; Back may display a stale page snapshot, so reload to test actual access |
| Wait over 30 minutes after authenticated session creation | A new protected request redirects to login; activity does not extend expiry |
| Restart application | All current sessions become invalid |

To test restart, log in and run on EC2:

```bash
sudo systemctl restart document-portal
systemctl is-active document-portal
```

Refresh the browser: login should be required again. Do not increase the worker count above 1 to address session loss. Do not screenshot old cookies as evidence; cookie replay can be tested separately in a private environment.



## Troubleshooting and rollback

| Error | Action |
| --- | --- |
| Port 443 timeout | Check public IPv4, IGW, SG 443, NACL and Nginx listener |
| 502 Bad Gateway | Check document-portal status, localhost:8000/healthz and nginx error.log |
| Proxy permission denied with SELinux Enforcing | Check getenforce and audit logs; if SELinux blocks reverse proxy connections, the administrator may enable httpd_can_network_connect using setsebool -P; do not disable SELinux globally |
| Certificate IP mismatch | Public IPv4 changed; regenerate with the correct SAN, reload and transfer the new public certificate |
| nginx -t failure | Do not reload; fix the error or restore the recorded nginx.conf.before-https backup, validate, then reload |
| Repeated login redirects | Check HTTPS, cookies, one worker and repeated service restarts |
| 503 Storage unavailable | Check S3/IAM in 5.5; this is not a certificate error |
| Upload 413 | Nginx permits a 6m request; the application still limits file contents to 5 MiB |

If restoring an HTTP-only configuration, stop using login until HTTPS is restored. Do not submit credentials over HTTP.

## Completion

HTTPS operates on 443, the backend remains local, cookies have the correct attributes, and session checks pass. Defer S3-dependent results until 5.5 if necessary. The certificate expires after 30 days; plan regeneration for the lab. Recheck public IPv4 after Stop/Start. Do not enable HSTS for this self-signed lab.

These commands are instructions for your EC2 instance; they have not been executed in your AWS account.

Continue to [5.5. S3 Storage and Data Security](../../5.5-s3-data-security/).

## References

- [Nginx HTTPS configuration](https://nginx.org/en/docs/http/configuring_https_servers.html)
- [OpenSSL req](https://docs.openssl.org/3.0/man1/openssl-req/)
