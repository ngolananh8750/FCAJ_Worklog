---
title: "Introduction"
date: 2026-10-26
weight: 1
chapter: false
pre: "<b>5.1. </b>"
---

## Introduction

The **Designing and Deploying a Secure Web Server on AWS Using VPC, EC2, S3, and CloudWatch** workshop guides participants through building a simple document storage website with network access control, data protection, and server monitoring.

**Practical scenario:** An internship team needs a small website to store sample learning materials. Users sign in, upload documents, view the document list, and download files when needed. Documents are stored in Amazon S3, with direct public access disabled.

To keep the implementation within the scope of the lab, the website consists of only a **login page** and a **documents page**, using one preconfigured shared account. The team does not implement account registration, password recovery, permissions for multiple users, or a separate database. The shared account is intended solely for workshop demonstrations; all files are samples that contain no personal information.

After completing the workshop, participants will be able to:

- Set up a VPC network environment and control traffic using security groups and network ACLs.
- Deploy a web application on EC2 with login, document upload, and document download features.
- Store documents in a private S3 bucket and grant EC2 access through an IAM role.
- Monitor CPU metrics, configure alarms, and collect application logs using CloudWatch.
- Test allowed and denied access scenarios, then clean up resources.

## Overview

The system uses four main services, each serving a specific role in the same scenario:

| Service | Role in the system | Lab activities |
| --- | --- | --- |
| **Amazon VPC** | Provides the network environment for the web server. | Create a VPC, public subnet, internet gateway, and route table; configure security groups, network ACLs, and an S3 gateway VPC endpoint. |
| **Amazon EC2** | Runs the web application and processes user requests. | Implement login, session management, document listing, uploads, and downloads; configure HTTPS. |
| **Amazon S3** | Stores the website's documents. | Create a private bucket, enable Block Public Access, verify encryption, and grant EC2 the minimum required read and write permissions. |
| **Amazon CloudWatch** | Monitors server activity and supports troubleshooting. | View CPU metrics, create alarms, and configure application log collection. |

**Functional scope:**

- Sign in using the preconfigured account and sign out when finished.
- Allow only authenticated users to list, upload, and download documents.
- Restrict sample files to PDF or TXT format, with a maximum size of 5 MB per file.
- Exclude document deletion through the website from the scope of this lab.

**Security focus:**

- Security groups control traffic at the resource level and are stateful; network ACLs control traffic at the subnet level and are stateless. Network ACL configurations must also allow the appropriate return traffic.
- Allow SSH only from the designated administrator IP address. Use HTTPS to access the application; HTTP is used only to redirect to HTTPS, if configured.
- Store passwords as hashes, validate the login session before every document operation, and never log passwords or session tokens.
- Use an IAM role for EC2 to access S3 rather than storing long-term access keys in the source code. Limit permissions to listing, reading, and writing within the required document scope.
- Keep the S3 bucket private, enable Block Public Access, and encrypt data at rest. The application validates file types and sizes and handles filenames safely before storing files.

**Task allocation:** Member 1 is responsible for VPC and firewall configuration; Member 2 for EC2 and the web application; Member 3 for S3 and data security; and Member 4 for CloudWatch and monitoring. All four members use the same architecture and coordinate testing on the same system.

To keep the deployment small, the architecture uses only one EC2 instance in one public subnet, without a NAT gateway, load balancer, or separate database. The team limits document storage and log retention and cleans up resources after the lab. This is a learning environment without server redundancy or high availability.

## Architecture Overview


The system is deployed in one AWS Region. EC2 runs in a public subnet within the VPC; S3 and CloudWatch are services accessed by EC2 and are not located inside the public subnet.

![Sơ đồ kiến trúc](/FCAJ_Worklog/images/5/so_do_kien_truc_aws.drawio.png?featherlight=false&width=90pc)


**Components and connections:**

| Component | Location and connections |
| --- | --- |
| **Users** | Access the web application on EC2 over HTTPS. |
| **Internet gateway and route table** | The public subnet has a default route to the internet gateway; EC2 uses a public IPv4 address to communicate with the internet according to the permitted network rules. |
| **Network ACL and security group** | The network ACL is associated with the subnet; the security group is associated with the EC2 network interface and controls inbound and outbound traffic. |
| **EC2 and IAM role** | EC2 runs the application, authenticates users, and performs S3 operations using the permissions granted to its IAM role. |
| **S3 gateway VPC endpoint** | Associated with the subnet's route table so that traffic from EC2 to S3 in the same Region passes through the endpoint. The endpoint does not replace IAM permissions or bucket policies. |
| **Private S3 bucket** | Stores sample documents; users interact with them through the application on EC2. |
| **CloudWatch** | Receives EC2 monitoring metrics and application logs once the agent and appropriate log delivery permissions are configured. In this architecture, logs are sent over HTTPS to the CloudWatch endpoint through the internet gateway. |

**System workflow:**

1. The user opens the website over HTTPS and enters their login credentials.
2. The application on EC2 validates the credentials; if valid, it creates a login session and displays the documents page.
3. When the user views the document list, the application validates the session and uses its IAM role to list the permitted documents in S3.
4. For an upload, the application validates the session, file type, size, and filename, then writes the document to S3 through the S3 gateway VPC endpoint.
5. For a download, the application validates the session, reads the document from S3, and streams the file to the browser over HTTPS. The website does not require public access to the bucket.
6. CloudWatch is used to monitor CPU activity, check alarms, and view the application logs configured for collection.
7. On logout, the session is invalidated; subsequent requests to document features require the user to sign in again.

**Required outcomes:** Authenticated users can upload and download documents; unauthenticated users are denied access to document features; direct public access to S3 files is denied; and CloudWatch displays monitoring data according to the configuration.

