---
title: "Workshop"
date: 2026-10-26
weight: 5
chapter: false
pre: "<b>5. </b>"
---

## Overview

The **Designing and Deploying a Secure Web Server on AWS Using VPC, EC2, S3, and CloudWatch** workshop helps the team practice deploying a simple website, controlling network access, protecting data, and monitoring server activity.

The main services used are:

| Service | Role in the workshop |
| --- | --- |
| **Amazon VPC** | Provides the network environment for the server; uses security groups and network ACLs to control traffic. |
| **Amazon EC2** | Runs the web application and handles login, document upload, listing, and download requests. |
| **Amazon S3** | Stores documents in a private bucket; supports hands-on practice with permissions, encryption, and data protection. |
| **Amazon CloudWatch** | Monitors EC2 performance metrics, sets up basic alarms, and collects application logs when configured. |

**Practical scenario: Build a simple internal document storage and sharing website on AWS.**

The internship team needs a small website to store and download sample learning materials. The website consists of a login page and a document management page. After signing in, users can upload documents, view the document list, and download documents to their computers.

To simplify implementation, the system uses only **one preconfigured shared account for the lab**, without account registration, password recovery, permissions for multiple users, or a separate database. Passwords are stored as hashes; secrets must not be included in publicly accessible source code. Only sample documents containing no personal data are used.

The team deploys an EC2 instance in a public subnet within the VPC to run the web application and handle login. Documents are stored in a private S3 bucket with Block Public Access and data encryption enabled. EC2 uses an IAM role with the minimum permissions required to list, upload, and read documents through an S3 gateway VPC endpoint. The browser sends requests to the application on EC2; the application validates the login session before each operation and transfers documents between the user and S3. CloudWatch monitors EC2 CPU usage, supports alarm testing, and receives application logs after log collection is configured.

**User workflow:** Sign in → Open the documents page → Upload a document or select one to download → Sign out.

Requirements for completing the scenario:

- Access the website over HTTPS, sign in using the preconfigured account, and sign out successfully. If a self-signed certificate is used in the lab, clearly state its trust limitations.
- Prevent unauthenticated users from listing, uploading, or downloading documents, including when they access feature URLs directly.
- Upload a sample document through the website, verify that it is stored in S3, and successfully download it after signing in.
- Restrict upload file types and sizes to a small scope, such as PDF/TXT files up to 5 MB; handle filenames safely.
- Configure security groups and network ACLs appropriately; allow SSH only from the designated administrator IP address. Use the HTTP port only for redirection to HTTPS, if applicable.
- Keep the S3 bucket private; grant EC2 permissions only for the required bucket or document folder, while denying public access.
- View CPU metrics, test alarms, and inspect application logs in CloudWatch; do not log passwords or login session tokens.
- Use one EC2 instance for the lab, limit document storage and log retention, and clean up resources after completion to keep costs down.

Each member is responsible for one area: **Member 1 — VPC and Firewall; Member 2 — EC2 and Web Server; Member 3 — S3 and Data Security; Member 4 — CloudWatch and Monitoring.** The entire team works on the same scenario and coordinates testing on the same system.

## Contents

- [5.1. Introduction](./5.1-introduction/)
- [5.2. Preparation Steps](./5.2-preparation/)
  - [5.2.1. Prepare the AWS Account and Access Permissions](./5.2-preparation/5.2.1-account-and-permissions/)
  - [5.2.2. Agree on the Region, Resource Names, and Website Content](./5.2-preparation/5.2.2-project-settings/)
- [5.3. Network Setup and Firewall Configuration](./5.3-network-and-firewall/)
  - [5.3.1. Create a VPC and Public Subnet](./5.3-network-and-firewall/5.3.1-vpc-and-subnet/)
  - [5.3.2. Create an Internet Gateway and Route Table](./5.3-network-and-firewall/5.3.2-gateway-and-route-table/)
  - [5.3.3. Configure a Security Group](./5.3-network-and-firewall/5.3.3-security-group/)
  - [5.3.4. Configure a Network ACL](./5.3-network-and-firewall/5.3.4-network-acl/)
  - [5.3.5. Create an S3 Gateway VPC Endpoint](./5.3-network-and-firewall/5.3.5-s3-endpoint/)
  - [5.3.6. Flow Logs and ports 80/81](./5.3-network-and-firewall/5.3.6-flow-logs/)
- [5.4. Deploy a Web Server on EC2](./5.4-ec2-web-server/)
  - [5.4.1. Launch EC2 and Connect via SSH](./5.4-ec2-web-server/5.4.1-launch-and-connect/)
  - [5.4.2. Install the Web Server](./5.4-ec2-web-server/5.4.2-install-web-server/)
  - [5.4.3. Implement Login, Document Upload, and Download Features](./5.4-ec2-web-server/5.4.3-document-web-app/)
  - [5.4.4. Configure HTTPS and Test Login Sessions](./5.4-ec2-web-server/5.4.4-https-and-session/)
- [5.5. Data Storage and Security on S3](./5.5-s3-data-security/)
  - [5.5.1. Create a Private Bucket and Verify Encryption](./5.5-s3-data-security/5.5.1-private-bucket/)
  - [5.5.2. Prepare Sample Documents and Verify Storage on S3](./5.5-s3-data-security/5.5.2-upload-documents/)
  - [5.5.3. Grant IAM Role Permissions and Test EC2 Document Reads and Writes on S3](./5.5-s3-data-security/5.5.3-ec2-access/)
  - [5.5.4. Versioning, Lifecycle, and recovery](./5.5-s3-data-security/5.5.4-versioning-lifecycle-recovery/)
- [5.6. Monitoring and Alarms with CloudWatch](./5.6-cloudwatch/)
  - [5.6.1. Monitor EC2 CPU Metrics](./5.6-cloudwatch/5.6.1-cpu-metrics/)
  - [5.6.2. Create and Test a CPU Alarm](./5.6-cloudwatch/5.6.2-cpu-alarm/)
  - [5.6.3. Collect and Inspect Application Logs](./5.6-cloudwatch/5.6.3-application-logs/)
  - [5.6.4. Dashboard and SNS notifications](./5.6-cloudwatch/5.6.4-dashboard-and-sns/)
- [5.7. System Testing](./5.7-testing/)
  - [5.7.1. Test Login, Uploads, Downloads, and the Firewall](./5.7-testing/5.7.1-website-and-firewall/)
  - [5.7.2. Verify S3 Access Permissions and Monitoring Results](./5.7-testing/5.7.2-s3-and-monitoring/)
- [5.8. Resource Cleanup](./5.8-cleanup/)
