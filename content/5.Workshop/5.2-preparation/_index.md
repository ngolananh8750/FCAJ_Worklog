---
title: "Preparation Steps"
date: 2026-10-27
weight: 2
chapter: false
pre: "<b>5.2. </b>"
---

## Overview

Before deploying the system, the team needs to prepare the AWS account and access permissions and agree on the Region, resource naming conventions, and website content. This preparation helps all four members work in the same environment, coordinate deployment, and avoid creating duplicate resources.

The workshop uses the scenario of **a website with login functionality for uploading, storing, and downloading documents**. The system consists of an EC2 instance running the web application in a public subnet, a private S3 bucket for document storage, and CloudWatch for monitoring. The VPC, security groups, and network ACLs provide networking and traffic control.

## Prerequisites

### AWS Account and Access Permissions

- Have an active AWS account with access to the AWS Management Console.
- Each member uses a separate IAM identity to carry out their assigned tasks; do not share the root account across the team.
- The administrator prepares appropriate permissions for each member, including permission to create and configure resources within their assigned scope, rather than only viewing them.
- Prepare an IAM role for EC2 so that the application can access S3 and send logs to CloudWatch within the required scope. The EC2 IAM role is separate from the team members' Console sign-in permissions.
- Agree on who will manage shared IAM permissions and handle requests for additional permissions when coordinating work across components.

| Team member | Task scope requiring permissions |
| --- | --- |
| **Member 1 — VPC and Firewall** | Create, configure, and clean up the VPC, subnet, internet gateway, route table, security group, network ACL, and S3 gateway VPC endpoint within the workshop scope. |
| **Member 2 — EC2 and Web Server** | Launch and manage EC2; use the network resources provided by the team; attach the approved IAM role; deploy and test the web application. |
| **Member 3 — S3 and Data Security** | Create and configure the bucket, Block Public Access, encryption, access policies, and sample documents within the workshop scope. |
| **Member 4 — CloudWatch and Monitoring** | View EC2 metrics; create, configure, and clean up alarms, log groups, and monitoring settings for the workshop. |

Details on preparing the account and access permissions are provided in [5.2.1. Prepare the AWS Account and Access Permissions](./5.2.1-account-and-permissions/).

### Computer and Lab Tools

- A computer with an internet connection and a browser for using the AWS Management Console.
- An SSH tool such as OpenSSH or Bitvise SSH Client to connect to EC2.
- An editor such as Visual Studio Code to edit source code and Markdown documentation.
- A screenshot tool to capture evidence of configurations and test results.
- A sample PDF or TXT file no larger than 5 MB that contains no personal data.

AWS resources are created and configured through the Console. Web server installation and application deployment on EC2 are performed over SSH, following the instructions in Section 5.4.

### Shared Team Settings

Before creating resources, the entire team needs to agree on the following:

- **AWS Region:** Use the same Region for the system's resources; place the S3 bucket and EC2 instance in the same Region to support connectivity through the S3 gateway VPC endpoint.
- **Resource names:** Agree on a prefix and naming conventions to identify workshop resources.
- **Network settings:** Agree on the VPC and subnet address ranges and the administrator IP address allowed to use SSH.
- **Handover information:** Record shared identifiers such as the VPC ID, Subnet ID, and Security Group ID; add the bucket name, IAM role, and EC2 ID when these resources are created in later sections.
- **Website:** Include a login page and a documents page, using one preconfigured lab account; support document listing, uploads, and downloads after login.
- **Security:** Store passwords as hashes; do not upload passwords, secret keys, or SSH keys to public source code repositories; keep the S3 bucket private.
- **HTTPS:** Agree on a certificate approach for the lab. If a self-signed certificate is used, clearly describe its trust limitations and the browser warning.
- **Costs:** Check the account's actual eligibility for promotional benefits, limit EC2 runtime, document storage, and log retention, and do not assume that the entire workshop is free.

Details of the settings and conventions are provided in [5.2.2. Choose the Region, Resource Names, and Demo Scope](./5.2.2-project-settings/).

## Expected Outcomes

After completing the preparation:

- All four members can sign in to the AWS Management Console using their own identities.
- Access permissions have been prepared according to assigned tasks, and the team knows whom to contact when encountering insufficient-permission errors.
- The team has agreed on the Region, naming conventions, network settings, and how to hand over resource information.
- Computers have an SSH tool, an editor, and a sample document ready for testing.
- The website scope, security requirements, and cost control approach have been agreed upon.


## Contents

1. [5.2.1. Prepare the AWS Account and Access Permissions](./5.2.1-account-and-permissions/)
2. [5.2.2. Choose the Region, Resource Names, and Demo Scope](./5.2.2-project-settings/)
