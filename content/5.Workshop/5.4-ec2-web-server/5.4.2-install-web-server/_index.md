---
title: "Install the Web Server"
date: 2026-10-30
weight: 2
chapter: false
pre: "<b>5.4.2. </b>"
---

## Objective

Member 2 installs **Nginx** on **workshop-web-01** running **Amazon Linux 2023**, enables automatic startup, and verifies web responses. In subsequent sections, Nginx will accept HTTPS connections and forward requests to the application.

Test the default page on localhost and then over port 80 using the public IPv4 address. The document portal follows in 5.4.3.

## Prerequisites

- Complete [5.4.1. Launch EC2 and Connect Using SSH](../5.4.1-launch-and-connect/).
- Connect to workshop-web-01 as ec2-user with sudo privileges.
- EC2 can download packages through the configured routes, DNS, security group outbound rules, and bidirectional NACL rules.
- The S3 endpoint policy does not block AWS operating system package buckets.

**Except for the explicitly marked SSH tunnel step, run all commands inside the SSH terminal on EC2.** Execute each block in order and check its results before continuing.

## Step 1: Check the OS and update packages

```bash
whoami
cat /etc/os-release
sudo dnf upgrade -y
```

Confirm ec2-user and Amazon Linux 2023. The upgrade command updates packages within the currently configured repository version. AL2023 uses versioned repositories; this command does not guarantee upgrading to the latest AL2023 release. For an older AMI, review AWS update guidance and agree on a release before upgrading the operating system.

If the update requires a reboot, run `sudo reboot`, wait for readiness, and reconnect through SSH. A reboot disconnects SSH; it does not require creating another EC2 instance.

![](/FCAJ_Worklog/images/54/inx1.png)


## Step 2: Install Nginx

```bash
sudo dnf install -y nginx
nginx -v
rpm -q nginx
```

The output should show the Nginx version and installed package. Record the actual version in the deployment log; it need not match reference screenshots. Do not use amazon-linux-extras or apt on this operating system.

This section installs Nginx only. Install the application runtime and libraries according to the source code in 5.4.3, avoiding unused software stacks.

![](/FCAJ_Worklog/images/54/inx2.png)

## Step 3: Validate configuration and start the service

```bash
sudo nginx -t
```

Continue only after seeing `syntax is ok` and `test is successful`. Then run:

```bash
sudo systemctl enable --now nginx
systemctl is-active nginx
systemctl is-enabled nginx
sudo systemctl status nginx --no-pager
```

Expected results: **active** and **enabled**. Active means the service is running; enabled means it is configured to start when the server boots.

If nginx -t reports an error, fix the indicated file/line before starting or reloading. Do not ignore the error and continue.

![](/FCAJ_Worklog/images/54/inx3.png)


## Step 4: Test HTTP locally on EC2

```bash
curl -I http://127.0.0.1/
curl -sS http://127.0.0.1/ | head -n 15
sudo ss -lntp
```

With the package's default configuration, expect **HTTP 200**, HTML content, and Nginx listening on port 80. If results differ, inspect the effective configuration with `sudo nginx -T` and the logs in step 6. Do not change firewall rules while localhost requests are still failing.

HTTP is used here only for local server testing. TLS is not configured, so https://PUBLIC_IPV4 is not expected to work yet. A successful localhost request does not prove Internet access to the website.

![](/FCAJ_Worklog/images/54/inx4.png)


## Step 5: View the default page through an SSH tunnel

On **your local computer**, open a second terminal/PowerShell window in the directory containing YOUR_KEY_PAIR.pem. Replace PUBLIC_IPV4 with the current EC2 address and run:

```bash
ssh -i "YOUR_KEY_PAIR.pem" -N -L 127.0.0.1:8080:127.0.0.1:80 ec2-user@PUBLIC_IPV4
```

Keep this window open. The `-N` option does not open a shell, so no new prompt is expected after successful connection. Open a browser on **that same local computer** at:

```text
http://127.0.0.1:8080
```

The Nginx default page should appear. The tunnel uses encrypted SSH on port 22; no inbound 80 or 8080 rules are needed in the security group/NACL. The 127.0.0.1:8080 address is for testing on your computer, not the public website URL.

If port 8080 is occupied, change only the first `127.0.0.1:8080` in the command to `127.0.0.1:8081`, then browse to port 8081. Close the tunnel using **Ctrl+C** after capturing the screenshot. Do not enter application credentials at this stage.


## Step 6: Locate configuration and logs

| Typical package location | Purpose |
| --- | --- |
| /etc/nginx/nginx.conf | Main configuration |
| /etc/nginx/conf.d/ | Additional configuration when included by the main file |
| /usr/share/nginx/html/ | Default page content |
| /var/log/nginx/access.log | HTTP request log |
| /var/log/nginx/error.log | Nginx error log |

Inspect the logs after the test requests:

```bash
sudo tail -n 20 /var/log/nginx/access.log
sudo tail -n 20 /var/log/nginx/error.log
sudo journalctl -u nginx -n 30 --no-pager
```

The error log may be empty if no errors occurred. Previously modified configurations may use different paths; `sudo nginx -T` shows the effective configuration. Do not publish full configuration/log output containing sensitive information.

After configuration edits in later sections, validate and reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Do not store passwords, private keys, .env files, or S3 documents in the default web directory. Section 5.4.3 sets up the application and its separate service; 5.4.4 configures HTTPS and Nginx-to-backend forwarding. Replacing the entire nginx.conf is unnecessary during this installation step.

## Common issues

| Symptom | Check/action |
| --- | --- |
| dnf cannot download packages | Check DNS, routes, outbound 80/443, inbound ephemeral ports, and S3 endpoint policy; do not disable package signature checks |
| nginx: command not found | Verify successful installation, the target instance, and operating system |
| Address already in use | Identify the process using sudo ss -lntp; do not stop an unidentified service |
| nginx -t reports an error | Correct the indicated file/line; check semicolons and braces if configuration was edited |
| localhost curl is refused | Check Nginx status, listen port, and journalctl |
| Tunnel does not open | Check SSH, public IPv4, private key, and local port 8080; keep the tunnel terminal open |
| Internet HTTP times out | Verify port 80 in SG/NACL and the current IP. Check localhost for Nginx. |

## Completion

Nginx is installed, configuration is valid, the service is active/enabled, and the default page has been checked using curl and a browser through an SSH tunnel. 

Continue to [5.4.3. Implement Login and Document Upload/Download](../5.4.3-document-web-app/).

## References

- [AL2023 package management](https://docs.aws.amazon.com/linux/al2023/ug/package-management.html)
- [Manage AL2023 updates](https://docs.aws.amazon.com/linux/al2023/ug/managing-repos-os-updates.html)
- [Nginx beginner guide](https://nginx.org/en/docs/beginners_guide.html)
- [Nginx command-line parameters](https://nginx.org/en/docs/switches.html)

