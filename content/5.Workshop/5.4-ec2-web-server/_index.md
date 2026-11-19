---
title: "Deploy a Web Server on EC2"
date: 2026-10-30
weight: 4
chapter: false
pre: "<b>5.4. </b>"
---

**Execution order:** 5.4.1–5.4.2 → 5.5.1–5.5.2 → 5.4.3 → 5.5.3 (steps 1–5) → 5.4.4 → 5.5.3 (step 6) → 5.5.4. Use the source bundle in the workshop ZIP’s `demo/` folder.

## Overview

This section deploys a **web server on Amazon EC2** to run the document portal with login, upload, and download functionality. The server uses the network and firewall configuration from section 5.3; documents are stored in a private S3 bucket.

**Member 2** creates EC2, installs the web server, deploys the application, and configures HTTPS. Member 1 supports networking/firewalls, Member 3 coordinates S3 permissions, and Member 4 receives log details for CloudWatch configuration in section 5.6.

This page introduces the scope and sequence. Detailed procedures are provided in the four subsections below.

## Objectives

- Launch one EC2 instance in subnet-web-public and administer it over SSH from an authorized source.
- Install the web server and application runtime.
- Implement login, logout, document listing, upload, and download.
- Configure HTTPS and verify login sessions before using the lab account's actual credentials.
- Prepare S3 integration through an IAM role and application logging for monitoring.

## Deployment prerequisites

| Component | Required information |
| --- | --- |
| Region | Singapore — ap-southeast-1 |
| VPC | vpc-secure-web — 10.0.0.0/16 |
| Subnet | subnet-web-public — 10.0.0.0/24 |
| Route table | rtb-web-public, with Internet routing through igw-secure-web and S3 routing through s3-gateway-endpoint |
| Security group | sg-secure-web, SSH from administrator public IPv4/32 and HTTPS from the Internet |
| Network ACL | nacl-web-public, configured for requests and responses |
| Planned EC2 name | workshop-web-01 |
| Planned key pair | workshop-web-key |
| IAM role / Instance profile | WorkshopEC2AppRole / WorkshopEC2AppRole, matching the administrator's actual configuration |
| S3 | Actual bucket name agreed in 5.2.2, in the same Region; portal/ prefix |

Complete [5.3. Network and Firewall Configuration](../5.3-network-and-firewall/) and verify resource IDs against the handover table. The EC2 operator needs resource launch permissions and iam:PassRole for the permitted role. Do not work around missing permissions by embedding access keys in source code.

In 5.4.1, enable **Auto-assign public IP** when launching EC2 in the prepared subnet. Select the existing security group; do not let the launch wizard create another group allowing SSH from the entire Internet.

If the bucket and S3 permissions are not ready, the server and interface can be deployed first. Complete upload/download testing must wait for section 5.5. A working login page alone does not demonstrate successful S3 storage.

## Application scope

The website uses one pre-created shared lab account. This website account is separate from IAM users operating the AWS Console. Registration, password recovery, multi-user authorization, and a database are outside this scope.

| Feature | Requirement |
| --- | --- |
| Login | Validate account credentials; store the password as a hash |
| Login session | Protect document pages and APIs from unauthenticated access; logout invalidates the session |
| Document listing | Display documents under portal/ in the designated bucket |
| Upload | Accept PDF/TXT files, up to 5 MB each; validate on the server |
| Download | Validate the session, read the document from S3, and return its contents to the user |
| Sample data | Use practice documents without personal or confidential information |

## Request flow and security requirements

Users access EC2 over HTTPS. The application validates the login session before handling document requests, then calls S3 using the IAM role attached to EC2. Same-Region S3 traffic uses the Gateway Endpoint created in 5.3.5. The bucket remains private; the browser receives documents through the application.

The IAM role supplies temporary credentials to the application through an instance profile. Use the AWS SDK credential mechanism; do not store long-term access keys in source code, Git, or public configuration files.

Keep password hashes and session secrets outside the public web directory. Do not log passwords, session cookies, or document contents. Expose only required web ports; do not expose the internal backend directly to the Internet. Use port 80 to verify Nginx initially and then to redirect to HTTPS.

If using a self-signed certificate for the lab, explain browser trust limitations and certificate verification in 5.4.4. Do not describe a self-signed certificate as publicly trusted by browsers.

## Contents

- [5.4.1. Launch EC2 and Connect Using SSH](./5.4.1-launch-and-connect/)
- [5.4.2. Install the Web Server](./5.4.2-install-web-server/)
- [5.4.3. Implement Login and Document Upload/Download](./5.4.3-document-web-app/)
- [5.4.4. Configure HTTPS and Verify Login Sessions](./5.4.4-https-and-session/)

Follow this order. Section 5.4.3 prepares the application; complete HTTPS in 5.4.4 before using login over the network. Then finalize the bucket and data permissions in 5.5, monitoring in 5.6, and overall testing in 5.7.

## Expected results

| Item | Required evidence |
| --- | --- |
| Server | EC2 runs in the correct VPC/subnet with sg-secure-web attached |
| Administration | SSH succeeds from the authorized /32 source |
| Web service | The service runs and recovers after a server reboot |
| HTTPS and sessions | Login/logout work correctly; unauthenticated requests are rejected |
| Data integration | Upload/download work after completing the bucket and permissions in 5.5 |
| Handover | Instance ID, access address, role/profile, and log location; no secrets in public documentation |

![](/FCAJ_Worklog/images/54/ec2tong.png)

Detailed screenshots belong in the individual subsections. Keep the lab to one EC2 instance, track usage time, and clean up in 5.8. Do not assume EC2, EBS, and public IPv4 addresses are all free.

Start with [5.4.1. Launch EC2 and Connect Using SSH](./5.4.1-launch-and-connect/).

## References

- [Use an IAM role for applications on EC2](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html)
