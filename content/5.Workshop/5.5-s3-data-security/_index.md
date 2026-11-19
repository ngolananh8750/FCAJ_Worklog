---
title: "S3 Storage and Data Security"
date: 2026-10-31
weight: 5
chapter: false
pre: "<b>5.5. </b>"
---

## Overview

This section uses **Amazon S3** to store documents for the website deployed on EC2. Users log in to the application to upload and download files; the S3 bucket remains private and does not expose documents through public direct URLs.

**Member 3** creates the bucket, prepares sample documents, and verifies data security. Member 2 coordinates application settings and EC2 permission testing; Member 1 supports the S3 Gateway Endpoint and networking. IAM role creation or modification must be performed by an authorized IAM operator; the S3 operator is not assumed to have these permissions.

This page introduces the scope and deployment order. Detailed procedures are provided in the three subsections below.

## Objectives

- Create a private bucket in the same Region as EC2 and the S3 Gateway Endpoint.
- Keep Block Public Access enabled and use SSE-S3 server-side encryption.
- Organize documents under portal/ and prepare files compatible with the application.
- Grant EC2 only the required list, read, and write permissions through an IAM role.
- Verify website uploads/downloads and confirm that unauthorized access is denied.

## Prerequisites

| Component | Information to verify |
| --- | --- |
| Region | Singapore — ap-southeast-1 |
| EC2 | workshop-web-01 |
| EC2 IAM role | WorkshopEC2AppRole |
| Instance profile | WorkshopEC2AppRole or the actual name verified in 5.4.1 |
| S3 Gateway Endpoint | s3-gateway-endpoint, associated with rtb-web-public |
| Planned bucket | secureweb-docs-683699-280958772766 |
| Document prefix | portal/ |
| Application | document-portal, running through Gunicorn at 127.0.0.1:8000 |
| User connection | HTTPS through Nginx as configured in 5.4.4 |

Use the bucket from 5.2.2 and enable Versioning for the recovery exercise in 5.5.4.

Complete [5.3.5. Create an S3 Gateway VPC Endpoint](../5.3-network-and-firewall/5.3.5-s3-endpoint/) and coordinate with [5.4. Deploy a Web Server on EC2](../5.4-ec2-web-server/). The bucket can be created before the application is complete, but full workflow testing requires HTTPS and the IAM role to be ready.

## Agreed storage configuration

| Property | Workshop configuration |
| --- | --- |
| Bucket type | General purpose |
| Block Public Access | Enable all public-access blocking options |
| Object Ownership | Bucket owner enforced, ACLs disabled |
| Default encryption | SSE-S3, using keys managed by Amazon S3 |
| Sample object storage class | S3 Standard |
| Static website hosting | Not used |
| Practice data | PDF or UTF-8 TXT without personal/confidential information |
| Application limit | At most 5 MiB per file, equal to 5 × 1024 × 1024 bytes |

SSE-S3 fits the current lab scope; this section does not introduce a dedicated KMS key. Encryption at rest does not replace access control. The portal/ prefix is part of an object key, not an operating-system directory with separate permissions.

For documents uploaded directly through the console to appear correctly in the 5.4.3 application, use simple names such as **huong-dan.txt** or **tai-lieu.pdf**, lowercase extensions, and no nested folders under portal/. The application adds a random prefix to uploaded filenames to avoid overwrites.

## Access flow and security layers

1. A user sends an HTTPS request to the application on EC2.
2. The application validates the login session and the file/request.
3. Boto3 uses temporary credentials from the EC2 IAM role to call S3.
4. Same-Region S3 traffic matching the configured route uses the Gateway Endpoint.
5. S3 evaluates applicable policies; the application returns documents only when access is permitted.

Do not store long-term access keys in source code or website configuration. Do not disable Block Public Access to resolve AccessDenied. The endpoint provides a network path, not bucket read/write permissions; SSE-S3 protects stored data and does not authorize website users.

## Application permission scope

| Task | Required permission | Scope |
| --- | --- | --- |
| List documents | s3:ListBucket | Bucket ARN, restricted through s3:prefix to portal/ |
| Download documents | s3:GetObject | Object ARNs under portal/* |
| Upload documents | s3:PutObject | Object ARNs under portal/* |

The application does not need s3:DeleteObject, bucket administration, or account-wide S3 permissions. Detailed policies and scope tests are covered in 5.5.3. Administrative bucket creation/cleanup permissions are separate from application runtime permissions.

A bucket policy can additionally require HTTPS. Before requiring access through a particular VPC Endpoint, evaluate administrative and service access because the condition can block console access outside the endpoint. Do not add such a condition to a working bucket without checking access paths and recovery options.

## Contents

- [5.5.1. Create a Private Bucket and Verify Encryption](./5.5.1-private-bucket/)
- [5.5.2. Prepare Sample Documents and Verify S3 Storage](./5.5.2-upload-documents/)
- [5.5.3. Grant IAM Role Permissions and Verify EC2 Read/Write Access to S3](./5.5.3-ec2-access/)

After creating the bucket, compare its actual name with /etc/document-portal/config.json without publishing the file's password hash. If configuration changes, restart document-portal; in-memory sessions become invalid and users must log in again.

## Expected results

| Item | Required evidence |
| --- | --- |
| Private bucket | Block Public Access enabled and no public grants |
| Encryption | SSE-S3 configuration and encryption details for a sample object |
| Data | PDF/TXT files under portal/ |
| EC2 permissions | Successful listing, upload, and download within the permitted scope |
| Permission boundaries | Out-of-scope operations denied in the 5.5.3 test cases |
| Website | Authenticated uploads/downloads succeed; unauthenticated requests are blocked |



Start with [5.5.1. Create a Private Bucket and Verify Encryption](./5.5.1-private-bucket/).
- [5.5.4. Versioning, Lifecycle, and recovery](./5.5.4-versioning-lifecycle-recovery/)
